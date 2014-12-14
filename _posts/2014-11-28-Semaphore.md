---
title: semaphore
layout: post
---
From [wikipedia](http://en.wikipedia.org/wiki/Semaphore_%28programming%29), a semaphore is  a variable or abstract data type that is used for controlling access, by multiple processes, to a common resource in a parallel programming or a multi user environment. 

A useful way to think of a semaphore is as a record of how many units of a particular resource are available, coupled with operations to safely.
Below is pseudo code for semaphore define.
{% highlight c++ linenos tabsize=4 %}
struct sem {
    int count; // maximum count of processes that can access the sem at the same time
    queue q; // waiting queues of processes on the semaphore.
};

{% endhighlight %}


There're 3 operations on semaphore
{% highlight c++ linenos tabsize=4 %}
// init sem's count to n
sem_init(sem, n);

// if sem's count is > 0, decrease sem's count by 1 and continue
// othewise, block and add the current thread to sem's waiting queue.
wait(sem);

// increase sem's count by 1
// if there're processes in the waiting queue, pick up one and wake it up Y
signal(sem);
{% endhighlight %}

A general scenario for semaphore is the [producer consumer problem](http://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem). You have a buffer with N unit and lots of producer and consumer processes. You need to make sure that producer won't add data into buffer if it's full and consumer won't try to remove data if the buffer is empty. 

A typical solution for producer-consumer problem with semaphore is given below:

{% highlight c++ linenos tabsize=4 %}
const int BUFFER_SIZE = 5;
queue q;
semaphore emptySem, fullSem, mutexSem;
init_sem(emptySem, BUFFER_SIZE);
init_sem(fullSem, 0);
init_sem(mutexSem, 1);
init_queue(q, BUFFER_SIZE);

void pruducer() {
    wait(emptySem);
    wait(mutexSem);
    produce(q);
    signal(mutexSem);
    signal(fullSem);
}

void consumer() {
    wait(fullSem);
    wait(mutexSem);
    consume(q);
    signal(mutexSem);
    signal(fullSem)
}

{% endhighlight %}

Line 1~7 initialized 3 semaphores and a queue. At first we've 5 empty slots and 0 full slot, so set empty sem counter to 5 and full sem to 0. mutexSem just acted like a mutex and was set to 1. 

Producer waits for empty sem at line 10. If we remove this line, more than current empty buffer's count's producers will call produce sequentially. If produce subroutine doesn't perform boundary check, there'll be a disaster. Even if produce subroutine performs boundary check and return without add extra element to queue, it still has chance to starve consumer process. Suppose we have a consumer that is waiting on mutexSem to consume the queue, in the worst case after first 5 producers that filled the buffer, the subsequent producer will 1 by 1 hold the mutexSem with nothing to do and the consumer is waiting for ever. So we must have a mechanism to block the producer while the queue is full. This is what line 10 is for.

Without line 11, multiple producer will access the queue concurrently and it's a disaster.

