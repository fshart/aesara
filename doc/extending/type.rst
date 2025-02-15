.. _aesara_type:

===============
:class:`Type`\s
===============


.. _type_contract:

:class:`Type`'s contract
========================

In Aesara's framework, a :class:`Type` is any object which defines the following
methods. To obtain the default methods described below, the :class:`Type` should be an
instance of `Type` or should be an instance of a subclass of `Type`. If you
will write all methods yourself, you need not use an instance of `Type`.

Methods with default arguments must be defined with the same signature,
i.e.  the same default argument names and values. If you wish to add
extra arguments to any of these methods, these extra arguments must have
default values.

.. class:: Type

    .. method:: filter(value, strict=False, allow_downcast=None)

      This casts a value to match the :class:`Type` and returns the
      cast value. If ``value`` is incompatible with the :class:`Type`,
      the method must raise an exception. If ``strict`` is True, ``filter`` must return a
      reference to ``value`` (i.e. casting prohibited).
      If ``strict`` is False, then casting may happen, but downcasting should
      only be used in two situations:

      * if ``allow_downcast`` is True
      * if ``allow_downcast`` is ``None`` and the default behavior for this
        type allows downcasting for the given ``value`` (this behavior is
        type-dependent, you may decide what your own type does by default)

      We need to define ``filter`` with three arguments. The second argument
      must be called ``strict`` (Aesara often calls it by keyword) and must
      have a default value of ``False``. The third argument must be called
      ``allow_downcast`` and must have a default value of ``None``.

    .. method:: filter_inplace(value, storage, strict=False, allow_downcast=None)

      If filter_inplace is defined, it will be called instead of
      filter() This is to allow reusing the old allocated memory. As
      of this writing this is used only when we transfer new data to a
      shared variable on the gpu.

      ``storage`` will be the old value. i.e. The old numpy array,
      CudaNdarray, ...

    .. method:: is_valid_value(value)

      Returns True iff the value is compatible with the :class:`Type`. If
      ``filter(value, strict = True)`` does not raise an exception, the
      value is compatible with the :class:`Type`.

      *Default:* True iff ``filter(value, strict=True)`` does not raise
      an exception.

    .. method:: values_eq(a, b)

      Returns True iff ``a`` and ``b`` are equal.

      *Default:* ``a == b``

    .. method:: values_eq_approx(a, b)

      Returns True iff ``a`` and ``b`` are approximately equal, for a
      definition of "approximately" which varies from :class:`Type` to :class:`Type`.

      *Default:* ``values_eq(a, b)``

    .. method:: make_variable(name=None)

      Makes a :term:`Variable` of this :class:`Type` with the specified name, if
      ``name`` is not ``None``. If ``name`` is ``None``, then the `Variable` does
      not have a name. The `Variable` will have its ``type`` field set to
      the :class:`Type` object.

      *Default:* there is a generic definition of this in `Type`. The
      `Variable`'s ``type`` will be the object that defines this method (in
      other words, ``self``).

    .. method:: __call__(name=None)

      Syntactic shortcut to ``make_variable``.

      *Default:* ``make_variable``

    .. method:: __eq__(other)

      Used to compare :class:`Type` instances themselves

      *Default:* ``object.__eq__``

    .. method:: __hash__()

      :class:`Type`\s should not be mutable, so it should be OK to define a hash
      function.  Typically this function should hash all of the terms
      involved in ``__eq__``.

      *Default:* ``id(self)``

    .. method:: get_shape_info(obj)

      Optional. Only needed to profile the memory of this :class:`Type` of object.

      Return the information needed to compute the memory size of ``obj``.

      The memory size is only the data, so this excludes the container.
      For an ndarray, this is the data, but not the ndarray object and
      other data structures such as shape and strides.

      ``get_shape_info()`` and ``get_size()`` work in tandem for the memory profiler.

      ``get_shape_info()`` is called during the execution of the function.
      So it is better that it is not too slow.

      ``get_size()`` will be called on the output of this function
      when printing the memory profile.

      :param obj: The object that this :class:`Type` represents during execution
      :return: Python object that ``self.get_size()`` understands


    .. method:: get_size(shape_info)

        Number of bytes taken by the object represented by shape_info.

        Optional. Only needed to profile the memory of this :class:`Type` of object.

        :param shape_info: the output of the call to get_shape_info()
        :return: the number of bytes taken by the object described by
            ``shape_info``.

    .. method:: clone(dtype=None, broadcastable=None)

       Optional, for TensorType-alikes.

       Return a copy of the type with a possibly changed value for
       dtype and broadcastable (if they aren't `None`).

       :param dtype: New dtype for the copy.
       :param broadcastable: New broadcastable tuple for the copy.

    .. method:: may_share_memory(a, b)

        Optional to run, but mandatory for `DebugMode`. Return ``True`` if the Python
        objects `a` and `b` could share memory. Return ``False``
        otherwise. It is used to debug when :class:`Op`\s did not declare memory
        aliasing between variables. Can be a static method.
        It is highly recommended to use and is mandatory for :class:`Type` in Aesara
        as our buildbot runs in `DebugMode`.

For each method, the *default* is what `Type` defines
for you. So, if you create an instance of `Type` or an
instance of a subclass of `Type`, you
must define ``filter``. You might want to override ``values_eq_approx``,
as well as ``values_eq``. The other defaults generally need not be
overridden.

For more details you can go see the documentation for :ref:`type`.


Additional definitions
----------------------

For certain mechanisms, you can register functions and other such
things to plus your type into aesara's mechanisms.  These are optional
but will allow people to use you type with familiar interfaces.

`transfer()`
~~~~~~~~~~~~

To plug in additional options for the transfer target, define a
function which takes an Aesara variable and a target argument and
returns eitehr a new transferred variable (which can be the same as
the input if no transfer is necessary) or returns None if the transfer
can't be done.

Then register that function by calling :func:`register_transfer()`
with it as argument.

An example
==========

We are going to base :class:`Type` ``double`` on Python's ``float``. We
must define ``filter`` and shall override ``values_eq_approx``.


**filter**

.. testcode::

    # Note that we shadow Python's function ``filter`` with this
    # definition.
    def filter(x, strict=False, allow_downcast=None):
        if strict:
            if isinstance(x, float):
                return x
            else:
                raise TypeError('Expected a float!')
        elif allow_downcast:
            return float(x)
        else:   # Covers both the False and None cases.
            x_float = float(x)
            if x_float == x:
                return x_float
            else:
                 raise TypeError('The double type cannot accurately represent '
                                 'value %s (of type %s): you must explicitly '
                                 'allow downcasting if you want to do this.'
                                 % (x, type(x)))

If ``strict`` is True we need to return ``x``. If ``strict`` is True and ``x`` is not a
``float`` (for example, ``x`` could easily be an ``int``) then it is
incompatible with our :class:`Type` and we must raise an exception.

If ``strict is False`` then we are allowed to cast ``x`` to a ``float``,
so if ``x`` is an ``int`` it we will return an equivalent ``float``.
However if this cast triggers a precision loss (``x != float(x)``) and
``allow_downcast`` is not True, then we also raise an exception.
Note that here we decided that the default behavior of our type
(when ``allow_downcast`` is set to ``None``) would be the same as
when ``allow_downcast`` is False, i.e. no precision loss is allowed.


**values_eq_approx**

.. testcode::

   def values_eq_approx(x, y, tolerance=1e-4):
       return abs(x - y) / (abs(x) + abs(y)) < tolerance

The second method we define is ``values_eq_approx``. This method
allows approximate comparison between two values respecting our :class:`Type`'s
constraints. It might happen that an optimization changes the computation
graph in such a way that it produces slightly different variables, for
example because of numerical instability like rounding errors at the
end of the mantissa. For instance, ``a + a + a + a + a + a`` might not
actually produce the exact same output as ``6 * a`` (try with a=0.1),
but with ``values_eq_approx`` we do not necessarily mind.

We added an extra ``tolerance`` argument here. Since this argument is
not part of the API, it must have a default value, which we
chose to be 1e-4.

.. note::

    ``values_eq`` is never actually used by Aesara, but it might be used
    internally in the future. Equality testing in
    :ref:`DebugMode <debugmode>` is done using ``values_eq_approx``.

**Putting them together**

What we want is an object that respects the aforementioned
contract. Recall that :class:`Type` defines default implementations for all
required methods of the interface, except ``filter``. One way to make
the :class:`Type` is to instantiate a plain :class:`Type` and set the needed fields:

.. testcode::

   from aesara.graph.type import Type

   double = Type()
   double.filter = filter
   double.values_eq_approx = values_eq_approx


Another way to make this :class:`Type` is to make a subclass of `Type`
and define ``filter`` and ``values_eq_approx`` in the subclass:

.. code-block:: python

   from aesara.graph.type import Type

   class Double(Type):

       def filter(self, x, strict=False, allow_downcast=None):
           # See code above.
           ...

       def values_eq_approx(self, x, y, tolerance=1e-4):
           # See code above.
           ...

   double = Double()

``double`` is then an instance of :class:`Type`\ `Double`, which in turn is a
subclass of `Type`.

There is a small issue with defining ``double`` this way. All
instances of `Double` are technically the same :class:`Type`. However, different
`Double`\ :class:`Type` instances do not compare the same:

.. testsetup::

   from aesara.graph.type import Type

   class Double(Type):

       def filter(self, x, strict=False, allow_downcast=None):
           if strict:
               if isinstance(x, float):
                   return x
               else:
                   raise TypeError('Expected a float!')
           elif allow_downcast:
               return float(x)
           else:   # Covers both the False and None cases.
               x_float = float(x)
               if x_float == x:
                   return x_float
               else:
                    raise TypeError('The double type cannot accurately represent '
                                    'value %s (of type %s): you must explicitly '
                                    'allow downcasting if you want to do this.'
                                    % (x, type(x)))

       def values_eq_approx(self, x, y, tolerance=1e-4):
           return abs(x - y) / (abs(x) + abs(y)) < tolerance

       def __str__(self):
           return "double"

   double = Double()

>>> double1 = Double()
>>> double2 = Double()
>>> double1 == double2
False

Aesara compares :class:`Type`\s using ``==`` to see if they are the same.
This happens in :class:`DebugMode`.  Also, :class:`Op`\s can (and should) ensure that their inputs
have the expected :class:`Type` by checking something like ``if x.type == lvector``.

There are several ways to make sure that equality testing works properly:

 #. Define ``Double.__eq__`` so that instances of type Double
    are equal. For example:

    .. testcode::

        def __eq__(self, other):
            return type(self) is Double and type(other) is Double

 #. Override :meth:`Double.__new__` to always return the same instance.
 #. Hide the Double class and only advertise a single instance of it.

Here we will prefer the final option, because it is the simplest.
:class:`Op`\s in the Aesara code often define the :meth:`__eq__` method though.


Untangling some concepts
========================

Initially, confusion is common on what an instance of :class:`Type` is versus
a subclass of :class:`Type` or an instance of :class:`Variable`. Some of this confusion is
syntactic. A :class:`Type` is any object which has fields corresponding to the
functions defined above. The :class:`Type` class provides sensible defaults for
all of them except ``filter``, so when defining new :class:`Type`\s it is natural
to subclass :class:`Type`. Therefore, we often end up with :class:`Type` subclasses and
it is can be confusing what these represent semantically. Here is an
attempt to clear up the confusion:


* An **instance of :class:`Type`** (or an instance of a subclass)
  is a set of constraints on real data. It is
  akin to a primitive type or class in C. It is a *static*
  annotation.

* An **instance of :class:`Variable`** symbolizes data nodes in a data flow
  graph. If you were to parse the C expression ``int x;``, ``int``
  would be a :class:`Type` instance and ``x`` would be a :class:`Variable` instance of
  that :class:`Type` instance. If you were to parse the C expression ``c = a +
  b;``, ``a``, ``b`` and ``c`` would all be :class:`Variable` instances.

* A **subclass of :class:`Type`** is a way of implementing
  a set of :class:`Type` instances that share
  structural similarities. In the ``double`` example that we are doing,
  there is actually only one :class:`Type` in that set, therefore the subclass
  does not represent anything that one of its instances does not. In this
  case it is a singleton, a set with one element. However, the
  :class:`TensorType`
  class in Aesara (which is a subclass of :class:`Type`)
  represents a set of types of tensors
  parametrized by their data type or number of dimensions. We could say
  that subclassing :class:`Type` builds a hierarchy of :class:`Type`\s which is based upon
  structural similarity rather than compatibility.


Final version
=============

.. testcode::

   from aesara.graph.type import Type

   class Double(Type):

       def filter(self, x, strict=False, allow_downcast=None):
           if strict:
               if isinstance(x, float):
                   return x
               else:
                   raise TypeError('Expected a float!')
           elif allow_downcast:
               return float(x)
           else:   # Covers both the False and None cases.
               x_float = float(x)
               if x_float == x:
                   return x_float
               else:
                    raise TypeError('The double type cannot accurately represent '
                                    'value %s (of type %s): you must explicitly '
                                    'allow downcasting if you want to do this.'
                                    % (x, type(x)))

       def values_eq_approx(self, x, y, tolerance=1e-4):
           return abs(x - y) / (abs(x) + abs(y)) < tolerance

       def __str__(self):
           return "double"

   double = Double()


We add one utility function, ``__str__``. That way, when we print
``double``, it will print out something intelligible.
