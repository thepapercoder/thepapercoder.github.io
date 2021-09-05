---
layout: post
title:  "Multiprocessing in python"
date:   2021-09-02 11:01:00 +0700
categories: [python, multiprocessing, best_practices]
---
When talking about python, a lot a of time you will find people complain about how slow it's.
One way to make it faster is to use multithreading. But again, python suffer by it global interpreter lock (GIL)
which make threading in python not really usefull when you need to do heavy stuff with CPU.
So another way is to use multiprocessing instead of multithreading.

I have recently working with python multiprocessing module a lot so here are some best practises I have for myself,
a note to be notice when working with multiprocessing in general and python multiprocessing module specificly.

## Avoid using it at the first place

Multiprocessing is a complex and can lead to a lot of problems that we can not foresee.
Start and cleanup properly process when exception, hard to debug and write unittest.
Even try/catch can not save us from buggy code.

So the best is try to avoid it at the first place, make sure you really need it and understand it carefully before start using it.

## Always remember to cleanup your processes

When you start a process/subprocess, you should always handle how to close it carefully.
Subprocess if not be handle to cleanup carefully can lead to 2 cases:

* Subprocess can become orphan and keep running even when the parent process have been killed.
* Or another case can be subprocess hang forever in a loop, maybe one of subprocess thread can not join which make the whole subprocess can not join with main process.

So in the end we have a orphan process from no where keep eating resource or a program that stuck and can not shutdown.

### Subprocess become orphan

How to avoid this case? We will need to to understand why this happened at the first place.

For the first reason, a subprocess can be orphan because:

It was start with daemon flag is True, this flag will create a daemon process. What daemon process is a background process that is not under the direct control of the parent process. They working at background and doesn't require subprocess.join in main process when main process when main process shutdown.

That's why we should avoid using daemon process, only using when you really need it. An example of when daemon process is used is when running a program in the background, it no longer is directly controlled by the terminal, but it can still write to the terminal and interfere with your work. Typically a daemon will separate itself from the terminal (in addition to forking) and its output/error would be redirected to files... Another used is when the os start, it need to run process in background to handle crontab, httpd, systemd...

The downside of daemon subprocess is we need a way to keep track the progress of the subprocess, handle restart/cleanup it when it die unexpected.

Remember *ALWAYS JOIN SUBPROCESS* on shutdown of main process.

The next reason can be main process being killed brutally by example - kill -9 or by os system. In this case, nothing much we can do except asking people to be careful when doing this.

### Forever loop

For the second reason, a subprocess can hang for several reason:

* A forever loop. Typical sistuation when you are polling from a queue or waiting for a multiprocessing event signal that never came.
* An hanging non-daemon thread that is not join or can not join. This can happen if thread is stuck in a loop (like above reason with process) or by our mistake in subprocess when handle exception not good which make the thread join never call.

For this, you should handle exception (try/catch/finally) to make sure all exception are catched. Also remember that the code in exception handle and finally should not raise another exception or else, you will have a hanging thread which lead to hanging process.

## Remember to handle terminate signal (INT, TERM) in all processes

Kill singal can be send to all process in tree. This can be a problem when you are using supervisor to control your application with stop as group flag is on. Or when you hit ctrl + c in your terminal. This will send SIGINT (or SIGTERM depend on your config) to your process/subprocess which will lead to subprocess can be stopped in an unwanted way.

One way is to catch this signals and set a shutdown flag (multiprocessing.Event object) and handle gracefully shutdown subprocess by yourself.

Example:

```python
import logging
import signal
import multiprocessing


shutdown_flag = multiprocessing.Event()


def handler_stop_signals(signum, frame):
    logging.info('Getting %s signal from system, turn on shutdown flag...', str(signum))
    shutdown_flag.set()


signal.signal(signal.SIGINT, handler_stop_signals)
signal.signal(signal.SIGTERM, handler_stop_signals)
```

