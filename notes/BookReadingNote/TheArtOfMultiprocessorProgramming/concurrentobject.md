# Concurrent Objects

## Linearizability

**Principle**:  Each method call should appear to take effect **instantaneously** at some moment between its invocation and response.

### Linearization Points

The usual way to show that a concurrent object implementation is linearizable is to identify for each method a linearization point where the method takes effect.

For lock-based implementations, each method’s critical section can serve as its linearization point. 

For implementations that do not use locking, the linearization point is typically a single step where the effects of the method call become **visible** (could be global variable) to other method calls.



## Java memory model

The Java programming language does not guarantee linearizability, or even sequential consistency, when reading or writing fields of shared objects. Why not? The principal reason is that strict adherence to sequential consistency would outlaw widely used compiler optimizations

The Java memory model satisfies the Fundamental Property of relaxed memory models: if a program’s sequentially consistent executions follow certain rules, then every execution of that program in the relaxed model will still be sequentially consistent.



**Double-checked singleton**

```Java
1 public static Singleton getInstance() {
2 	if (instance == null) {
3 		synchronized(Singleton.class) {
4 			if (instance == null)
5 			instance = new Singleton();
6 		}
7 	}
8 	return instance;
9 }
```

This pattern, once common, is incorrect. At Line 5, the constructor call appears to take place before the instance field is assigned, but the Java memory model allows these steps to occur out of order, effectively making a **partially initialized** Singleton object visible to other programs.

In the Java memory model, objects reside in a shared memory and each thread has a private working memory that contains cached copies of fields it has read or written. In the absence of explicit synchronization (explained later), a thread that writes to a field ***might not propagate that update to memory right away***, and a thread that reads a field might not update its working memory if the field’s copy in memory changes value.

At this point, we can guarantee only that a thread’s **own** reads–writes appear to that thread to happen in order, and that any field value read by a thread was written to that field



Certain statements are synchronization events. Usually, the term “synchronization” implies some form of atomicity or mutual exclusion. In Java, however, it also implies reconciling a thread’s working memory with the shared memory. Some synchronization events cause a thread to write cached changes back to shared memory, making those changes visible to other threads. Other synchronization events cause the thread to invalidate its cached values, forcing it to reread field values from memory, making other threads’ changes visible. Synchronization events are linearizable: they are totally ordered, and all threads agree on that ordering. 