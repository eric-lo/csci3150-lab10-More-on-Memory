#Pinning the Pages

Many management facilities ensure that process have effective and equitable access to memory resources. The operating system maps and controls the relationship between physical memory and the virtual memory space of a process. These activities are, for the most part, transparent to the user and controlled by the operating system. However, for more many real-time applications you may need to make more efficient use of system resources by controlling virtual memory usage.

Pinning the pages in main memory is one way to ensure that a process stays in main memory and is exempt from paging. So, a real-time process must have the ability to pin the pages of its address space in memory, in order to guarantee immediate execution when triggered by an event, such as an interrupt. Real-time processes require deterministic timing, and paging can be a source of unexpected delays in program execution. Critical timing requirements can be thrown away if some portion of a real-time application has been previously paged out by the operating system. So, without pinning the pages in memory, valuable time may be spent retrieving pages.

However, for some real-time process, we may only pin a key portion of pages in memory to reach the timing requirement. In this way, the precious memory space can be conserved for other pieces of the system to execute efficiently.

## Pinning all the pages with `mlockall()`

The mlockall\(\) function pins all the pages mapped to the address space of calling process in main memory. Any shared library utilized by the application is also locked into the memory in its entirety. This prevents any part of the process from being paged out of main memory. All pages are guaranteed to be resident in main memory upon the successful return from `mlockall()`.

`int mlockall(int flags);`

In Linux, there are two types of user-space memory.

* Mapped memory - Has a file behind it, mapped with `mmap()` which we mentioned before. This type of memory can be paged out of main memory even if there is no _swap_ in the system because the contents can always be restored from the file-system.
* Anonymous memory - Not mapped to a file. Examples are the program stack, uninitialized memory\(.bss\) and heap\(malloc allocate memory\). Anonymous memory can only be paged out to swap disk.

And `mlockall()` protects both types of memory from swapping. And mlockall\(\) requires that one of two flags specified:  MCL\_CURRENT and/or MCL\_FUTURE.

* The MCL\_CURRENT flag specifies that all pages mapped to the process at the moment `mlockall()`is called are pinned in main memory. If the process address space was to later grow, through dynamic memory allocation for example, those new pages would not be pinned in memory. 
* The MCL\_FUTURE flag specifies that any new pages provisioned and mapped to the process after the call to `mlockall()` are pinned in main memory. Therefore, if the process's address space was to grow through dynamic memory allocation, for example, all new pages would be automatically pinned in the memory.

Passing both flags to mlockall\(\) guarantees that all current pages and all future pages, provisioned in response to subsequent growth, are pinned in the memory. The figure below shows the memory space of a process while calling `mlockall()` function.

![](/assets/MemorySpace.png)

A: When the process is first started, all of its pages are not pinned in memory.

B: After a call to `mlockall(MCL_CURRENT | MCL_FUTURE)`,  all current pages allocated to the process are pinned into main memory.

C: When the process calls `malloc()`,  the heap grows to high memory address. As we has also specified MCL\_FUTURE flag. The pages of the newly allocated memory will be pinned to the memory immediately.

The `munlockall()` function unpins all pages mapped by a call to the `mlockall()` function, even if the MCL\_FUTURE flag was specified on the call to `mlockall()` function. The call to the` munlockall()`  function cancels the MCL\_FUTURE flag. 

### Example for `mlockall() `


_mlockall\_example.c:_ 

```C
#include<stdio.h>
#include<sys/mman.h>
#define SIZE 500

struct record {
	char name[128];
	int count;
	unsigned char dummy_array[1024000];
	struct record * next;
};

int main(void){
	struct record * a[SIZE];
	int i;
	printf("Starting mlockall_example.\n");
	/* 
	 * Reallocate memory required by the application before
	 * real-time processing begins
	 */
	/* Insert code here */
	
	/* Pin all pages mapped to the process in memory */
	printf("Pin all the pages in memory.\n");
	if(mlockall(MCL_CURRENT | MCL_FUTURE) < 0) {
		perror("mlockall");
		return 1;
	}
	/*
	 * Begin real-time processing 
	 */

	/*
	 * Release all memory belonging to this process 
	 */
	printf("Finished, clean up.\n");
	munlockall();
	return 0;
}
```

 
To build and run the example:

`$ gcc mlockall_example.c -o mlockall`

In order to run mlockall program, we need to use `sudo`.  
![](/assets/mlockall_simple.PNG)

## Pinning pages of a specific region

The `mlock()` function pins the pages of a pre-allocated specified region in memory. 

` int mlock(const void *addr, size_t len);`

The `addr` and `len` arguments of the `mlock()` function determine the boundaries of the pre-allocated region. On a  successful call to `mlock()`, the pages of the specified region are pinned in the memory. The `mlock() `function pins all pages containing any part of the request range specified by `addr` and` len`. The figure below shows the memory space of the process both before and after a call to `mlock()` function.

![](/assets/mlock_double.PNG)



A: Prior to the call to the `mlock()` function, pages of the buffer space in the read/write segment are not pinned and are subject to paging. 

B: After the call to the `mlock()` function the buffer space cannot be paged out of memory. 

### Example for `mlock()`

This example shows how to lock a specific area of memory using `mlock()`.

_mlock\_example.c:_

```C
#include<stdio.h>
#include<sys/mman.h>

#define DATA_SIZE 2048
double buffer[DATA_SIZE];

int main(){
	if( mlock(buffer, DATA_SIZE * sizeof(double)) == -1)
		perror("mlock");
	printf("Pin the pages successfully\n");
	/*
	 * Start real-time work here
	 */
	if( munlock(buffer, DATA_SIZE * sizeof(double)) == -1 )
		perror("munlock");
	printf("Unpin the pages successfully\n");
	
	return 0;
}
```

To build and run the example:

`$ gcc mlock_example.c -o mlock`

![](/assets/mlock_simple.PNG)

**References**

[Memory Locking](https://docs.zoho.com/writer/published/b0tmj177a813c040b4c67bb380f907986ccf4?mode=embed)

