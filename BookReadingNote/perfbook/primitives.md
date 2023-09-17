# Primitives

## Compiler barrier for GCC

```C
#define ACCESS_ONCE(x) (*(volatile typeof(x) *)&(x))
#define READ_ONCE(x) \
({ typeof(x) ___x = ACCESS_ONCE(x); ___x; })
#define WRITE_ONCE(x, val) \
do { ACCESS_ONCE(x) = (val); } while (0)
#define barrier() __asm__ __volatile__("": : :"memory")
```

The __sync_synchronize() primitive issues a “memory barrier”, which constrains both the compiler’s and the CPU’s ability to reorder operations



## POSIX Reader-Writer Locking

```C
1 pthread_rwlock_t rwl = PTHREAD_RWLOCK_INITIALIZER;
2 unsigned long holdtime = 0;
3 unsigned long thinktime = 0;
4 long long *readcounts;
5 int nreadersrunning = 0;
6
7 #define GOFLAG_INIT 0
8 #define GOFLAG_RUN 1
9 #define GOFLAG_STOP 2
10 char goflag = GOFLAG_INIT;
11
12 void *reader(void *arg)
13 {
14 int en;
15 int i;
16 long long loopcnt = 0;
17 long me = (long)arg;
18
19 __sync_fetch_and_add(&nreadersrunning, 1);
20 while (READ_ONCE(goflag) == GOFLAG_INIT) {
21 continue;
22 }
23 while (READ_ONCE(goflag) == GOFLAG_RUN) {
24 if ((en = pthread_rwlock_rdlock(&rwl)) != 0) {
25 fprintf(stderr,
26 "pthread_rwlock_rdlock: %s\n", strerror(en));
27 exit(EXIT_FAILURE);
28 }
29 for (i = 1; i < holdtime; i++) {
30 wait_microseconds(1);
31 }
32 if ((en = pthread_rwlock_unlock(&rwl)) != 0) {
33 fprintf(stderr,
34 "pthread_rwlock_unlock: %s\n", strerror(en));
35 exit(EXIT_FAILURE);
36 }
37 for (i = 1; i < thinktime; i++) {
38 wait_microseconds(1);
39 }
40 loopcnt++;
41 }
42 readcounts[me] = loopcnt;
43 return NULL;
44 }
```

### READ_ONCE 

READ_ONCE here serves as volatile, it's just a hint for compiler. Compared to volatile, it's better for code reader to notice.



### Memory barrier

**Rule of thumb**: You do not need memory barriers unless you are using more than one variable to communicate between multiple threads.





## Compiler pitfalls

### Load tearing

On (say) an 8-bit system with 16-bit pointers, the compiler might have no choice but to use a pair of 8-bit instructions to access a given pointer. Because the C standard must support all manner of systems, the standard cannot rule out load tearing in the general case.

### Store tearing 

Compiler might split a 64-bit store into two 32-bit stores in order to reduce the overhead of explicitly forming the 64-bit constant in a register, even on a 64-bit CPU

### Load fusing

Occurs when the compiler uses the result of a prior load from a given variable instead of repeating the load

```C
1 while (!need_to_stop)
2	 do_something_quickly();
```

```C
1 if (!need_to_stop)
2 for (;;) {
3 	do_something_quickly();
4 	do_something_quickly();
5 	do_something_quickly();
6 	do_something_quickly();
7 	do_something_quickly();
8 	do_something_quickly();
9 	do_something_quickly();
10 	do_something_quickly();
11 	do_something_quickly();
12 	do_something_quickly();
13 	do_something_quickly();
14 	do_something_quickly();
15 	do_something_quickly();
16 	do_something_quickly();
17 	do_something_quickly();
18 	do_something_quickly();
19 }
```

The compiler might reasonably unroll this loop sixteen times in order to reduce the per-invocation of the backwards branch at the end of the loop. Worse yet, because the compiler knows that do_something_quickly() does not store to need_to_stop, the compiler could quite reasonably decide to check this variable only once



```C
1 int *gp;
2
3 void t0(void)
4 {
5 	WRITE_ONCE(gp, &myvar);
6 }
7
8 void t1(void)
9 {
10 	p1 = gp;
11 	do_something(p1);
12 	p2 = READ_ONCE(gp);
13 	if (p2) {
14 		do_something_else();
15 		p3 = *gp;
16 	}
17 }
```

