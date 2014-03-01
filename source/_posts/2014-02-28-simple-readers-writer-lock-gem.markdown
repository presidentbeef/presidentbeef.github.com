---
layout: post
title: "Simple Readers-Writer Lock Gem"
date: 2014-02-28 08:53
comments: true
categories: 
---

A [readers-writer lock](http://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock) can be used to allow many concurrent read-only operations on a resource but ensure exclusive access for modifying operations performed by "writers". For my purposes, I needed a readers-writer lock at the thread level, basically to control access to a shared array. In my scenario, the array is accessed through a server which may server many clients at once. Some requests will be to read elements from the array, while other requests might be adding elements to the array. There is no reason to restrict reads to one client at a time, but elements need to be added while no other client is reading or writing to the array.

[My implementation](https://github.com/presidentbeef/rwlock) is very simple (the entire `RWLock` class is 25 lines of code) because it relies on Ruby's [SizedQueue](http://rdoc.info/stdlib/thread/SizedQueue) class. `SizedQueue` provides a thread-safe queue with a maximum size. If a thread attempts to add elements to a queue that is full, it will be blocked until an element is removed from the queue to make room. This is a key piece of funtionality used for the readers-writer lock implementation.

The `RWLock` class only really needs to provide two methods: one to provide read access, and one to provide write access. Since this is Ruby, the methods will take a block to execute the reading/writing code:

```ruby
class RWLock
  def read_sync
    #lock magic
    yield
    #lock magic
  end

  def write_sync
    #lock magic
    yield
    #lock magic
  end
end
```

The internal state of the lock will be a `SizedQueue` and a `Mutex`.

```ruby
  def initialize max_size = 10
    @write_lock = Mutex.new
    @q = SizedQueue.new(max_size)
  end
```

The `SizedQueue` will essentially be used as a counting semaphore. Each time a reader enters `read_sync`, the lock will push an element onto the queue. What the element actually is doesn't matter, but I used `true` because it's cheap. If the queue is full, the reader will block until a space has opened up.

```ruby
  def read_sync
    @q.push true
    yield
  ensure
    @q.pop
  end
```

When a writer calls `write_sync`, it synchronizes on the mutex to prevent multiple concurrent writers. Then it adds *n* elements to the queue, where *n* is equal to the maximum size of the queue.

This has two effects: first, the writer is forced to wait for all current readers to finish. Second, it essentially prevents any new readers from gaining access (there is a small chance one will sneak in, but the writer will still have to wait for it).

```ruby
  def write_sync
    @write_lock.synchronize do
      @q.max.times { @q.push true }

      begin
        yield
      ensure
        @q.clear
      end
    end
  end
```

Once the writer is finished, the queue is cleared, allowing all waiting readers to jump in. It is most likely waiting readers will get in before waiting writers, since the write mutex is held while the queue is emptied, but no effort is made to guarantee that one way or another. In practice, though, this seems to balance well between readers and writers.

One obvious downside of this overall approach is the `SizedQueue` limits the number of concurrent readers. A larger queue will cause writers to wait longer (assuming many readers) while a smaller queue may cause readers to wait on other readers. The upside is readers cannot monopolize the resource and cause writer starvation.

Unfortunately, `SizedQueue#clear` has been broken forever, since it was simply inherited from `Queue` and didn't actually notify waiting threads that the queue is empty. For some reason, this does not appear to matter in Ruby 1.8, but in Ruby 1.9 and 2.0 it caused a deadlock.

This has been fixed in Ruby 1.9.3p545 and 2.1.1. For broken versions, the `RWLock` gem monkey-patches `SizedQueue` to fix the behavior. Unfortunately, Ruby 2.0 also had a bug in `SizedQueue#push`, so it is completely incompatible. The code does work under JRuby, but there are faster implementations using Java primitives.

RWLock is available as [a gem](https://rubygems.org/gems/rwlock) and of course the [code is on GitHub](https://github.com/presidentbeef/rwlock).
