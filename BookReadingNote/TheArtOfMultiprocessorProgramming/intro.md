# Introduction

## Shared objects and synchronization

On the first day of your new job, your boss asks you to find all primes between 1 and 1010 (never mind why), using a parallel machine that supports ten concurrent threads. 

As a first attempt, you might consider giving each thread an equal share of the input domain, problems are:

* Equal ranges of inputs do not necessarily produce equal amounts of work.

```Java
1 Counter counter = new Counter(1); // shared by all threads
2 void primePrint {
3 long i = 0;
4 long limit = power(10, 10);
5 while (i < limit) { // loop until all numbers taken
6 	i = counter.getAndIncrement(); // take next untaken number
7 	if (isPrime(i))
8 		print(i);
9 	}
10 }
```

A more promising way to split the work among the threads is to assign each thread one integer at a time. 

Partition the workload by threads asking for work themselves dynamically.



## Properties of mutex

* deadlock-freedom
* starvation-freedom



