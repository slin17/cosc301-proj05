#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <assert.h>
#include <ctype.h>
#include "bootsect.h"
#include "bpb.h"
#include "direntry.h"
#include "fat.h"
#include "dos.h"

#define CLUSTER_SIZE_bpb (bpb->bpbSecPerClust * bpb->bpbBytesPerSec)

//prototypes for functions from dos_ls/dos_cat/dot_cp and linked list;
//function implementations are at the end
typedef struct node{
	uint16_t cluster;
	struct node *next;
} Node; 
void cluster_add(uint16_t cluster, Node **clusterlist);

void cluster_clear( Node *file_list);

int check_cluster(uint16_t cluster, Node *headoflist);

int fix_in(struct direntry* dirent,char* filename,uint8_t *image_buf,
	   struct bpb33* bpb, Node **clusterlist );

uint16_t get_dirent(struct direntry *dirent, char *buffer);

void write_dirent(struct direntry *dirent, char *filename, 
		  uint16_t start_cluster, uint32_t size);

int follow_dir(uint16_t cluster, int indent,
               uint8_t *image_buf, struct bpb33* bpb, Node** cluster_list);

void usage(char *progname) {
    fprintf(stderr, "usage: %s <imagename>\n", progname);
    exit(1);
}

void create_dirent(struct direntry *dirent, char *filename, 
		   uint16_t start_cluster, uint32_t size,
		   uint8_t *image_buf, struct bpb33* bpb);

void print_message(int is_inconsist);




void free_cluster(struct direntry *dirent, uint8_t *image_buf, struct bpb33 *bpb, int actual_size) {
    
    uint16_t cluster = getushort(dirent->deStartCluster);
    uint16_t size = 0;
    uint16_t prev_cluster = cluster;

    while (size < actual_size) {
        size += CLUSTER_SIZE_bpb;
        prev_cluster = cluster;
        cluster = get_fat_entry(cluster, image_buf, bpb);
    }
    if (size != 0) {
        set_fat_entry(prev_cluster, FAT12_MASK & CLUST_EOFS, image_buf, bpb);
    }
    // mark the rest of clusters pointed by the FAT chain as free
    while (!is_end_of_file(cluster)) {
        uint16_t oldcluster = cluster;
        cluster = get_fat_entry(cluster, image_buf, bpb);
        set_fat_entry(oldcluster, FAT12_MASK & CLUST_FREE, image_buf, bpb);
    }
}


int fix_in(struct direntry* dirent, char* filename, uint8_t *image_buf,
           struct bpb33* bpb, Node **cluster_list) 
{//fix inconsitencies with cluster size in FAT compared to directory entry 
    uint16_t cluster = getushort(dirent->deStartCluster);
    int size_in_clusters = 0;
    cluster_add(cluster, cluster_list);
    uint16_t prev_cluster = cluster;
    if (is_end_of_file(cluster)) {
        size_in_clusters = 512;
    }
    while (!is_end_of_file(cluster) && cluster ){ 
        if (cluster == (FAT12_MASK & CLUST_BAD)) {
            printf("Bad cluster number %d \n", cluster);
            set_fat_entry(prev_cluster, FAT12_MASK & CLUST_EOFS, image_buf, bpb);
            break;
        }

        if (cluster == (FAT12_MASK & CLUST_FREE)) {
            set_fat_entry(prev_cluster, FAT12_MASK & CLUST_EOFS, image_buf, bpb);
            break;   
        }

        size_in_clusters += CLUSTER_SIZE_bpb;
        prev_cluster = cluster;
        cluster = get_fat_entry(cluster, image_buf, bpb);
        
        if (prev_cluster == cluster) {
            printf("Cluster refers to itself! Now set it as end of file. \n");
            set_fat_entry(prev_cluster, FAT12_MASK & CLUST_EOFS, image_buf, bpb);
            break;   
        }

        cluster_add(cluster, cluster_list);
    }

