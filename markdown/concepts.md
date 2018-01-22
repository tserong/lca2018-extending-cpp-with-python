Minimal Python invocation from C++

```c++
#include <Python.h>

int main(int argc, char ** argv)
{
    Py_SetProgramName(argv[0]);
    Py_Initialize();

    PyRun_SimpleString("print('Look! Python!')\n");

    // or, try: PyRun_SimpleFile(fp, filename);

    Py_Finalize();
    return 0;
}
```

Note: Time for some code.  If you want to invoke Python from C++, the least you
need to do is this.  Py_SetProgramName() allegedly informs the interpreter about paths to Python run-time libraries.  Py_Initialize() initializes the interpreter.  PyRun_SimpleString() (or File) runs some code, and Py_Finalize() shuts the interpreter down.

That's fine as far as it goes, but we wanted to be able to a bit more than that.  As I mentioned, we wanted to be able to load python modules, and call into them from C++, and we want those python modules to be able to in turn call functions in our C++ code.


Calling Python module functions from C++
```c++
    auto pName = PyString_FromString("my_module");
    auto pModule = PyImport_Import(pName);
    Py_DECREF(pName);
    // error check for pModule == nullptr here

    auto pFunc = PyObject_GetAttrString(pModule, "my_func");
    if (pFunc != nullptr) {
        auto pValue = PyObject_CallObject(pFunc, nullptr);
        // inspect pValue if you care
        Py_XDECREF(pValue);
    }

    Py_XDECREF(pFunc);
    Py_DECREF(pModule);
```

Note: To load a module and call a function within it, you'd do something like this (after doing the Py_Initialize() business).

Note how I'm suddenly introducing reference counting, without bothering to have explained it yet.  Python refcounts everything -- if you create some python object, or get a value back from a python function, you need to decref it once you're done with it.  If you're not sure if it's NULL, you can XDECREF it.


Exposing C++ functions to Python
```c++
// C++ function to be exposed to python
static PyObject * answer(PyObject * self, PyObject * args) {
    return PyLong_FromLong(42);
}

// table of functions to expose
static PyMethodDef UQMethods[] = {
    { "answer", answer, METH_VARARGS,
      "Return the answer to the ultimate question" },
    { nullptr, nullptr, 0, nullptr }
};

// Later, in main()...
Py_InitModule("ultimate_question", UQMethods);
```

Note: To do the converse, have some C++ code that's callable from Python, you'd do something like this.


Calling C++ functions from Python
```py
import ultimate_question

print(ultimate_question.answer())
```

Note: and that's how you call back into it from Python.  Looks just like Python.
