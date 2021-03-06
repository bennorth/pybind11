Functions
#########

Before proceeding with this section, make sure that you are already familiar
with the basics of binding functions and classes, as explained in :doc:`/basics`
and :doc:`/classes`. The following guide is applicable to both free and member
functions, i.e. *methods* in Python.

Return value policies
=====================

Python and C++ use wildly different ways of managing the memory and lifetime of
objects managed by them. This can lead to issues when creating bindings for
functions that return a non-trivial type. Just by looking at the type
information, it is not clear whether Python should take charge of the returned
value and eventually free its resources, or if this is handled on the C++ side.
For this reason, pybind11 provides a several `return value policy` annotations
that can be passed to the :func:`module::def` and :func:`class_::def`
functions. The default policy is :enum:`return_value_policy::automatic`.

Return value policies can also be applied to properties, in which case the
arguments must be passed through the :class:`cpp_function` constructor:

.. code-block:: cpp

    class_<MyClass>(m, "MyClass")
        def_property("data"
            py::cpp_function(&MyClass::getData, py::return_value_policy::copy),
            py::cpp_function(&MyClass::setData)
        );

The following table provides an overview of the available return value policies:

.. tabularcolumns:: |p{0.5\textwidth}|p{0.45\textwidth}|

+--------------------------------------------------+----------------------------------------------------------------------------+
| Return value policy                              | Description                                                                |
+==================================================+============================================================================+
| :enum:`return_value_policy::automatic`           | This is the default return value policy, which falls back to the policy    |
|                                                  | :enum:`return_value_policy::take_ownership` when the return value is a     |
|                                                  | pointer. Otherwise, it uses :enum:`return_value::move` or                  |
|                                                  | :enum:`return_value::copy` for rvalue and lvalue references, respectively. |
|                                                  | See below for a description of what all of these different policies do.    |
+--------------------------------------------------+----------------------------------------------------------------------------+
| :enum:`return_value_policy::automatic_reference` | As above, but use policy :enum:`return_value_policy::reference` when the   |
|                                                  | return value is a pointer. This is the default conversion policy for       |
|                                                  | function arguments when calling Python functions manually from C++ code    |
|                                                  | (i.e. via handle::operator()). You probably won't need to use this.        |
+--------------------------------------------------+----------------------------------------------------------------------------+
| :enum:`return_value_policy::take_ownership`      | Reference an existing object (i.e. do not create a new copy) and take      |
|                                                  | ownership. Python will call the destructor and delete operator when the    |
|                                                  | object's reference count reaches zero. Undefined behavior ensues when the  |
|                                                  | C++ side does the same.                                                    |
+--------------------------------------------------+----------------------------------------------------------------------------+
| :enum:`return_value_policy::copy`                | Create a new copy of the returned object, which will be owned by Python.   |
|                                                  | This policy is comparably safe because the lifetimes of the two instances  |
|                                                  | are decoupled.                                                             |
+--------------------------------------------------+----------------------------------------------------------------------------+
| :enum:`return_value_policy::move`                | Use ``std::move`` to move the return value contents into a new instance    |
|                                                  | that will be owned by Python. This policy is comparably safe because the   |
|                                                  | lifetimes of the two instances (move source and destination) are decoupled.|
+--------------------------------------------------+----------------------------------------------------------------------------+
| :enum:`return_value_policy::reference`           | Reference an existing object, but do not take ownership. The C++ side is   |
|                                                  | responsible for managing the object's lifetime and deallocating it when    |
|                                                  | it is no longer used. Warning: undefined behavior will ensue when the C++  |
|                                                  | side deletes an object that is still referenced and used by Python.        |
+--------------------------------------------------+----------------------------------------------------------------------------+
| :enum:`return_value_policy::reference_internal`  | Indicates that the lifetime of the return value is tied to the lifetime    |
|                                                  | of a parent object, namely the implicit ``this``, or ``self`` argument of  |
|                                                  | the called method or property. Internally, this policy works just like     |
|                                                  | :enum:`return_value_policy::reference` but additionally applies a          |
|                                                  | ``keep_alive<0, 1>`` *call policy* (described in the next section) that    |
|                                                  | prevents the parent object from being garbage collected as long as the     |
|                                                  | return value is referenced by Python. This is the default policy for       |
|                                                  | property getters created via ``def_property``, ``def_readwrite``, etc.     |
+--------------------------------------------------+----------------------------------------------------------------------------+

.. warning::

    Code with invalid return value policies might access unitialized memory or
    free data structures multiple times, which can lead to hard-to-debug
    non-determinism and segmentation faults, hence it is worth spending the
    time to understand all the different options in the table above.

One important aspect of the above policies is that they only apply to instances
which pybind11 has *not* seen before, in which case the policy clarifies
essential questions about the return value's lifetime and ownership.  When
pybind11 knows the instance already (as identified by its type and address in
memory), it will return the existing Python object wrapper rather than creating
a new copy.

.. note::

    The next section on :ref:`call_policies` discusses *call policies* that can be
    specified *in addition* to a return value policy from the list above. Call
    policies indicate reference relationships that can involve both return values
    and parameters of functions.