    int is_inconsist = 0;
    uint32_t size_in_dirent = getulong(dirent->deFileSize);
    //printf("size in dirent: %d\n", size_in_dirent);
    //printf("size in cluster: %d\n", size_in_clusters);
    //fix inconsistency in FAT
    if (size_in_clusters != 0 && size_in_dirent < size_in_clusters - 512 ) { 
        is_inconsist = 1;
	printf("Find inconsistency in file: %s. Size in FAT is bigger by more than 512 bytes.\n", filename);
	printf("Size according to directory entry: %d\n",size_in_dirent);
	printf("Size according to FAT chain: %d\n",size_in_clusters);
        free_cluster(dirent, image_buf, bpb, size_in_dirent);
    }
    //fix the size entry
    else if (size_in_dirent > size_in_clusters) { 
	is_inconsist = 1;	
	printf("Find inconsistency in file: %s. Size is bigger in directory entry.\n", filename);
	printf("Size according to directory entry: %d\n",size_in_dirent);
	printf("Size according to FAT chain: %d\n",size_in_clusters);
        putulong(dirent->deFileSize, size_in_clusters);
    }

    if (size_in_dirent == 0) {
        if (dirent->deAttributes == ATTR_NORMAL && dirent->deName[0] != SLOT_EMPTY && dirent->deName[0] != SLOT_DELETED) {
            printf("File with size 0 found. Deleting the entry. \n");
            dirent->deName[0] = SLOT_DELETED;
        }
    }

    return is_inconsist;
}

uint32_t calc_size(uint16_t cluster, uint8_t *image_buf, struct bpb33 *bpb, Node **cluster_list)
{//calculate the size of cluster
    uint16_t cluster_size = CLUSTER_SIZE_bpb; 

    uint32_t size = 0;
    cluster_add(cluster, cluster_list);
    
    while (!is_end_of_file(cluster)) {   
        
        if (cluster == (FAT12_MASK & CLUST_BAD)) {
            printf("Bad cluster: cluster number %d \n", cluster);
        }

        size += cluster_size;

        cluster = get_fat_entry(cluster, image_buf, bpb);
        cluster_add(cluster, cluster_list);
    }

    return size;
}

int find_orphan_creatDir (uint8_t *image_buf, struct bpb33* bpb, Node *list) 
{
    int orphan_located = 0;
    int inconst = 0; //to keep track of inconsitency for print_message
    uint16_t cluster;
    
    uint16_t check_clust = (FAT12_MASK & CLUST_FIRST); //get the cluster 2 in the data region
    struct direntry *dirent; 
    
    for ( ; check_clust < 2849; check_clust++) { //for loop to prints all the orphan clusters
        if (!check_cluster(check_clust, list) && (get_fat_entry(check_clust, image_buf, bpb) != CLUST_FREE))  {
	//cluster not in the referenced list but the fat entry does not indicate free
            printf("Orphan cluster located; cluster number %d \n", check_clust);
            inconst = 1;
            orphan_located = 1;
        }
    } 
    
    int orphan_count = 0;
    
    uint16_t clust = (FAT12_MASK & CLUST_FIRST);
    
    while (orphan_located) {
        orphan_located = 0;
        for ( ; clust  < 2849; clust++) {
            if (!check_cluster(clust, list) && (get_fat_entry(clust, image_buf, bpb) != CLUST_FREE))  {
                inconst = 1;
                orphan_located = 1;
                break;
            }
        } 
        if (orphan_located) {//if orphan is located, create a directory entry in the root directory
            orphan_count++;
            cluster = 0; 
            dirent = (struct direntry*)cluster_to_addr(cluster, image_buf, bpb);
            char filename[13];
            memset(filename, '\0', 13);
            strcat(filename, "found");
            char str[3];
            memset(str, '\0', 3);
            int orphan_count_copy = orphan_count;
            sprintf(str, "%d", orphan_count_copy);
            strcat(filename, str);
            strcat(filename, ".dat");
            int size_clus = calc_size(clust, image_buf, bpb, &list);
            cluster_add(clust, &list);
            create_dirent(dirent, filename, clust, size_clus, image_buf, bpb);
            inconst = 1;
        }
         
    }
    return inconst;
}

