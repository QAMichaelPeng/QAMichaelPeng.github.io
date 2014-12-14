---
title: read write lock
layout: post
---
Read write lock is widely used in concurrent environments. It enables multiple reader to read resource concurrently but only one writer can write to it without any read action at the same time. The concept is easy to understand, but to implement a well behaved read write lock is not so easy.

Implement 1: two mutex for read and write, one reader count
{% highlight c++ linenos tabsize=4 %}
class RWLock
{
private:
    mutex rlock_;
    mutex wlock_;
    int readers_ = 0;
public:
    void LockWrite() {
        wlock_.lock();
    }

    void UnlockWrite() {
        wlock_.unlock();
    }

    void LockRead() {
        lock_guard<mutex> guard(rlock_);
        if (readers_++ == 0) {
            LockWrite();
        }
    }

    void UnlockRead() {
        lock_guard<mutex> guard(rlock_);
        if (--readers_ == 0) {
            // unlock reader thread may be different from the thread who locked the writer
            UnlockWrite();
        }
    }
};
{% endhighlight %}

There's a big drawback in the implementation that the last reader thread who call unlock writers may be different from the first thread who lock writers. It will cause undefine behavor. 

* [std::mutex::unlock](http://www.cplusplus.com/reference/mutex/mutex/unlock/):

    If the mutex is not currently locked by the calling thread, it causes undefined behavior.
* [ReleaseMutex](http://msdn.microsoft.com/en-us/library/windows/desktop/ms685066(v=vs.85).aspx):

    The ReleaseMutex function fails if the calling thread does not own the mutex object.
* [pthread_mutex_unlock](http://www.lehman.cuny.edu/cgi-bin/man-cgi?pthread_mutex_lock+3)

    If a thread attempts to unlock a mutex that it has not  locked  or a mutex that is unlocked, undefined behavior results.



To fix the underfined behavior issue, we can use condition variable to notify between the reader and writer processes.

{% highlight c++ linenos tabsize=4 %}
class RWLockCV{
    mutex lock_;
    condition_variable cv_;
    int writers_ = 0;
    int readers_ = 0;
public:
    void LockWrite() {
        unique_lock<mutex> lock(lock_);
        while (readers_ > 0 || writers_ > 0) cv_.wait(lock);
        ++writers_;
    }

    void UnlockWrite() {
        unique_lock<mutex> lock(lock_);
        --writers_;
        cv_.notify_all();
    }

    void LockRead() {
        unique_lock<mutex> lock(lock_);
        while (writers_ > 0) cv_.wait(lock);
        ++readers_;
    }

    void UnlockRead() {
        unique_lock<mutex> lock(lock_);
        if (--readers_ == 0) cv_.notify_all();
    }
};
{% endhighlight %}

In the above code every lock and unlock pair are in a function's scope and no lock is unlocked in another process/thread. How does it work? Let's study several cases.
<ol> 
<li> Two writer process. 
{% highlight c++ linenos tabsize=4 %}
// process 1 at time t1
writer1.LockWrite();
// process 2 at time t2
Writer2.LockWrite();
// process 1 at time t3
writer1.UnlockWrite();
{% endhighlight %}

At time t1, readers and writers are 0, writer1 set writers_ to 1.

At time t2, writer2 see writers_ > 0, unlock and wait on cv

At time t3, writer1 set writers_ to 0, and notify all processes waiting on cv, then release the lock. Writer2 is notified, then tried to acquire the lock and acquired the lock while writer1 released it. 
</li>
<li>

One writer and one reader. Writer first acquired the lock.

{% highlight c++ linenos tabsize=4 %}
// process 1 at time t1
writer1.LockWrite();
// process 2 at time t2
reader1.LockRead();
// process 1 at time t3
writer1.UnlockWrite();
{% endhighlight %}

Similar analysis as above. Just replace writer2 with reader1.
</li>
<li>
One writer and two readers

{% highlight c++ linenos tabsize=4 %}
// process 1 at time t1
writer1.LockWrite();
// process 2 at time t2
reader1.LockRead();
// process 3 at time t3
reader2.LockWrite();
// process 1 at time t4
writer1.UnlockWrite();
{% endhighlight %}
At t4 when writer1 set writers to 0 and notify all, both reader1 and reader2 will wake up and try to acquire the lock. After writer1 released the lock, if reader2 acquired the lock, it will then set readers_ to 1 and release the lock, began reading. Then reader 1 will acquire the lock, set readers_ to 2, released the lock and began read.

From this case we see the importance of notify_all. If we use notify_one in UnlockWriter, only one reader will be token up, another will still hang on the cv.
</li>
<li>
One reader and two writers.

{% highlight c++ linenos tabsize=4 %}
// process 1 at time t1
reader.LockRead();
// process 2 at time t2
writer1.LockWrite();
// process 3 at time t3
writer2.LockWrite();
// process 1 at time t4
reader1.UnlockWrite();
{% endhighlight %}
At t4, when reader1 set readers_ to 0 and call cv_.notify_all, both writer1 and writer2 will wake up and try to acquire lock. After reader1 released the lock, if writer2 acquired the lock, increment writers_ to 1 and release the lock, writer1 then acquired the lock, but since writers_ is now 1, it can do nothing but wait again. <b>It's a waste if we have many writers waiting on the cv.</b> Only one can get the chance to execute, all others are woken up and soon sleep again. To solve the issue, we should use two cv: one for reader and call notify_all on that when unlock, another for writer and call notify_one when unlock.
</li>
<li>
Multiple readers starve writer.
{% highlight c++ linenos tabsize=4 %}
// process 1 at time t1
reader.LockRead();
// process 2 at time t2
writer1.LockWrite();
// process 3 at time t3
reader2.LockRead();
// process 1 at time t4
reader1.UnlockWrite();
// process 4 at time t5
reader3.LockRead();
// process 3 at time t6
reader2.UnlockWrite();
{% endhighlight %}

At t3, when reader2 tried to lock, it got the lock and set readers_ to 2. At t4, after reader1 unlocked, writer1 was woken up but since readers_ is 1 now, it had to wait again. At t6 when reader2 released the lock, there was a reader 3 and writer1 had to wait again. <b>So if there always came a reader before the last reader called UnlockReader, writer1 would never get a chance to execute.</b> This issue is even harder to solve. We need a queue to recors each read/write request time and make those request served in the order they come as possible as we can. In some cases we can't gurante that if a reader comes while the buffer is empty.
</li>
</ol>
