# Gilectomy

---

* [Larry Hastings](http://www.larryhastings.com/) - Release Manager for Python 3.4 and 3.5 - python.org

The [GIL](https://wiki.python.org/moin/GlobalInterpreterLock) does not allow use of multiple cores.

* Removing the GIL (Global Interperter Lock)
  * Technical Considerations
    * Reference Counting
    * C Extensions and Parallism
      * Protected by the GIL - only run one thread at a time - allows for unsafe operations
    * Atomicity
  * Political Considerations
    * Don't hurt single threaded performance (Don't make them slower) //This is HARD//
    * Don't break C Extensions like Python 3.
    * Don't make it too complicated - the GIL is easy/simple internally. Very easy to work with and debug. To sacrifice this simplicity to get rid of the GIL is a bad idea.
* Tracing Garbage Collection
  * Doesn't hurt single threaded performance
  * Breaks C Extensions
  * Probably makes it complicated
* Software Transactional Memory
  * ACID transactions.
  * Do stuff, check if the memory has changed - COMMIT.
  * Doesn't hurt performance
  * Breaks C Extensions
  * Really fucking complicated

### Proposal

* Reference Counting
  * Switch to atomic increment and decrement (Use CPU instruction) to add/remove one.
  * Global & Static variables is not a huge deal
  * It's going to break C extensions and there's nothing we can do :(
  * We will lock every thing - Lists, Dicts using a PY_LOCK(o) API.
    * PyTypeObject -> ob_lock(o) __LOCK__
    * PyTypeObject -> ob_unlock(o) __UNLOCK__
* What needs a lock?
  * All mutable locks
    * Str has hash() which computes a has which may lead to leads in environments where it's computed independently across cores with UTF-8.
* We will break C Extensions BUT we will allow two builds - one with a GIL and one without a GIL.
  * `./configure -with-gil-initmodule`
  * `./configure -without-gil-initmodule`
* New API = Enforce best practices.
  * Use PyType_Ready
  * PEP 489 ( Multiphase c ext initialization )
* It's complicated 😞

### [It isn't easy to remove the GIL](http://www.artima.com/weblogs/viewpost.jsp?thread=214235)

* It's easy
  1. Atomic incr & decrement
  2. Pick the Lock
  3. Lock DictObject
  4. Lock ListObject
  5. Lock 10 FreeLists
  6. Disable Garbage Collection - Tracking and Untracking (GC)
  7. Murder the GIL :3
  8. Switch Python from storing the current thread state (It's stored in thread local storage - change CPython to look it up)
  9. Fix tests.

### Benchmarking
  * Fibonnachi Sequence
  * Gilectomy version is slower than GIL Python.
    * Slow with Wall
    * Slow with CPU Time.
  * Why?
    * Some of it is due to the Locking
    * Primary is Syncronization issues and Cache-Misses
    * What makes CPUs fast?
      * Cache speed - L1, L2, L3 - then to RAM in terms of speed
      * In the Gilectomy version - Cache's are always missed - Cache never stays warm.
      * Each core has to go back to RAM a lot more often than the GIL version.
      * There's a solution!

* [Buffered Reference Counting](http://www.cs.tau.ac.il/~maon/teaching/2014-2015/seminar/seminar1415a-lec12-conc-rc-sum.pdf)
  * Get rid of CPU Atomic increment and decrement
  * Create a transaction log.
    * Syncronized atomic increment and decrement.
  * Using seperate Incr and Decr transaction logs.
  * Benchmarks!
    * It's better - however it's not better than GIL -yet-

### How do we make it more correct?
  * Fix garbage Collection
    * Stop the world
    * Buffered track/Untracking
  * Auto-Lock around C Extensions - ensures capatability - mostly.
  * Preserve returned borrowed references.
