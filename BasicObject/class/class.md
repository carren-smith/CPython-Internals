# class

### category

* [related file](#related-file)
* [memory layout](#memory-layout)
* [fields](#fields)
	* [im_func](#im_func)
	* [im_self](#im_self)
* [free_list](#free_list)

#### related file
* cpython/Objects/classobject.c
* cpython/Include/classobject.h

#### memory layout

the **PyMethodObject** represents the type **method** in c-level

    class C(object):
        def f1(self, val):
            return val

    >>> c = C()
    >>> type(c.f1)
    method

![layout](https://github.com/zpoint/CPython-Internals/blob/master/BasicObject/class/layout.png)

#### fields

the layout of **c.f**

![example0](https://github.com/zpoint/CPython-Internals/blob/master/BasicObject/class/example0.png)

##### im_func

as you can see from the layout, field **im_func** stores the [function](https://github.com/zpoint/CPython-Internals/blob/master/BasicObject/func/func.md) object that implementing the method

    >>> C.f1
    <function C.f1 at 0x10b80f040>

##### im_self

field **im_self** stores the instance object this method bound to

    >>> c
    <__main__.C object at 0x10b7cbcd0>

when you call

	>>> c.f1(123)
	123

the **PyMethodObject** delegate the real call to **im_func** with **im_self** as the first argument

    static PyObject *
    method_call(PyObject *method, PyObject *args, PyObject *kwargs)
    {
        PyObject *self, *func;
		/* get im_self */
        self = PyMethod_GET_SELF(method);
        if (self == NULL) {
            PyErr_BadInternalCall();
            return NULL;
        }
		/* get im_func */
        func = PyMethod_GET_FUNCTION(method);
		/* call im_func with im_self as the first argument */
        return _PyObject_Call_Prepend(func, self, args, kwargs);
    }

#### free_list

    static PyMethodObject *free_list;
    static int numfree = 0;
    #ifndef PyMethod_MAXFREELIST
    #define PyMethod_MAXFREELIST 256
    #endif

free_list is a single linked list, it's used for **PyMethodObject** to safe malloc/free overhead

**im_self** field is used to chain the element

the **PyMethodObject** will be created when you trying to access the bound-method, not when the instance is created

    >>> c1 = C()
    >>> id(c1)
    4514815184
    >>> c2 = C()
    >>> id(c2)
    4514815472
    >>> id(c1.f1) # c1.f1 is created in this line, after this line, the reference count of c1.f1 becomes 0 and c1.f1 deallocated
    4513259240
    >>> id(c1.f1) # the id is resued
    4513259240
    >>> id(c2.f1)
    4513259240

now, let's see an example of free_list

	>>> c1_f1_1 = c1.f1
	>>> c1_f1_2 = c1.f1
    >>> id(c1_f1_1)
    4529762024
    >>> id(c1_f1_2)
    4529849392

assume the free_list is empty now

![free_list0](https://github.com/zpoint/CPython-Internals/blob/master/BasicObject/class/free_list0.png)

    >>> del c1_f1_1
    >>> del c1_f1_2

![free_list1](https://github.com/zpoint/CPython-Internals/blob/master/BasicObject/class/free_list1.png)

    >>> c1_f1_3 = c1.f1
    >>> id(c1_f1_3)
    4529849392

![free_list2](https://github.com/zpoint/CPython-Internals/blob/master/BasicObject/class/free_list2.png)