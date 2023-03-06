# The Readers-Writers Problem

At a generic level, the readers-writers problem is a part of a larger subset of computing problems arising in concurrency. It is actually a set of 3 problems, with one problem
arising from the solution of the previous.
Here, we will define the first readers-writers problem, proposed and solved by Courtois.

Say, several concurrent processes are sharing some set of global variables. More specifically, 
let's say that a database is being shared by a set of concurrent processes running in a computing environment.
Let us divide these processes in terms of the nature of operations they perform on the database.
 * Some processes may only want to read the database. (Referred to as `Reader`)
 * Some processes may want to perform both read and write operations in the database. (Referred to as `Writer`)
 
 We define the following condition to assure concurrency among the processes - 
 > Exclusive access to the database be provided - Either to a sole writer, or a set of readers. 
 
 Here, the sole constraint to be satisfied is that no reader be kept waiting unless a writer has already secured access to use the shared object.
 Or - _No reader should wait for other readers to finish reading just because the writer is waiting._
 
 
---
In order to delve deeper into the approach to solve this initial problem and the potential problems of this approach, we need to know about Semaphores.

A semaphore S is an integer variable that, apart from initialization, is accessed only through two standard atomic operations: `wait()` and `signal()`.
The `wait()` operation was originally termed `P` (from the Dutch proberen, “to test”); `signal()` was originally called `V` (from verhogen, “to increment”).
 `wait()` and `signal()` are performed atomically, i.e., when one method modifies a particular semaphore value, no other method must be allowed to modify the same value.

Each process that wishes to use a resource performs a `wait()` operation on a semaphore that provides mutually exclusive access to that resource. 
When a process releases a resource, it performs a `signal()` operation.

---

```C
shared data : 
struct semaphore{
int value;     //Default value provided as 1 (binary semaphore)
struct process *list; //A waiting queue associated with each semaphore object.
};

```

Then the `wait()` operation is implemented as - 

```C
wait(semaphore *S) {
 S->value--;
 if (S->value < 0) {
   S->list.enqueue(process);
   //i.e., add to the list of processes waiting on a semaphore.
   process.block();
   //The caller process is sent to a blocked state using this call.
 }
}


```

> When a process executes the `wait()` method and finds that the semaphore value is not positive, it must wait. Rather than busy waiting, we block that process of the CPU itself.

The `signal()` operation is implemented as - 

```C 
signal(semaphore *S) {
 S->value++;
 if(S->value <= 0) {
   process* next_P = S->list->pop();
   //A process with identifier P is woken up.
   wakeup(next_P);
   // The wakeup() method resumes the execution of a blocked process P.
   }
}


```

> When some other process is executing `signal()` on a semaphore S, a blocked process will be woken up from the associated list of waiting processes for S.

---

## The Classical Solution (Most Intutive)

We define the following shared data and variables :
(i) `Data set` - can be a file or simply a global variable, from which the readers try to read and writers try to read/write.
(ii) Integer `readcount` intialized to 0.
     * Indicates the number of readers reading the data at the moment.
(iii) Semaphore `mutex` intialized to 1.
     * Protects the readcount variable, as multiple readers might try to modify it.
(iv) Semaphore `wrt` initialized to 1.
     * Protects the dataset from concurrent access by multiple readers and writers.
     
```C
int readcount = 0;
semaphore *mutex = New semaphore(1);
semaphore *wrt = New semaphore(1);

```
The structure of the writer process is then - 

### Classical Writer Process

```C 
do {
   /*
   .
   Initial Section
   .
   */
   wait(wrt);
   /*.
   ...
   Writing performed
   ...
   */
   signal(wrt);
} while(TRUE);


```

> The writer process will try to acquire the `wrt` semaphore. If succesful, it can perform a write operation on the data. When finished, it will increment the semaphore.


The structure of the reader process is then - 

### Classical Reader Process
```C 
do {
     /* 
     .
     Initial Section
     .
     */
     wait(mutex);
     readcount++;
     if(readcount==1)
       wait(wrt);
     signal(mutex);
     /*.
     ...
     Reading is performed
     ...
     */
     wait(mutex);
     reacount--;
     if(readcount == 0) {
        signal(wrt);
     }
     signal(mutex);
} while(TRUE);


```
 
