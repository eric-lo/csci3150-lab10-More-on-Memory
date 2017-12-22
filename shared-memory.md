# mmap

[mmap](http://man7.org/linux/man-pages/man2/mmap.2.html) is a system call that maps files or devices into memory. In this part we will introduce:

* read file
* write to file 
* cons and pros of `mmap`

## Read file

We use `mmap` to open the following code whose name is _mmap\_read.c._

```C
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <errno.h>

int main(int argc, char *argv[]) {
    int fd, offset;
    char *map;
    struct stat fileInfo;
    // Usage of running this program
    if (argc != 1) {
        fprintf(stderr, "usage: ./mmap_read\n");
        exit(1);
    }
    // Before mapping a file to memory, we need to get a file descriptor for it
    // by using the open() system call
    if ((fd = open("mmap_read.c", O_RDONLY)) == -1) {
        perror("open");
        exit(1);
    }
    if (stat("mmap_read.c", &fileInfo) == -1) {
        perror("stat");
        exit(1);
    }

    // mmap to read
    map = mmap(0, fileInfo.st_size, PROT_READ, MAP_SHARED, fd, 0);
    if (map == MAP_FAILED) {
        perror("mmap");
        exit(1);
    }

    // Print the first line
    printf("The first line is:\n");
    offset = 0;
    while(1) {
        if (map[offset] == '\n') {
            printf("\n");
            break;
        } else {
            printf("%c", map[offset]);
        }
        offset += 1;
    }

    // Free the mmapped memory
    if (munmap(map, fileInfo.st_size) == -1) {
        close(fd);
        perror("Error un-mmapping the file");
        exit(1);
    }
    // Un-mmaping doesn't close the file, so we still need to do that
    close(fd);

    return 0;
}
```

![](/assets/mmap_read.png)

## Write to file

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <errno.h>

int main(int argc, char *argv[]) {
    char * map;
    int fd, offset = 0;
    struct stat fileInfo;
    size_t fileSizeOld, fileSizeNew, textSize;
    // usage of running this program
    if (argc != 2) {
        fprintf(stderr, "usage: mmap_write text\n");
        fprintf(stderr, "       e.g., mmap_write 'hello world'\n");
        exit(1);
    }
    const char *text = argv[1];
    const char *filePath = "./try_mmap_write";
    printf("We will write text '%s' to '%s'.\n", text, filePath);
    // Open a file for writing.
    // Creating the file if it doesn't exist.
    if ((fd = open(filePath, O_RDWR | O_CREAT, (mode_t)0664 )) == -1) {
        perror("open");
        exit(1);
    }
    if (stat(filePath, &fileInfo) == -1) {
        perror("stat");
        exit(1);
    }
    // If the file is not empty, show its content
    if (fileInfo.st_size != 0) {
        map = mmap(0, fileInfo.st_size, PROT_READ, MAP_SHARED, fd, 0);
        if (map == MAP_FAILED) {
            close(fd);
            perror("mmap");
            exit(1);
        }
        printf("The content in '%s' before writing:\n", filePath);
        while (offset < fileInfo.st_size) {
            printf("%c", map[offset]);
            offset++;
        }
        printf("\n");
        if (munmap(map, fileInfo.st_size) == -1) {
            close(fd);
            perror("Error un-mmapping the file");
            exit(1);
        }
    }
    // Stretch the file size to write the array of char
    fileSizeOld = fileInfo.st_size;
    // printf("old: %zu\n", fileSizeOld);
    textSize = strlen(text);
    fileSizeNew = fileInfo.st_size + textSize;
    // printf("new: %zu\n", fileSizeNew);
    if (ftruncate(fd, fileSizeNew) == -1) {
        close(fd);
        perror("Error resizing the file");
        exit(1);
    }
    // mmap to write
    map = mmap(0, fileSizeNew, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (map == MAP_FAILED) {
        close(fd);
        perror("mmap");
        exit(1);
    }
    for (size_t i = 0; i < textSize; i++) {
        /* printf("Writing character %c at %zu\n", text[i], i); */
        map[i+fileSizeOld] = text[i];
    }
    // Write it now to disk
    if (msync(map, fileSizeNew, MS_SYNC) == -1) {
        perror("Could not sync the file to disk");
    }
    // Free the mmapped memory
    if (munmap(map, fileSizeNew) == -1) {
        close(fd);
        perror("Error un-mmapping the file");
        exit(1);
    }
    // Un-mmaping doesn't close the file, so we still need to do that
    close(fd);

    return 0;
}
```

![](/assets/mmap_write.png)

## Pros and Cons of Using `mmap`

Maybe you are wondering why do we use  `mmap` for file access instead of the standard `read` and `write` system calls. We will list some advantages and disadvantages of `mmap` below.

### Advantages of `mmap`

* Reading from and writing to a memory-mapped file avoids the extraneous copy that occurs when using the read or write system calls, where the data must be copied to and from a user-space buffer.
* Aside from any potential page faults, reading from and writing to a memory-mapped file does not incur any system call or context switch overhead. It is as simple as accessing memory.

* When multiple processes map the same object into memory, the data is shared among all the processes. This can save a lot of memory, which is common in the kind of server systems. Read-only and shared writable mappings are shared in their entirety; private writable mappings have their not-yet-COW \([copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write)\) pages shared.

### Disadvantages of `mmap`

* Memory mappings are always an integer number of pages in size. Thus, the difference between the size of the backing file and an integer number of pages is "wasted" as slack space. For small files, a significant percentage of the mapping may be wasted. For example, with 4 KB pages, a 7 byte mapping wastes 4,089 bytes.
* The memory mappings must fit into the process' address space. With a 32-bit address space, a very large number of various-sized mappings can result in fragmentation of the address space, making it hard to find large free contiguous regions. This problem, of course, is much less apparent with a 64-bit address space.

* There is overhead in creating and maintaining the memory mappings and associated data structures inside the kernel. This overhead is generally obviated by the elimination of the double copy mentioned in the previous section, particularly for larger and frequently accessed files.

**References:**

* [Manual for`mmap`](http://man7.org/linux/man-pages/man2/mmap.2.html)
* [Manual for`open`](https://linux.die.net/man/3/open)
* [Memory mapped file](https://beej.us/guide/bgipc/output/html/multipage/mmap.html)
* [mmap and read/write string to file](https://gist.github.com/sanmarcos/991042)
* [When should I use mmap for file access](http://stackoverflow.com/questions/258091/when-should-i-use-mmap-for-file-access)
* [Linux System Programming](https://www.safaribooksonline.com/library/view/linux-system-programming/0596009585/ch04s03.html)



