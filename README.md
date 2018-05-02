# Bloaty McBloatface

This Rails app tries to reproduce the memory bloat that seems to be
endemic with 64-bit Linux deployments.

## Ruby Memory

Heavyweight Ruby processes like Sidekiq often grow from 200MB on startup
to 2GB or more over time.  There can be many causes:

1. Unintended memory references in Ruby code (e.g. a cache)
2. A memory leak in MRI or native extension
3. Ruby heap fragmentation http://rubykaigi.org/2017/presentations/nateberkopec.html
4. Lower-level fragmentation or overallocation in glibc https://www.speedshop.co/2017/12/04/malloc-doubles-ruby-memory.html

This app attempts only to reproduce the last one.  On a side note, it's
safe to say that, outside of ruby-core, Nate Berkopec is the expert on
Ruby memory issues.  [Contract him](https://speedshop.co) if you need help.

## The Problem

Linux glibc's memory allocator uses "arenas" to avoid thread-contention
during memory allocation.  An arena is a 64MB block of memory.  When
a thread wants a piece of memory, glibc will:

1. find and lock an available arena
2. reserve a piece of memory within that arena
3. unlock the arena

The problem comes in Step 1.  Each thread defaults to using the last
arena it used.  If all arenas are locked, glibc will create a new arena.
This is one way multithreaded Ruby processes on 64-bit Linux
will bloat up over time... in 64MB chunks.

glibc will create up to #cores * 8 arenas.  Sidekiq, by default, will
have approximately 25 threads running normally.  If all threads are very busy,
it's possible glibc has allocated 20 or more arenas, meaning 1280MB of
RAM.  Each thread by default draws from the arena it last used so it's
possible that each of those 20 arenas is used.

As each arena fragments, it will eventually be unable to fulfill a
malloc() call and a new arena created which is used thereafter for that
thread.  In this way, glibc + lots of threads leads to a lot of wasted
space.

> **Note**: this is my best understanding of the problem. It may be
> slightly wrong, mostly wrong or [not even wrong](https://en.wikipedia.org/wiki/Not_even_wrong).

## The Solution

1. Use jemalloc.  One of its explicit goals is to minimize fragmentation.
2. Set `MALLOC_ARENA_MAX=2`.

This will force glibc to stick with two working arenas, no matter how
contended.  Heavily multithreaded systems like Sidekiq might see a small
performance dip but with greatly reduced memory bloat.

### Setup

* Install Docker/OSX.
* Run `docker-compose up`
* Run `docker-compose exec bloat bundle exec rails db:seed`
* Run `ab -k -c 25 -n 1000 http://localhost:3000/posts`

We run Rails in production mode so it's not constantly trying to reload
code.  We configure Puma to use 25 threads in one process, identical to
Sidekiq and enable database caching so networking with the database is not an issue.