> The first and the last reader processes do something different from the others. They are responsible for acquiring and releasing exclusive access to the shared data for their kind.

> The first reader increments `readcount` to 1, and then tries to decrement the semaphore `wrt`. When successful, it will have secured exclusive access to the shared data for all the concurrently running read processes. After all the readers finish, only then the `wrt` semaphore is incremented, and the grasp over data released.

---

### POSSIBILITY OF STARVATION
Consider the case when a certain number of reader processes are underway in reading the shared data. Then, a `writer` process arrives and tries to perform `wait()` operation on `wrt`, but is sent to the blocked state. 

Now, let's say that a steady stream of new reader processes keep arriving in the ready queue. The `wrt` semaphore won't become positive till readcount reaches 0. i.e., the 
`writer` process will not be woken up till that happens.

So, it will forever starve in the waiting queue of the semaphore `wrt`.

_Hence, the most intuitive solution allows the writer processes to starve under certain conditions. We need some mechanism to avoid this._

---
> We need a mechanism that will give precedence to the `writer` process, over the set of reader processes that arrive after the `writer` tried to perform `wait(wrt)`.


## STARVE-FREE SOLUTION : 

We define an additional sempahore - the `entry_mut`, initialized to 1.
```C
semaphore *entry_mut = New semaphore(1);

```


_THEN, THE `writer` process can now be implemented as_ - 

### Starve-Free Writer Process

```C
do{
   /*
   .
   Initial Section
   .
   */
   
    wait(entry_mut);
    wait(wrt);

    /**
     * 
     * Critical Section
     * 
    */

    // Exit Section
    signal(wrt);
    signal(entry_mut);
   /*.
   ...
   Writing performed
   ...
   */
}while(true);


```

The code for the `reader` process can be written as - 

### Starve-Free Reader Process

```C 
do {
     /* 
     .
     Initial Section
     .
     */
     wait(entry_mut);
     wait(mutex);
     readcount++;
     if(readcount==1)
       wait(wrt);
     signal(mutex);
     signal(entry_mut);
     /*.
     ...
     Reading is performed
     ...
     */
     wait(mutex);
     reacount--;
     if(readcount == 0) {
        signal(wrt);
     }
     signal(mutex);
} while(TRUE);


```

#### What was done here ? 

> Intuitively, we tried to bridge the discrimination between the readers and writer processes by introducing a new mutex lock. We sort of protect the entry section for the readers code. The reader and writer processes will fall into the same waiting queue whenever required, and this will ensure that a definite waiting time results for both readers and writers if there's an overwhelming stream of the other processes.

## (i) MUTUAL EXCLUSION
As discussed before, `mutex` protects the readcount variable, while `wrt` provides exclusive access to a writer vs a set of readers. The initial system allowed for mutual exclusion in the critical regions of the respective processes, where they performed either reads, or read & writes on shared data.
The newly proposed solution is also mutually exclusive, as `entry_mut` is responsible for putting a reader process in its rightful place, providing mutually exclusive access to the problem as a whole.

## (ii) PROGRESS
The readers and writers cannot enter deadlock for the proposed solution. It is guaranteed that the stream of reader (or writer) processes present in the waiting queue will be cleared even if a single writer(or reader) requests access.

## (iii) BOUNDED WAITING TIME 
As the `entry_mut` semaphore queues up both readers and writers in a single queue, providing mutual access, bounded waiting time is satisfied. Let's say there is a steady stream of reader processes flowing in. Then, at the point of time when a single writer process requests access to the data, it will be guaranteed to have its requested access in a finite amount if time.


## REFERENCES - 

(i) Abraham Silberschatz, Peter B. Galvin, Greg Gagne - Operating System Concepts-Wiley (2012)
(ii) CS-342 Operating Systems - Asst. Prof. Dr. İbrahim Körpeoğlu (2008-2009- Spring) {VIDEO LECTURES - Bilkent University}
