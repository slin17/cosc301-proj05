#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>

#include "bootsect.h"
#include "bpb.h"
#include "direntry.h"
#include "fat.h"
#include "dos.h"

typedef struct node {
	uint16_t cluster;
	struct node *next;
} File; 

int in_list(uint16_t cluster, File *head) {
    File *tmp = head;
    while (tmp != NULL) {
        if (cluster == tmp->cluster) {
		return 1;
        }
        tmp = tmp->next;
    }
    return 0;
}

int list_add(uint16_t cluster, File **head) {
    if (in_list(cluster, *head)) {//don't want to add if it's already in the list
        return 0;
    }
    File *newfile = (File*) malloc(sizeof(File));
    newfile->cluster = cluster;
    newfile->next = NULL;
    File *tmp = *head;
    if (tmp == NULL) {
        *head = newfile;
        return 0;
    }
    while (tmp->next != NULL) {
        tmp = tmp->next;
    }
    tmp->next = newfile;
    newfile->next = NULL;
}

void list_clear(File *list_head) {
    while (list_head != NULL) {
        File *tmp = list_head;
        list_head = list_head->next;
        free(tmp);
    }
}

uint16_t get_dirent(struct direntry *dirent, char *buffer)
{//get directory entry (similar implementation to the one in dos_cat.c
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
    if (name[0] == SLOT_EMPTY)
    {
	return followclust;
    }

    /* skip over deleted entries */
    if (((uint8_t)name[0]) == SLOT_DELETED)
    {
	return followclust;
    }

    if (((uint8_t)name[0]) == 0x2E)
    {
	// dot entry ("." or "..")
	// skip it
        return followclust;
    }

    /* names are space padded - remove the spaces */
    for (i = 8; i > 0; i--) 
    {
	if (name[i] == ' ') 
	    name[i] = '\0';
	else 
	    break;
    }

    /* remove the spaces from extensions */
    for (i = 3; i > 0; i--) 
    {
	if (extension[i] == ' ') 
	    extension[i] = '\0';
	else 
	    break;
    }

    if ((dirent->deAttributes & ATTR_WIN95LFN) == ATTR_WIN95LFN)
    {
	// ignore any long file name extension entries
	//
	// printf("Win95 long-filename entry seq 0x%0x\n", dirent->deName[0]);
    }
    else if ((dirent->deAttributes & ATTR_DIRECTORY) != 0) 
    {
        // don't deal with hidden directories; MacOS makes these
        // for trash directories and such; just ignore them.
	if ((dirent->deAttributes & ATTR_HIDDEN) != ATTR_HIDDEN)
        {
            strcpy(buffer, name);
            file_cluster = getushort(dirent->deStartCluster);
            followclust = file_cluster;
        }
    }
    else 
    {
        /*
         * a "regular" file entry
         * print attributes, size, starting cluster, etc.
         */
        strcpy(buffer, name);
        if (strlen(extension))  
        {
            strcat(buffer, ".");
            strcat(buffer, extension);
        }
    }

    return followclust;
}

int chk&fix(struct direntry* dirent, unint8_t *image_buf, struct bpb33* bpb, char* filename, File **clus_list)
{
	int clus_size = count_size_in_clusters(dirent, image_buf, bpb, cluster_list);
} 

void traverse_root(unint8_t *image_buf, struct bpb33* bpb) {
	uint16_t cluster = 0; //indicates root directory
	File* list = NULL; 
	int inconst = 0; //inconsistency checker
	struct direntry *dirent = (struct direntry*)cluster_to_addr(cluster, image_buf, bpb);
	char buffer[MAXFILENAME]; 
	
	int i = 0;
	for ( ; i < bpb->bpbRootDirEnts; i++) { //go through every entry in root dir
		uint16_t followclust = get_dirent(dirent, buffer);
		if (dirent->deAttributes == ATTR_NORMAL) {//if it's a normal file
			if (chk&fix(dirent, image_buf, bpb, buffer, &list)){
				inconst = 1;
			}
		} 
	}
}

void usage(char *progname) {
    fprintf(stderr, "usage: %s <imagename>\n", progname);
    exit(1);
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

    // your code should start here...
    traverse_root(image_buf, bpb);





    unmmap_file(image_buf, &fd);
    return 0;
}
