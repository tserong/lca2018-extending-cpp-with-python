## ceph-mgr

Note: Back to Ceph.  The new ceph-mgr daemon is what's responsible for loading
our python plugin modules.  John Spray did the initial implementation which used
essentially the techniques we just went through, but it's slightly more complicated.
ceph-mgr loads whichever plugin python modules you ask it to, there can be many,
and because we want to be able to implement long-running server type things,
each of these are loaded in a separate thread.  It also exposes a couple of
module-like-things from C++ code, a logger to capture stdout and stderr, and one
called ceph_state with functions for getting information from the cluster, getting
and setting config options, sending commands to the cluster and so forth.

Python modules loaded by ceph-mgr are required to adhere to a certain simple API.


The Python side
```py
class Module(MgrModule):
    COMMANDS = [ { "cmd": "activate", "..." }, ... ]

    def handle_command(self, cmd):
        self.log.info("handle_command: " + cmd)

    def serve(self):
        while self.run: time.sleep(1)

    def shutdown(self):
        self.run = False

    def notify(self, notify_type, notify_id):
        self.log.info("OMG! Something happened!")
```

Note: on the python side, there's a class called MgrModule which you inherit from,
it sets up a logger and includes a handful of helper methods, but the interesting
part is the bits you override to make a module that does something.

Initially there were two modules included with ceph-mgr, one to give you filesystem
statistics from the command line, and one which provided a REST API that other
applications could use to talk to the ceph cluster.  Two examples of the two
different modes.


The C++ side
```c++
class MgrPyModule {
private:
  PyObject *pClassInstance;

public:
  MgrPyModule(const std::string &module_name);

  int load();

  int handle_command(const cmdmap_t &cmdmap, /* ... */);
  int serve();
  void shutdown();
  void notify(const std::string &notify_type, /* ... */);
}
```

Note: On the C++ side, there's an instance of a C++ class for each loaded python
plugin module, with a suspiciously similar structure.  ceph-mgr keeps a list of
these, so it can, say, call all their notify methods when something interesting
happens.  Importantly, each loaded plugin module runs in its own thread; as I
mentioend before, this is what enables us to load python modules that expose
long-running sevices via the serve method.


Instantiating a Python class
```c++
int MgrPyModule::load() {
  auto pName = PyString_FromString(module_name.c_str());
  auto pModule = PyImport_Import(pName);

  // It's easy if every plugin class is named "Module"
  auto pClass = PyObject_GetAttrString(pModule, "Module");

  auto pArgs = PyTuple_Pack(1, pName);
  // Calling a class gives you an instance of it
  pClassInstance = PyObject_CallObject(pClass, pArgs);

  // Lots of Py_DECREF()ing goes here...
  return 0;
}
```

Note: This is basically the same as the earlier example of calling a python
funciton, it's just that if you call a python class, it returns an instance
of it.  The extra argument to CallObject is the handle that MgrModule keeps to
use when logging (the module name)


Calling a Python method
```c++
int MgrPyModule::serve() {
  PyGILState_STATE gstate = PyGILState_Ensure();

  auto pValue = PyObject_CallMethod(pClassInstance,
      const_cast<char*>("serve"), nullptr);

  if (pValue == NULL) return -EINVAL;   // log error here

  Py_XDECREF(pValue);

  PyGILState_Release(gstate);
  return 0;
}
```

Note: Once again, I've suddendly introduced something new.  Remember how I said
this thing is multithreaded?  See that PyGILState_Ensure and Release business?
That means we need to talk about...


## The GIL

Note: The Python interpreter isn't fully thread-safe, so there's a global interpreter lock which needs to be held by the current thread before it can safely access python objects.


```c++
int PyModules::init() {
  // ...
  PySys_SetPath(our_custom_module_path);
  PyEval_InitThreads();         // Init and acquire GIL

  for (/* each module_name */) {
    auto mod = std::unique_ptr<MgrPyModule>(
               new MgrPyModule(module_name));
    mod->load();
  }

  PyEval_SaveThread();          // Drop GIL
  return 0;
}
```

Note: PyEval_InitThreads() initializes and acquires the GIL, PyEval_SaveThread() drops it, and as we already saw, any C++ code than needs to call into python or manipulate python objects needs to call PyGILState_Ensure() first and PyGILState_Release() last


### Another Small Detail

Note: The stdout and stderr capture is kinda cute


Walks like a file, quacks like a file...
```c++
PyObject* log_write(PyObject*, PyObject* args) {
    char* m = nullptr;
    if (PyArg_ParseTuple(args, "s", &m)) {
        // m is written to ceph-mgr's log file here
    }
    Py_RETURN_NONE;
}

PyObject* log_flush(PyObject*, PyObject*){
    Py_RETURN_NONE;
}
```

Note: We've got write and flush, which is just enough methods to look like a python file object.  So, then we can replace sys.stderr and sys.stdout with our C++ functions, like so:


All your stdout, stderr are belong to us
```c++
static PyMethodDef log_methods[] = {
  { "write", log_write, METH_VARARGS, "write stdout/err" },
  { "flush", log_flush, METH_VARARGS, "flush" },
  { nullptr, nullptr, 0, nullptr }
};

int PyModules::init() {
  // ...
  auto py_logger = Py_InitModule("ceph_logger", log_methods);
  PySys_SetObject("stderr", py_logger);
  PySys_SetObject("stdout", py_logger);
  // ...
}
```


## Easy, Right?