.. note::

   As an alternative to elaborate call policies and lifetime management logic,
   consider using smart pointers (see the section on :ref:`smart_pointers` for
   details). Smart pointers can tell whether an object is still referenced from
   C++ or Python, which generally eliminates the kinds of inconsistencies that
   can lead to crashes or undefined behavior. For functions returning smart
   pointers, it is not necessary to specify a return value policy.

.. _call_policies:

Additional call policies
========================

In addition to the above return value policies, further `call policies` can be
specified to indicate dependencies between parameters. There is currently just
one policy named ``keep_alive<Nurse, Patient>``, which indicates that the
argument with index ``Patient`` should be kept alive at least until the
argument with index ``Nurse`` is freed by the garbage collector. Argument
indices start at one, while zero refers to the return value. For methods, index
``1`` refers to the implicit ``this`` pointer, while regular arguments begin at
index ``2``. Arbitrarily many call policies can be specified. When a ``Nurse``
with value ``None`` is detected at runtime, the call policy does nothing.

This feature internally relies on the ability to create a *weak reference* to
the nurse object, which is permitted by all classes exposed via pybind11. When
the nurse object does not support weak references, an exception will be thrown.

Consider the following example: here, the binding code for a list append
operation ties the lifetime of the newly added element to the underlying
container:

.. code-block:: cpp

    py::class_<List>(m, "List")
        .def("append", &List::append, py::keep_alive<1, 2>());

.. note::

    ``keep_alive`` is analogous to the ``with_custodian_and_ward`` (if Nurse,
    Patient != 0) and ``with_custodian_and_ward_postcall`` (if Nurse/Patient ==
    0) policies from Boost.Python.

.. seealso::

    The file :file:`tests/test_keep_alive.cpp` contains a complete example
    that demonstrates using :class:`keep_alive` in more detail.

.. _python_objects_as_args:

Python objects as arguments
===========================

pybind11 exposes all major Python types using thin C++ wrapper classes. These
wrapper classes can also be used as parameters of functions in bindings, which
makes it possible to directly work with native Python types on the C++ side.
For instance, the following statement iterates over a Python ``dict``:

.. code-block:: cpp

    void print_dict(py::dict dict) {
        /* Easily interact with Python types */
        for (auto item : dict)
            std::cout << "key=" << item.first << ", "
                      << "value=" << item.second << std::endl;
    }

It can be exported:

.. code-block:: cpp

    m.def("print_dict", &print_dict);

And used in Python as usual:

.. code-block:: pycon

    >>> print_dict({'foo': 123, 'bar': 'hello'})
    key=foo, value=123
    key=bar, value=hello

For more information on using Python objects in C++, see :doc:`/advanced/pycpp/index`.

Accepting \*args and \*\*kwargs
===============================

Python provides a useful mechanism to define functions that accept arbitrary
numbers of arguments and keyword arguments:

.. code-block:: python

   def generic(*args, **kwargs):
       ...  # do something with args and kwargs

Such functions can also be created using pybind11:

.. code-block:: cpp

   void generic(py::args args, py::kwargs kwargs) {
       /// .. do something with args
       if (kwargs)
           /// .. do something with kwargs
   }

   /// Binding code
   m.def("generic", &generic);

The class ``py::args`` derives from ``py::tuple`` and ``py::kwargs`` derives
from ``py::dict``. Note that the ``kwargs`` argument is invalid if no keyword
arguments were actually provided. Please refer to the other examples for
details on how to iterate over these, and on how to cast their entries into
C++ objects. A demonstration is also available in
``tests/test_kwargs_and_defaults.cpp``.

.. warning::

   Unlike Python, pybind11 does not allow combining normal parameters with the
   ``args`` / ``kwargs`` special parameters.

Default arguments revisited
===========================

The section on :ref:`default_args` previously discussed basic usage of default
arguments using pybind11. One noteworthy aspect of their implementation is that
default arguments are converted to Python objects right at declaration time.
Consider the following example:

.. code-block:: cpp

    py::class_<MyClass>("MyClass")
        .def("myFunction", py::arg("arg") = SomeType(123));

In this case, pybind11 must already be set up to deal with values of the type
``SomeType`` (via a prior instantiation of ``py::class_<SomeType>``), or an
exception will be thrown.

Another aspect worth highlighting is that the "preview" of the default argument
in the function signature is generated using the object's ``__repr__`` method.
If not available, the signature may not be very helpful, e.g.:

.. code-block:: pycon

    FUNCTIONS
    ...
    |  myFunction(...)
    |      Signature : (MyClass, arg : SomeType = <SomeType object at 0x101b7b080>) -> NoneType
    ...

The first way of addressing this is by defining ``SomeType.__repr__``.
Alternatively, it is possible to specify the human-readable preview of the
default argument manually using the ``arg_v`` notation:

.. code-block:: cpp

    py::class_<MyClass>("MyClass")
        .def("myFunction", py::arg_v("arg", SomeType(123), "SomeType(123)"));

Sometimes it may be necessary to pass a null pointer value as a default
argument. In this case, remember to cast it to the underlying type in question,
like so:

.. code-block:: cpp

    py::class_<MyClass>("MyClass")
        .def("myFunction", py::arg("arg") = (SomeType *) nullptr);