For multiprocessing, you can handle even better than this simple example. See how people handle kill signal in [mpire package on github](https://github.com/Slimmer-AI/mpire/blob/master/mpire/signal.py)

- There can be a case, when you will want to delay SIGINT or SIGTERM when subprocess is being create (or process pool is being create) or else you can get some weird error.
- Handle case for warm - gracefully and cool shutdown - force everything to stop (hitting ctrl + C 1 and 2 times)

*Note* Signal handler register can be call multiple time in multiple thread. But only the last handler being register will take effect. Other previous registed handler will be override and ignore.

## 3. Logging in multiprocessing

Python logging module and logger object is thread safe which allow us to share logger between threads in a safe way. But not for multiprocessing.

This is because logging module can write log to many destinations config via handler. This destinations can be file, socket, http or our custom handler.
Base on this, handler like file or socket should not be share between multiple processes. Sharing this can lead to missing log in case of rotation file of file handler, one process can override the file. From python docs:
> logging to a single file from multiple processes is not supported, because there is no standard way to serialize access to a single file across multiple processes in Python.

To handle logging for multiprocessing. You can do it in several ways:

- One way of doing this is to have all the processes log to a SocketHandler, and have a separate process which implements a socket server which reads from the socket and logs to file
- Or using a lock to make sure 1 process writing log at a time
- The third one can be using a Queue and a QueueHandler to send all logging events to one of the processes in your multi-process application

Example code for using Queue and QueueHandler to send all log to a single process and let it handle logging to files:

```python
import logging
import logging.config
import logging.handlers
from multiprocessing import Process, Queue
import random
import threading
import time

def logger_thread(q):
    while True:
        record = q.get()
        if record is None:
            break
        logger = logging.getLogger(record.name)
        logger.handle(record)


def worker_process(q):
    qh = logging.handlers.QueueHandler(q)
    root = logging.getLogger()
    root.setLevel(logging.DEBUG)
    root.addHandler(qh)
    levels = [logging.DEBUG, logging.INFO, logging.WARNING, logging.ERROR,
              logging.CRITICAL]
    loggers = ['foo', 'foo.bar', 'foo.bar.baz',
               'spam', 'spam.ham', 'spam.ham.eggs']
    for i in range(100):
        lvl = random.choice(levels)
        logger = logging.getLogger(random.choice(loggers))
        logger.log(lvl, 'Message no. %d', i)


if __name__ == '__main__':
    q = Queue()
    d = {
        'version': 1,
        'formatters': {
            'detailed': {
                'class': 'logging.Formatter',
                'format': '%(asctime)s %(name)-15s %(levelname)-8s %(processName)-10s %(message)s'
            }
        },
        'handlers': {
            'console': {
                'class': 'logging.StreamHandler',
                'level': 'INFO',
            },
            'file': {
                'class': 'logging.FileHandler',
                'filename': 'mplog.log',
                'mode': 'w',
                'formatter': 'detailed',
            },
            'foofile': {
                'class': 'logging.FileHandler',
                'filename': 'mplog-foo.log',
                'mode': 'w',
                'formatter': 'detailed',
            },
            'errors': {
                'class': 'logging.FileHandler',
                'filename': 'mplog-errors.log',
                'mode': 'w',
                'level': 'ERROR',
                'formatter': 'detailed',
            },
        },
        'loggers': {
            'foo': {
                'handlers': ['foofile']
            }
        },
        'root': {
            'level': 'DEBUG',
            'handlers': ['console', 'file', 'errors']
        },
    }
    workers = []
    for i in range(5):
        wp = Process(target=worker_process, name='worker %d' % (i + 1), args=(q,))
        workers.append(wp)
        wp.start()
    logging.config.dictConfig(d)
    lp = threading.Thread(target=logger_thread, args=(q,))
    lp.start()
    # At this point, the main process could do some useful work of its own
    # Once it's done that, it can wait for the workers to terminate...
    for wp in workers:
        wp.join()
    # And now tell the logging thread to finish up, too
    q.put(None)
    lp.join()
```

**Note** If you are using multiprocessing.Pool then the queue should be change to `multiprocessing.Manager().Queue()` not `multiprocessing.Queue()` like above example. The reason will be talk more deeply in section *What can be pass into multiprocessing.Process args*. But the reason can be shorten as Process and Pool use different way to serialize object/args when starting and send shared objects to subprocess. This reason make the different.

## Max open file or process limit config should be config

In linux, when you run the command `ulimit -a` you can see the config for per-user limitations for various resources. One of them is user limit for number of file desciptor, process/thread per ssh session your user can create.

```bash
> ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 50593
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 50593
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

When you are working with multiple thread and process and forking too many thread/process can lead to your ssh session can not start more thread and process because of the limit and make the application to raise exception like `Can not start new thread`.

In this case, we should take a look at 2 configs:

- `open files`: This will apply to all of your ssh session, limit for number of file desciptor you can use. In linux, file descriptor can represent for a lot of things, an open file, open socket, process, thread...
- `max user processes`: This will limit for each ssh session. Control number of thread and process you can create.

This configs was created to avoid fork boom. Limit one process create too many subprocess, threads that booming the os which eventually make the os crash.

**Note** One note is after [update the config](https://www.tecmint.com/increase-set-open-file-limits-in-linux/), you will need to restart your application/ssh session for new config to take effect.

## Use multiprocessing context and start subprocess method

Python process can be start with 3 kind of methods depend on the os python is running on. From [python doc](https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods)

```text
spawn
The parent process starts a fresh python interpreter process.
The child process will only inherit those resources necessary to run the process object’s run() method.
In particular, unnecessary file descriptors and handles from the parent process will not be inherited.
Starting a process using this method is rather slow compared to using fork or forkserver.

Available on Unix and Windows. The default on Windows and macOS.

fork
The parent process uses os.fork() to fork the Python interpreter.
The child process, when it begins, is effectively identical to the parent process.
All resources of the parent are inherited by the child process.
Note that safely forking a multithreaded process is problematic.

Available on Unix only. The default on Unix.

forkserver
When the program starts and selects the forkserver start method, a server process is started.
From then on, whenever a new process is needed, the parent process connects to the server and 
requests that it fork a new process. The fork server process is single threaded so it is safe 
for it to use os.fork(). No unnecessary resources are inherited.

Available on Unix platforms which support passing file descriptors over Unix pipes.
```

In our code, we can create new process using 1 of the above start method with multiprocessing context. Example:

```python
import multiprocessing as mp

def foo(q):
    q.put('hello')

if __name__ == '__main__':
    ctx = mp.get_context('spawn')
    q = ctx.Queue()
    p = ctx.Process(target=foo, args=(q,))
    p.start()
    print(q.get())
    p.join()
```

Note that objects related to one context may not be compatible with processes for a different context. In particular, locks created using the fork context cannot be passed to processes started using the spawn or forkserver start methods.

## Queue handle and process terminate caution

When using queue in multiprocessing, we will need to remember to clean it up before exit the program before joining processes/threads or else, the process using this queue can get stuck/hang and can not exit.

For example, if threre was a case when I'm using queue for logging by Queue with QueueHandler from python logging module.

```python
import logging
from multiprocessing import queue
from logging import QueueHandler, QueueListener

# ------------------------------------
# In main process
q = queue.Queue(-1)  # no limit on size
listener.start()
# run the main program...
# ...
listener.stop()

# ------------------------------------
# In subprocess, init queue handler to send log to queue
queue_handler = QueueHandler(q)
handler = logging.StreamHandler()
listener = QueueListener(q, handler)
root = logging.getLogger()
root.addHandler(queue_handler)
# The log output will display the thread which generated
# the event (the main thread) rather than the internal
# thread which monitors the internal queue. This is what
# you want to happen.
root.warning('Look out!')
```

Basically what QueueListener does is it will create a thread and listen on log queue to handle log from other process. And QueueListener.stop will wait for the sentinel message (value None by default) which will be send to log queue by QueueListener itself when we call stop.

But this can lead to a case when we the queue is already gone when we shutdown the application and collected by gc. I was once put the listener.stop function inside a \_\_del\_\_ method of a object to manage subprocess.

After this, I got a hanging forever queue listesner thread and process never exit. After checking, I can see in listerner thread, we still can enqueue sentinel message to log queue sucessfully, but checking the queue, no message was found. So no sentinel messge was recieved, thread hang happened.

After digging, I found an warning in python docs for terminating process

```text
Warning: If this method is used when the associated process is using a pipe or queue then 
the pipe or queue is liable to become corrupted and may become unusable by other process.
Similarly, if the process has acquired a lock or semaphore etc. then terminating it is
liable to cause other processes to deadlock.
```

```text
Warning As mentioned above, if a child process has put items on a queue 
(and it has not used JoinableQueue.cancel_join_thread), then that process will not terminate 
until all buffered items have been flushed to the pipe.
This means that if you try joining that process you may get a deadlock unless you are sure 
that all items which have been put on the queue have been consumed. Similarly, if the child
process is non-daemonic then the parent process may hang on exit when it tries to join all 
its non-daemonic children.
```

And

```text
Warning multiprocessing.pool objects have internal resources that need to be properly managed (like any other resource) by using the pool as a context manager or by calling close() and terminate() manually. Failure to do this can lead to the process hanging on finalization.
Note that is not correct to rely on the garbage colletor to destroy the pool as CPython does not assure that the finalizer of the pool will be called (see object.__del__() for more information).
```

So there was 2 problems, I was calling stop log listener obj in del method which is not good since del method was call by python internally and we have no control of when this method will be call.

And second, calling terminate process is not recommend by python, we should use shutdown flag to signal process to stop and then join the process.

## What can be pass into a multiprocessing.Process args

Python use pickle for serialize and deserialize object/data. So anything that can be pickled can be send to Process class args.

Here is the list of type that can be send via pickle:

- None, True, and False
- integers, floating point numbers, complex numbers
- strings, bytes, bytearrays
- tuples, lists, sets, and dictionaries containing only picklable objects
- functions defined at the top level of a module (using def, not lambda)
- built-in functions defined at the top level of a module
- classes that are defined at the top level of a module
- instances of such classes whose \_\_dict\_\_ or the result of calling \_\_getstate\_\_() is picklable

But you might notice that there is some other type of object can be pass to a process as args like Queue, Event, Lock from multiprocessing module and other type of object from multiprocessing Manager.

This is because multiprocessing module using an *extended version* of of pickle which can transfer stuff like Queue and Event to subprocess.

- When a multiprocessing.Queue is passed to a child process, what is actually sent is a file descriptor (or handle) obtained from pipe
- With Lock you can check the anwser in [this](https://www.py4u.net/discuss/18341) to know more about how lock being share between process in python for both linux and windows
- With object from multiprocessing Manager, actually you still need this object to be able to be serializeble by pickle to be send over pipe or network.

**Note** The serialization for process args is different in multiprocessing.Pool vs multiprocessing.Process. So best recommand is when using Pool, always use multiprocessing.Manager to share your resource like Lock, Queue.

## Programming guideline

This is a copy version of the guideline to work with multiprocessing module from [python official docs](https://docs.python.org/3/library/multiprocessing.html#programming-guidelines)

### All start methods

The following applies to all start methods.

#### Avoid shared state

As far as possible one should try to avoid shifting large amounts of data between processes.

It is probably best to stick to using queues or pipes for communication between processes rather than using the lower level synchronization primitives.

#### Picklability

Ensure that the arguments to the methods of proxies are picklable.

#### Thread safety of proxies

Do not use a proxy object from more than one thread unless you protect it with a lock.

(There is never a problem with different processes using the same proxy.)

#### Joining zombie processes

On Unix when a process finishes but has not been joined it becomes a zombie. There should never be very many because each time a new process starts (or active_children() is called) all completed processes which have not yet been joined will be joined. Also calling a finished process’s Process.is_alive will join the process. Even so it is probably good practice to explicitly join all the processes that you start.

#### Better to inherit than pickle/unpickle

When using the spawn or forkserver start methods many types from multiprocessing need to be picklable so that child processes can use them. However, one should generally avoid sending shared objects to other processes using pipes or queues. Instead you should arrange the program so that a process which needs access to a shared resource created elsewhere can inherit it from an ancestor process.

#### Avoid terminating processes

Using the Process.terminate method to stop a process is liable to cause any shared resources (such as locks, semaphores, pipes and queues) currently being used by the process to become broken or unavailable to other processes.

Therefore it is probably best to only consider using Process.terminate on processes which never use any shared resources.

#### Joining processes that use queues

Bear in mind that a process that has put items in a queue will wait before terminating until all the buffered items are fed by the “feeder” thread to the underlying pipe. (The child process can call the Queue.cancel_join_thread method of the queue to avoid this behaviour.)

This means that whenever you use a queue you need to make sure that all items which have been put on the queue will eventually be removed before the process is joined. Otherwise you cannot be sure that processes which have put items on the queue will terminate. Remember also that non-daemonic processes will be joined automatically.

An example which will deadlock is the following:

```python
from multiprocessing import Process, Queue

def f(q):
    q.put('X' * 1000000)

if __name__ == '__main__':
    queue = Queue()
    p = Process(target=f, args=(queue,))
    p.start()
    p.join()                    # this deadlocks
    obj = queue.get()
```

A fix here would be to swap the last two lines (or simply remove the p.join() line).

#### Explicitly pass resources to child processes

On Unix using the fork start method, a child process can make use of a shared resource created in a parent process using a global resource. However, it is better to pass the object as an argument to the constructor for the child process.

Apart from making the code (potentially) compatible with Windows and the other start methods this also ensures that as long as the child process is still alive the object will not be garbage collected in the parent process. This might be important if some resource is freed when the object is garbage collected in the parent process.

So for instance

```python
from multiprocessing import Process, Lock

def f():
    ... do something using "lock" ...

if __name__ == '__main__':
    lock = Lock()
    for i in range(10):
        Process(target=f).start()
should be rewritten as

from multiprocessing import Process, Lock

def f(l):
    ... do something using "l" ...

if __name__ == '__main__':
    lock = Lock()
    for i in range(10):
        Process(target=f, args=(lock,)).start()
```

#### Beware of replacing sys.stdin with a “file like object”

multiprocessing originally unconditionally called:

```python
os.close(sys.stdin.fileno())
```

in the multiprocessing.Process._bootstrap() method — this resulted in issues with processes-in-processes. This has been changed to:

```python
sys.stdin.close()
sys.stdin = open(os.open(os.devnull, os.O_RDONLY), closefd=False)
```

Which solves the fundamental issue of processes colliding with each other resulting in a bad file descriptor error, but introduces a potential danger to applications which replace sys.stdin() with a “file-like object” with output buffering. This danger is that if multiple processes call close() on this file-like object, it could result in the same data being flushed to the object multiple times, resulting in corruption.

If you write a file-like object and implement your own caching, you can make it fork-safe by storing the pid whenever you append to the cache, and discarding the cache when the pid changes. For example:

```python
@property
def cache(self):
    pid = os.getpid()
    if pid != self._pid:
        self._pid = pid
        self._cache = []
    return self._cache
```

For more information, see bpo-5155, bpo-5313 and bpo-5331

### The spawn and forkserver start methods

There are a few extra restriction which don’t apply to the fork start method.

#### More picklability

Ensure that all arguments to Process.__init__() are picklable. Also, if you subclass Process then make sure that instances will be picklable when the Process.start method is called.

#### Global variables

Bear in mind that if code run in a child process tries to access a global variable, then the value it sees (if any) may not be the same as the value in the parent process at the time that Process.start was called.

However, global variables which are just module level constants cause no problems.

#### Safe importing of main module

Make sure that the main module can be safely imported by a new Python interpreter without causing unintended side effects (such a starting a new process).

For example, using the spawn or forkserver start method running the following module would fail with a RuntimeError:

```python
from multiprocessing import Process

def foo():
    print('hello')

p = Process(target=foo)
p.start()
```

Instead one should protect the “entry point” of the program by using if \_\_name\_\_ == '\_\_main\_\_': as follows:

```python
from multiprocessing import Process, freeze_support, set_start_method

def foo():
    print('hello')

if __name__ == '__main__':
    freeze_support()
    set_start_method('spawn')
    p = Process(target=foo)
    p.start()
```

The freeze_support() line can be omitted if the program will be run normally instead of frozen.

This allows the newly spawned Python interpreter to safely import the module and then run the module’s foo() function.

Similar restrictions apply if a pool or manager is created in the main module.