void traverse_root(uint8_t *image_buf, struct bpb33* bpb)
{
    Node* list = NULL;
    int is_inconsist = 0;
    uint16_t cluster = 0; //indicates root directory
    struct direntry *dirent = (struct direntry*)cluster_to_addr(cluster, image_buf, bpb);
    char buffer[MAXFILENAME];
    int i = 0;
    for ( ; i < bpb->bpbRootDirEnts; i++)
    {
        uint16_t followclust = get_dirent(dirent, buffer);
        if (dirent->deAttributes == ATTR_NORMAL) {//normal file
            if (fix_in(dirent, buffer, image_buf, bpb, &list)) {
                is_inconsist = 1;
		//printf("INCON1:%d\n", is_inconsist); for testing
            }
        }
        cluster_add(followclust, &list);//adds to cluster list to keep track of visited ones
        if (is_valid_cluster(followclust, bpb)) {
            cluster_add(followclust, &list);
            if (follow_dir(followclust, 1, image_buf, bpb, &list)) {
                is_inconsist = 1;
		//printf("INCON:%d\n", is_inconsist); for testing
            }
        }
        dirent++;
    }

    if(find_orphan_creatDir(image_buf, bpb, list)){
    	is_inconsist=1;
    }
    print_message(is_inconsist);
    cluster_clear(list);
}



int main(int argc, char** argv) {
    uint8_t *image_buf;
    int fd;
    struct bpb33* bpb;
    if (argc < 2) {
	usage(argv[0]);
    }

    image_buf = mmap_file(argv[1], &fd);
    bpb = check_bootsector(image_buf);
    traverse_root(image_buf, bpb);
    free(bpb);
    unmmap_file(image_buf, &fd);
    return 0;
}

//Linked list functions
void cluster_add(uint16_t cluster, Node **file_list) {
    if (!check_cluster(cluster, *file_list)) {    
       Node *newnode= malloc(sizeof(Node));
       newnode->cluster = cluster;
       newnode->next = NULL;
       Node *curr = *file_list;
       if (curr == NULL) {
           *file_list = newnode;
           return;
       }
       while (curr->next != NULL) {
           curr = curr->next;
       }
       curr->next = newnode;
       newnode->next = NULL;
    }
}

void cluster_clear( Node *file_list) {
    while (file_list != NULL) {
         Node *tmp = file_list;
        file_list = file_list->next;
        free(tmp);
    }
}

int check_cluster(uint16_t cluster, Node *headoflist)
{
    Node *current = headoflist;
    while (current != NULL) {
        if (cluster == current->cluster) {
            return 1;
        }
        current = current->next;
    }
    return 0;
}


// functions for inconsistency problems
uint16_t get_dirent(struct direntry *dirent, char *buffer)
{
    uint16_t followclust = 0;
    memset(buffer, 0, MAXFILENAME);

    int i;
    char name[9];
    char extension[4];
    uint16_t file_cluster;
    name[8] = ' ';
    extension[3] = ' ';
    memcpy(name, &(dirent->deName[0]), 8);
    memcpy(extension, dirent->deExtension, 3);
    if (name[0] == SLOT_EMPTY){
	return followclust;
    }

    /* skip over deleted entries */
    if (((uint8_t)name[0]) == SLOT_DELETED) {
	return followclust;
    }
    if (((uint8_t)name[0]) == 0x2E){
	// dot entry ("." or "..")
	// skip it
        return followclust;
    }

    /* names are space padded - remove the spaces */
    for (i = 8; i > 0; i--){
	if (name[i] == ' ') 
	    name[i] = '\0';
	else 
	    break;
    }

    /* remove the spaces from extensions */
    for (i = 3; i > 0; i--){
	if (extension[i] == ' ') 
	    extension[i] = '\0';
	else 
	    break;
    }

    if ((dirent->deAttributes & ATTR_WIN95LFN) == ATTR_WIN95LFN) {
	// ignore any long file name extension entries
	//
	// printf("Win95 long-filename entry seq 0x%0x\n", dirent->deName[0]);
    }
    else if ((dirent->deAttributes & ATTR_DIRECTORY) != 0){
        // don't deal with hidden directories; MacOS makes these
        // for trash directories and such; just ignore them.
	if ((dirent->deAttributes & ATTR_HIDDEN) != ATTR_HIDDEN){
            strcpy(buffer, name);
            file_cluster = getushort(dirent->deStartCluster);
            followclust = file_cluster;
        }
    }
    else{
        /*
         * a "regular" file entry
         * print attributes, size, starting cluster, etc.
         */
        strcpy(buffer, name);
        if (strlen(extension)){
            strcat(buffer, ".");
            strcat(buffer, extension);
        }
    }
    return followclust;
}