* C compiler can fuse non-adjacent loads

The compiler is within its rights to fuse the read on lines 10 and 15, which means that if line 10 loads NULL and line 12 loads &myvar, line 15 could load NULL, resulting in a fault

### Store fusing

Occurs when the compiler notices a pair of successive stores to a given variable with no intervening loads from that variable. In this case, the compiler is within its rights to omit the first store.

```C
1 void shut_it_down(void)
2 {
3 	status = SHUTTING_DOWN; /* BUGGY!!! */
4 	start_shutdown();
5 	while (!other_task_ready) /* BUGGY!!! */
6 		continue;
7 	finish_shutdown();
8 	status = SHUT_DOWN; /* BUGGY!!! */
9 	do_something_else();
10 }
11
12 void work_until_shut_down(void)
13 {
14 	while (status != SHUTTING_DOWN) /* BUGGY!!! */
15 		do_more_work();
16 	other_task_ready = 1; /* BUGGY!!! */
17 }
```

Assuming that neither start_shutdown() nor finish_shutdown() access status, the compiler could reasonably **remove the store to status** on line 3.

### Code reordering

Code reordering is a common compilation technique used to combine common subexpressions, reduce register pressure, and improve utilization of the many functional units available on modern superscalar microprocessors.

Suppose that the *do_more_work()* function on line 15 does not access other_task_ready. Then the compiler would be within its rights to move the assignment to other_task_ready on line 16 to **precede** line 14.



### Invented load

```C
1 ptr = global_ptr;
2 if (ptr != NULL && ptr < high_address)
3 	do_low(ptr);
```

```C
1 if (global_ptr != NULL &&
2 	global_ptr < high_address)
3 		do_low(global_ptr);
```

Compiler optimized away a temporary variable, thus loading from a shared variable more often than intended.



### Invented store

A compiler emitting code for *work_until_shut_down()*  might notice that *other_task_ready* is not accessed by *do_more_ work()*, and stored to on line 16. If *do_more_work()* was a complex inline function, it might be necessary to do a **register spill**, in which case one attractive place to use for temporary storage is other_task_ ready. After all, there are no accesses to it, so what is the harm?

A non-zero store to this variable at just the wrong time would result in the while loop on line 5 terminating prematurely



```C
1 if (condition)
2 	a = 1;
3 else
4 	do_a_bunch_of_stuff(&a);
```

```C
1 a = 1;
2 if (!condition) {
3 	a = 0;
4 	do_a_bunch_of_stuff(&a);
5 }
```

A compiler emitting code for above code might know that the value of a is initially zero, which might be a strong temptation to optimize away one branch by transforming this code to that in Listing 4.21. Here, line 1 unconditionally stores 1 to a, then resets the value back to zero on line 3 if condition was not set. This transforms the if-then-else into an if-then, saving one branch.

### Dead-code elimination

Dead-code elimination can occur when the compiler notices that the value from a load is never used, or when a variable is stored to, but never loaded from



## Solution

### Volatile

Volatile is a hint to the implementation to avoid aggressive optimization involving the object because the value of the object might be changed by means undetectable by an implementation. Furthermore, for some implementations, volatile might indicate that special hardware instructions are required to access the object. See 6.8.1 for detailed semantics.

1. Implementations are forbidden from tearing an aligned volatile access when machine instructions of that access’s size and type are available.Concurrent code relies on this constraint to avoid unnecessary load and store tearing
2. Implementations must not assume anything about the semantics of a volatile access, nor, for any volatile access that returns a value, about the possible set of values that might be returned.Concurrent code relies on this constraint to avoid optimizations that are inapplicable given that other processors might be concurrently accessing the location in question.
3. Aligned machine-sized non-mixed-size volatile accesses interact naturally with volatile assembly-code sequences before and after. Concurrent code relies on this constraint in order to achieve the desired ordering properties from combinations of volatile accesses



**To summarize**, the volatile keyword can prevent load tearing and store tearing in cases where the loads and stores are machine-sized and properly aligned. It can also prevent load fusing, store fusing, invented loads, and invented stores.

However, although it does prevent the compiler from reordering volatile accesses with each other, it does **<u>nothing to prevent the CPU from reordering these accesses</u>**.

Furthermore, it does nothing to prevent either compiler or CPU from reordering non-volatile accesses with each other or with volatile accesses.







