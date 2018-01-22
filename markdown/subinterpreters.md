## Actually, No

Note: Remember how I said there was a module providing a REST API?  That's a
web service.  Later, another separate module was added to provide a simple read
only web accessible cluster status dashboard.  That's also a web service.  Both
were implemented using CherryPy.  I wanted to try the dashboard, so I loaded
it on a test cluster which was already running the REST API module, which is
actually where my personal odyssey with this stuff began.


CherryPy: There Can Be Only One
```py
    cherrypy.config.update({'server.socket_port': ....})
    cherrypy.tree.graft(...)  # or cherrypy.tree.mount(...)
    cherrypy.engine.start()
    cherrypy.engine.block()

```

Note: Both the rest module and the status dashboard followed this pattern.
I'm pretty usre nobody ever imagined doing this with CherryPy -- trying
to run two of them at once in the same process.  That set of statements will
be run possibly in lockstep by two seaparate threads, so the first one that
makes it the whole way through wins, if you're lucky.  Whichever thread hits
cherrypy.engine.start() last fails complaining it can't be called more than
once from the same thread (even though the modules run in separate OS threads,
they're both run by the same python interpreter, so the complaint about being
run in the same thread is a bit of a lie).

So I brought that up on the ceph-devel mailing list.  One possiblity; run
myltiple cherrypys on different ports, or use virtualhosts, but there's still
the problem of ensuring start() only gets called once, so maybe a separate
cherrypy module and module dependencies.  Or, we could look at using
python sub-interpreters, which are pretty much isolated from each other.


> Doing separate sub-interpreters would also be an option that would
give us more robustness generally in the face of python modules that
do global things.  I don't think there was any fundamental reason I
didn't use sub-interpreters when writing this stuff originally,
they're just a comparatively sparesely documented part of CPython.
&mdash; John Spray


## RTFM

Note:

