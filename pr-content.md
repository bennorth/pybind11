Hi,

I recently hit segfaults on Python exit while using a pybind11-based module.  I have found and fixed a bug in it, but while investigating, found what I think are bugs in pybind11 too.  I think the code in `make_static_property_type()` needs to `inc_ref()` `PyProperty_Type` because of its use as the `tp_base` of the new type; similarly for other assignments to a new type's `tp_base`.  In my case, enough refs were being destroyed that on exit, Python tried to dealloc `PyProperty_Type`, with bad results.

In pure Python, creating a heap-subtype of 'property' increases the refcount of `property` (i.e., `PyProperty_Type`) by 3:
```bash
python -c "
import sys
print('before:', sys.getrefcount(property))
st = type('st', (property,), {})
print('after: ', sys.getrefcount(property))"
```
```
before: 23
after:  26
```
This makes sense; the three new references in `st` are: the `__base__`; the sole entry in the `__bases__` tuple; the entry in the `__mro__`.

However, importing a trivial pybind11-based module only increases the refcount by 2 even though importing the module also creates a heap-subtype of property (in `make_static_property_type()`):
```c++
#include <pybind11/pybind11.h>
struct Empty {};
PYBIND11_MODULE(empty_struct, m)
{ pybind11::class_<Empty>(m, "Empty"); }
```
```bash
# ... compile into 'empty_struct' module ...

python -c "
import sys
print('before:', sys.getrefcount(property))
import empty_struct
print('after: ', sys.getrefcount(property))"
```
```
before: 23
after:  25
```
While reading around, I found [Python issue 980082](http://bugs.python.org/issue980082) which suggests a new type's `tp_base` ought to be `inc_ref()`'d.  Also, pybind11's `make_new_python_type()` does `inc_ref()` the `tp_base` of the new type.

This PR `inc_ref()`s the other three assignments to a new type's `tp_base`.  This seemed just about worth making a tiny function for.

Thanks!
