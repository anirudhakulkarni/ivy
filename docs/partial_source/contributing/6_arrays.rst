Arrays
======

.. _`empty class`: https://github.com/unifyai/ivy/blob/529c8c0f128ff28331da7c8f52912d777d786cbe/ivy/__init__.py#L8
.. _`overwritten`: https://github.com/unifyai/ivy/blob/529c8c0f128ff28331da7c8f52912d777d786cbe/ivy/functional/backends/torch/__init__.py#L11
.. _`self._data`: https://github.com/unifyai/ivy/blob/529c8c0f128ff28331da7c8f52912d777d786cbe/ivy/array/__init__.py#L89
.. _`ArrayWithElementwise`: https://github.com/unifyai/ivy/blob/529c8c0f128ff28331da7c8f52912d777d786cbe/ivy/array/elementwise.py#L12
.. _`ivy.Array.add`: https://github.com/unifyai/ivy/blob/529c8c0f128ff28331da7c8f52912d777d786cbe/ivy/array/elementwise.py#L22
.. _`programmatically`: https://github.com/unifyai/ivy/blob/529c8c0f128ff28331da7c8f52912d777d786cbe/ivy/__init__.py#L148
.. _`backend type hints`: https://github.com/unifyai/ivy/blob/8605c0a50171bb4818d0fb3e426cec874de46baa/ivy/functional/backends/torch/elementwise.py#L219
.. _`Ivy type hints`: https://github.com/unifyai/ivy/blob/8605c0a50171bb4818d0fb3e426cec874de46baa/ivy/functional/ivy/elementwise.py#L1342
.. _`__setitem__`: https://github.com/unifyai/ivy/blob/8605c0a50171bb4818d0fb3e426cec874de46baa/ivy/array/__init__.py#L234
.. _`function wrapping`: https://github.com/unifyai/ivy/blob/0f131178be50ea08ec818c73078e6e4c88948ab3/ivy/func_wrapper.py#L138

There are two types of array in Ivy, there is the :code:`ivy.NativeArray` and also the :code:`ivy.Array`.

Native Array
------------

The :code:`ivy.NativeArray` is simply a placeholder class for a backend-specific array class,
such as :code:`np.ndarray`, :code:`tf.Tensor` and :code:`torch.Tensor`

When no framework is set, this is an `empty class`_.
When a framework is set, this is `overwritten`_ with the backend-specific array class.

Ivy Array
---------

The :code:`ivy.Array` is a simple wrapper class, which wraps around the :code:`ivy.NativeArray`,
storing it in `self._data`_.

All functions in the Ivy functional API which accept *at least one array argument* in the input are implemented
as instance methods in the :code:`ivy.Array` class.

The organization of these instance methods follows the same organizational structure as the
files in the functional API.
The :code:`ivy.Array` class inherits from many category-specific array classes, such as `ArrayWithElementwise`_,
each of which implement the category-specific instance methods.

Each instance method simply calls the functional API function internally,
but passes in :code:`self` as the first array argument. `ivy.Array.add`_ is a good example.

Given the simple set of rules which underpin how these instance methods should all be implemented,
if a source-code implementation is not found, then this instance method is added `programmatically`_.
This serves as a helpful backup in cases where some functions are accidentally missed out.

The benefit of the source code implementations is that this makes the code much more readable,
without important methods being entirely absent.
It also enables other helpful perks, such as auto-completions in the IDE etc.

Array Handling
--------------

When calling wrapped backend methods, we must pass in :code:`ivy.NativeArray` instances.
For example, :code:`torch.sin` will throw an error if we try to pass in an :code:`ivy.Array` instance.
It must be provided with a :code:`torch.Tensor`, and this is reflected in the `backend type hints`_.

However, all Ivy functions must return :code:`ivy.Array` instances, which is reflected in the `Ivy type hints`_.
The reason we always return :code:`ivy.Array` instances from Ivy functions is to ensure that any subsequent Ivy code is
fully framework-agnostic, with all operators performed on the returned array now handled by the special methods of the
:code:`ivy.Array` class, and not the special methods of the backend array class (:code:`ivy.NativeArray`).

For example, calling any of (:code:`+`, :code:`-`, :code:`*`, :code:`/` etc.) on the array will result in
(:code:`__add__`, :code:`__sub__`, :code:`__mul__`, :code:`__div__` etc.) being called on the array class.

For most special methods, this is not be a problem and each backend is generally quite consistent,
but for some functions such as `__setitem__`_,
there are substantial differences which must be addressed in the :code:`ivy.Array` implementation.

Given that all Ivy functions return :code:`ivy.Array` instances,
all Ivy functions must also support :code:`ivy.Array` instances in the input,
otherwise it would be impossible to chain functions together!

Therefore, every function must adopt the following pipeline:

#. convert all :code:`ivy.Array` instances in the input arguments to :code:`ivy.NativeArray` instances
#. call the backend-specific function, passing in these :code:`ivy.NativeArray` instances
#. convert all of the :code:`ivy.NativeArray` instances which are returned from the backend function back into \
   :code:`ivy.Array` instances
#. return these

Given the repeating nature of this piece of logic, this is all entirely handling in the `function wrapping`_,
as explained in the :ref:`Function Wrapping` section.

All Ivy functions also accept :code:`ivy.NativeArray` instances in the input.
Firstly, :code:`ivy.Array` instances must be converted to :code:`ivy.NativeArray` instances anyway,
and so supporting them in the input is not a problem.
Secondly, this makes it easier to combine backend-specific code with Ivy code,
without needing to explicitly wrap any arrays before calling sections of Ivy code.

Therefore, all input arrays to Ivy functions have type :code:`Union[ivy.Array, ivy.NativeArray]`,
whereas the output arrays have type :code:`ivy.Array`. This is further explained in the :ref:`Type Hints` section.

However, :code:`ivy.NativeArray` instances are not permitted for the :code:`out` argument,
which is used in most functions.
This is because the :code:`out` argument dicates the array to which the result should be written, and so it effectively
serves the same purpose as the function return when no :code:`out` argument is specified.
This is further explained in the :ref:`Inplace Updates` section.

As a final point, extra attention is required for *compositional* functions,
as these do not directly defer to a backend implementation.
If the first line of code in a compositional function performs operations on the input array,
then this would be a problem as they will be performed on the :code:`ivy.NativeArray` instance
and not on an :code:`ivy.Array`.

Therefore, all compositional functions have a seperate piece of wrapped logic to ensure that all :code:`ivy.NativeArray`
instances are converted to :code:`ivy.Array` instances before entering into the compositional function.