* Py_NewInterpreter() creates a new sub-interpreter which is an almost
totally separate environment with its own set of imported modules, sys,
stdout, stderr etc.
* It returns a pointer to a PyThreadState (this doesn't create a thread,
it's just the name of a data structure that somehow represents a thread)
* Call Py_ThreadState_Swap() to switch between sub-interpreters


> I think I'm liking the [sub-interpreter] option for general robustness.  My best
> guess right now is that's probably adding a Py_NewInterpreter() call to
> MgrPyModule's constructor [...]
>
> Given the sparseness of the docs it's probably best to just try it and
> see what catches on fire.  I'll keep you posted.
&mdash; Tim Serong


## FM Conflicts

Note: Remember PyGilState_Ensure() and PyGILState_Relase()?


> ...combining [sub-interpreters] with PyGILState_*() APIs is delicate, because these APIs assume a bijection between Python thread states and OS-level threads, an assumption broken by the presence of sub-interpreters.

https://docs.python.org/3/c-api/init.html

Note: I didn't even know bijection was a word, but OK.  It means that those APIs assume each python thread state is paired with exactly one OS-level thread, and vice versa.  But when we introduce subinterpretrs, one OS thread can have many python alleged thread states, which breaks that model.


>  Python supports the creation of additional interpreters (using Py_NewInterpreter()), but mixing multiple interpreters and the PyGILState_*() API is unsupported.

https://docs.python.org/3/c-api/init.html

Note: So let's just assume we can't use those APIs.


## The GIL Redux

Note: I needed to do whatever it was that those PyGILState_ APIs were doing,
so I made a little C++ GIL class.


```c++
class Gil {
private:
  PyThreadState *pThreadState;

public:
  Gil(PyThreadState *ts) : pThreadState(ts) {
    // Acquire the GIL, set the current thread state
    PyEval_RestoreThread(pThreadState);
  }

  ~Gil() {
    PyEval_SaveThread();    // Release GIL, reset TS to NULL
  }
};
```

Note: Do *not* nest these, or the second one will deadlock.  Instantiate one
to acquire the GIL, and when it goes out of scope, the GIL is released.

So let's combine that with a refactored MgrPyModule class, where each
instance of that class spawns a new Python subinterpreter, and keeps track of
that subinterpreter's thread state.


```c++
int PyModules::init() {
  // ...
  std::string sys_path = out_custom_module_path;
  PyEval_InitThreads();              // Init and acquire GIL

  pMainThreadState = PyEval_SaveThread();        // Drop GIL

  for (/* each module_name */) {
    auto mod = std::unique_ptr<MgrPyModule>(
               new MgrPyModule(module_name, sys_path,
                               pMainThreadState));
    mod->load();
  }
  return 0;
```

Note: Instaed of PySys_SetPath, we're figuring out the path and passing that to
MgrPyModule (remember separate sub-interpreters are pretty much separate
environments).  We're also dropping the GIL immediately after intializing and
acquiring it, but remembering the main thread's python thread state object,
and passing that to the MgrPyModule constructor too.


Sub-interpreter Creation Works
```c++
MgrPyModule::MgrPyModule(/* ... */) : module_name(mod_name_),
  pMainThreadState(main_ts_), pMyThreadState(nullptr)
{
  // Gil instance instead of PyGILState_Ensure
  Gil gil(pMainThreadState);

  // Save new interprepter thread state as instance variable
  pMyThreadState = Py_NewInterpreter();
  // (...error checks elided again...)

  Py_InitModule("ceph_state", CephStateMethods);
  PySys_SetPath((char*)(sys_path.c_str()));
  // No need for PyGILState_Release; Gil::~Gil() called
}
```


Sub-interpreter Destruction _Almost_ Works
```c++
MgrPyModule::~MgrPyModule() {
  if (pMyThreadState != nullptr) {
    Gil gil(pMyThreadState);

    Py_XDECREF(pClassInstance);

    Py_EndInterpreter(pMyThreadState);
    // Thread state is now NULL

    // Need to swap the main thread state back in
    // so Gil::~Gil() can release the GIL
    PyThreadState_Swap(pMainThreadState);
  }
}
```

Note: you'd think that Py_EndInterpreter would make perfect sense here,
except you might have loaded a python module that itself started up some
separate python thread that we have NFI idea about.


### Python can Create Threads

Note:
This can happen when using CherryPy in a module, becuase CherryPy
runs an extra thread as a timeout monitor, which spends most of its
life inside a time.sleep(60).  Unless you are very, very lucky with
the timing calling this destructor, that thread will still be stuck
in a sleep, and Py_EndInterpreter() will abort.

This could of course also happen with a poorly written module which
made no attempt to clean up any additional threads it created.


_Maybe_ terminate the sub-interpreter?
```c++
MgrPyModule::~MgrPyModule() {
  // ...we could detect extra python threads and *not*
  // end the subinterpreter if there are any...
  if (pMyThreadState !=
        PyInterpreterState_ThreadHead(pMyThreadState->interp)
      || PyThreadState_Next(pMyThreadState) != nullptr) {

    // log "module still has active threads" warning here

  } else {
    Py_EndInterpreter(pMyThreadState);
    PyThreadState_Swap(pMainThreadState);
  }
}
```

Note: This python thread detection method was figured out by reading the
source for Py_EndInterperter...


Eh, screw it...
```c++
MgrPyModule::~MgrPyModule() {
  if (pMyThreadState != nullptr) {
    Gil gil(pMyThreadState);

    Py_XDECREF(pClassInstance);

    // The safest thing to do is just not call
    // Py_EndInterpreter(), and let Py_Finalize() kill
    // everything after *all* modules are shut down.
  }
}
```


## OK, Cool

Note: So, all the pieces are in the right place.  I think.


#### \*\* Caught signal (Segmentation fault) \*\*


> Dammit, I just tried using mgr built with this changeset to load the restful module from #14457. It loads fine, but segfaults on shutdown...
&mdash; Tim Serong


> This is a tough one to review because the subinterpreter functionality in python itself is so obscure. It would be nice to spin the Gil class out into a separate commit.
&mdash; John Spray


> Splitting out the Gil change was a really good idea. Turns out that change by itself exhibited the segfault on shutdown, even without the sub-interpreters, and now I've learned even more than I ever wanted to know about embedding Python :-)
&mdash; Tim Serong

Note: There's a lesson here -- one commit per logical change, every time.


### Every OS Thread
### Must Have at Least One
### Python Thread State

Note: the PYGILState_Ensure functions would create a new python thread state
automatically if one didn't exist for the current thread, but we're not using
those anymore, and each module we're loading has a serve method, which runs in
a separate OS thread.


#### (This is where things get ugly)


This would be ideal, but violates a black box
```c++
Gil::Gil(PyThreadState *ts) : pThreadState(ts)
{
  // Acquire the GIL, set the current thread state
  PyEval_RestoreThread(pThreadState);

  if (pThreadState->thread_id !=
      PyThread_get_thread_ident()) {
    // Create and switch to new python thread state
    // if current thread state doesn't match OS thread
    pNewThreadState =
      PyThreadState_New(pThreadState->interp);
    PyThreadState_Swap(pNewThreadState);
  }
}
```

Note: However, this means we're accessing pThreadState->thread_id, but
the Python C API docs say that "The only public data member is
PyInterpreterState *interp", i.e. doing this would violate
something that's meant to be a black box.


_\*sigh\*_
```c++
Gil::Gil(PyThreadState *ts, bool new_thread = false) :
  pThreadState(ts)
{
  // Acquire the GIL, set the current thread state
  PyEval_RestoreThread(pThreadState);

  if (new_thread) {
    // If called from a separate OS thread, manually
    // create and switch to a new python thread state
    pNewThreadState =
      PyThreadState_New(pThreadState->interp);
    PyThreadState_Swap(pNewThreadState);
  }
}
```

Note: onus is now on the caller.  John has since added a SafeThreadState
class which carrys a record of which POSIX thread the python thread state
relates to, and it'll assert if you call Gil without passing new_thread
when you really should.  Now that we know that works, this could be
refactored further and have Gil figure out whether or not it needs to
create a new python thread state.  So, further improvement.


Some care required
```c++
int MgrPyModule::serve() {
  // This method is called from a thread not created by
  // Python, so tell Gil to create a new python thread state
  Gil gil(pMyThreadState, true);

  auto pValue = PyObject_CallMethod(pClassInstance,
      const_cast<char*>("serve"), nullptr);

  if (pValue == NULL) return -EINVAL;   // log error here

  Py_XDECREF(pValue);

  return 0;
}
```


Just for the record...
```c++
Gil::~Gil()
{
  if (pNewThreadState != nullptr) {
    // Destroy new thread state
    PyThreadState_Swap(pThreadState);
    PyThreadState_Clear(pNewThreadState);
    PyThreadState_Delete(pNewThreadState);
  }
  // Release the GIL, reset the thread state to NULL
  PyEval_SaveThread();
};
```


## All Good?

Note: So now we're all good.  My PR for the sub-interpreters and Gil class
was actually merged at this point.


## Not Quite

Note: if one of our cherrypy modules loads, and something's already listening
on that port, the entire ceph-mgr daemon process exits.


```nohighlight
mgr[py] ENGINE Error in 'start' listener <bound method Server
Traceback (most recent call last):
  File ".../cherrypy/process/wspbus.py", line 205, in publish
    output.append(listener(*args, **kwargs))
  File ".../cherrypy/_cpserver.py", line 168, in start
    ServerAdapter.start(self)
  File ".../cherrypy/process/servers.py", line 170, in start
    wait_for_free_port(*self.bind_addr)
  File ".../cherrypy/process/servers.py", line 438, in wait_
    raise IOError("Port %r not free on %r" % (port, host))

IOError: Port 7000 not free on '127.0.0.1'

mgr[py] ENGINE Shutting down due to error in start listener
```

Note: That's the backtrace, let's go find out WTF cherrypy is doing


/usr/lib/python2.7/site-packages/cherrypy/process/servers.py
```py
def wait_for_free_port(host, port, timeout=None):
    """Wait for the specified port to become free."""
    for trial in range(50):
        try:
            check_port(host, port, timeout=timeout)
        except IOError:
            time.sleep(timeout)
        else:
            return

    raise IOError("Port %r not free on %r" % (port, host))
```


/usr/lib/python2.7/site-packages/cherrypy/process/wspbus.py
```py
class Bus(object):
    def start(self):
        """Start all services."""

        self.state = states.STARTING
        try:
            self.publish('start')
            self.state = states.STARTED
        except:
            self.log("Shutting down due to error [...]",
                     level=40, traceback=True)
            self.exit()
```


/usr/lib/python2.7/site-packages/cherrypy/process/wspbus.py
```py
class Bus(object):
    def exit(self):
        # ...

        if exitstate == states.STARTING:
            # exit() was called before start() finished,
            # possibly due to Ctrl-C because a start
            # listener got stuck. In this case, we could
            # get stuck in a loop where Ctrl-C never exits
            # the process, so we just call os.exit here.
            os._exit(70)  # EX_SOFTWARE
```

Note: Oh, lovely.  That's really not very polite.


Least Worst Soultion
```py
# cherrypy likes to sys.exit on error.
# don't let it take us down too!
def os_exit_noop():
    pass

os._exit = os_exit_noop
```


## Done

Note: now we have subinterpreters working, multiple threads, loading
modules which don't stomp on each other, and it's all nice, but you
do still have to take some care when writing modules because any one
can still kill the whole mgr daemon.

I should note that ceph-mgr only loads the modules the admin has told
it to; it won't just load everyhing automatically, so if there ever
were a misbehaving python module it could be disabled until someone
could debug and fix it.  So anyone else attempting the techniques I've
outlined here should probably do something similar.


## Oh, Wait

Note: There is just one more thing...


## Python 3

Note: When I was writing the slides for this talk, I actually tried to
compile some of the example code.  It didn't work.


`char *` becomes `wchar_t *`


`PyString_FromString()`

becomes

`PyUnicode_FromString()`


`Py_InitModule()`

becomes

`PyModule_Create()` and `PyImport_AppendInittab()`

