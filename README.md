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