/* write the values into a directory entry */
void write_dirent(struct direntry *dirent, char *filename, 
		  uint16_t start_cluster, uint32_t size)
{
    char *p, *p2;
    char *uppername;
    int len, i;
    /* clean out anything old that used to be here */
    memset(dirent, 0, sizeof(struct direntry));
    /* extract just the filename part */
    uppername = strdup(filename);
    p2 = uppername;
    for (i = 0; i < strlen(filename); i++) {
	if (p2[i] == '/' || p2[i] == '\\') 
	{uppername = p2+i+1;}
    }
    /* convert filename to upper case */
    for (i = 0; i < strlen(uppername); i++) {
	uppername[i] = toupper(uppername[i]);
    }
    /* set the file name and extension */
    memset(dirent->deName, ' ', 8);
    p = strchr(uppername, '.');
    memcpy(dirent->deExtension, "___", 3);
    if (p == NULL) 
    {fprintf(stderr, "No filename extension given - defaulting to .___\n");}
    else {
	*p = '\0';
	p++;
	len = strlen(p);
	if (len > 3) len = 3;
	memcpy(dirent->deExtension, p, len);
    }

    if (strlen(uppername)>8) 
    {uppername[8]='\0';}
    memcpy(dirent->deName, uppername, strlen(uppername));
    free(p2);

    /* set the attributes and file size */
    dirent->deAttributes = ATTR_NORMAL;
    putushort(dirent->deStartCluster, start_cluster);
    putulong(dirent->deFileSize, size);
    /* could also set time and date here if we really cared... */
}

/* create_dirent finds a free slot in the directory, and write the directory entry */
void create_dirent(struct direntry *dirent, char *filename, 
		   uint16_t start_cluster, uint32_t size,
		   uint8_t *image_buf, struct bpb33* bpb)
{
    while (1){
	if (dirent->deName[0] == SLOT_EMPTY){
	    /* we found an empty slot at the end of the directory */
	    write_dirent(dirent, filename, start_cluster, size);
	    dirent++;

	    /* make sure the next dirent is set to be empty, just in
	       case it wasn't before */
	    memset((uint8_t*)dirent, 0, sizeof(struct direntry));
	    dirent->deName[0] = SLOT_EMPTY;
	    return;
	}
	if (dirent->deName[0] == SLOT_DELETED) {
	    /* we found a deleted entry - we can just overwrite it */
	    write_dirent(dirent, filename, start_cluster, size);
	    return;
	}
	dirent++;
    }
}

int follow_dir(uint16_t cluster, int indent,
        uint8_t *image_buf, struct bpb33* bpb, Node** cluster_list)
{   
    int is_inconsist = 0;
    while (is_valid_cluster(cluster, bpb)){   
        cluster_add(cluster, cluster_list);
        struct direntry *dirent = (struct direntry*)cluster_to_addr(cluster, image_buf, bpb);
        int numDirEntries = (bpb->bpbBytesPerSec * bpb->bpbSecPerClust) / sizeof(struct direntry);
        char buffer[MAXFILENAME];

        int i = 0;
        for ( ; i < numDirEntries; i++) {
            cluster_add(cluster, cluster_list);
            uint16_t followclust = get_dirent(dirent, buffer);
            if (fix_in(dirent, buffer, image_buf, bpb, cluster_list)) {//check if there's any inconsistency fixed 
                is_inconsist = 1; //for print message 
            }
            if (followclust) {
                if (follow_dir(followclust, indent+1, image_buf, bpb, cluster_list)) {
                    is_inconsist = 1;
                }
            }
            dirent++;
        }
        cluster = get_fat_entry(cluster, image_buf, bpb);
    }
    return is_inconsist;
}

void print_message(int is_inconsist)
{
    if (is_inconsist) 
    {printf("There was inconsistency in the image. But the problem is fixed by scandisk! \n");}
    else 
    {printf("There is no inconsistency in the image. It is a good image! \n");}
}
