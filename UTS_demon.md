```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>
#define STACK_SIZE 1024*1024

#define errExit(msg)	do { perror(msg); exit(EXIT_FAILURE);} while(0)

static char stack_child[STACK_SIZE];

char* const argv[]={"/bin/bash", NULL};

int childFunc(void* arg)
{
	printf("in child process.\n");

	sethostname("thebeeman-host",8);	

	execv(argv[0], argv);	

	return 1;
}

void main()
{
	//char* stack_child = (char *)malloc(stack_size);
	
	int pid_child;	
	
	pid_child = clone(childFunc, stack_child + STACK_SIZE, SIGCHLD | CLONE_NEWUTS, NULL);	
	
	if (pid_child == -1)
		errExit("clone");
	printf("clone() returned %ld\n", (long) pid_child);
	
	waitpid(pid_child, NULL, 0);
	
	printf("child has terminated.\n");
}
```