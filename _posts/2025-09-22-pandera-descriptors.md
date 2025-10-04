---
layout: post
title:  "Adding Descriptors to Pandera's Models"
date:   2025-09-22 12:00:00 -0700
categories: python open-source
---

How and why I updated Pandera's [DataFrameModel](https://pandera.readthedocs.io/en/latest/dataframe_models.html)
to use Python's descriptors for data attributes


> **TLDR:** Assigning [descriptors](https://docs.python.org/3/howto/descriptor.html) to class attributes 
> allows those attributes to behave like [Properties](https://docs.python.org/3/library/functions.html#property).
> The values can be computed lazily, only when needed,
> and referenced `.directly` instead of requiring a `.method_call()`

[skip to the recipe](https://github.com/unionai-oss/pandera/pull/2136/commits/a530024b0c2bf926333413f424f64cadbed3dada)

## Reusing Field definitions
While implementing type-checking for Pandas dataframes using Pandera,
I wanted to create re-usable Field definitions.
Repeating the definition of a `date` field in every `DataFrameModel` subclass
or using inheritance for individual fields felt clunky and was difficult to read.

It turns out each field in a DataFrameModel definition needs to be a unique instance.
You can read about the details [in this github issue](https://github.com/unionai-oss/pandera/issues/1680).

### The solution for reusable Fields
With a little help from Niels Bantilan, the maintainer of Pandera, we came up with this solution
using partials to define reusable Field definitions

```Python
from functools import partial
from pandera import DataFrameModel, Field

NormalizedField = partial(Field, ge=0, le=1)

class GoodModelDF(DataFrameModel):
    xnorm: float = NormalizedField()
    ynorm: float = NormalizedField()

class AlsoGoodModelDF(DataFrameModel):
    xnorm: float = NormalizedField()
```


## Debugging, and surprising behavior
While debugging the problems with reusable fields, I discovered the unexpected
behavior that motivated this change.

My first attempt to reuse a Field, resulted in a perplexing error:
```Python
NormalizedField: float = Field(ge=0, le=1)

class BadModelDF(DataFrameModel):
    field_0: float = GenericField
    field_1: float = GenericField  # Bug: this breaks the model
    
    class Config:
        strict = True
```
Calling `BadModelDF.validate(some_dataframe)` raised the exception:
`SchemaError: column 'field_0' not in DataFrameSchema {'field_1': <Schema Column(name=field_1, type=DataType(float64))>`
*The root cause of that error is due to the behavior of [DataFrameModel._build_columns_index](https://github.com/unionai-oss/pandera/blob/ede8a4354cb41a5ef28218f5fbcf7bd64a761cf7/pandera/api/pandas/model.py#L69)
, but is not relevant to this story.*


Digging into the [DataFrameModel source code](https://github.com/unionai-oss/pandera/blob/ede8a4354cb41a5ef28218f5fbcf7bd64a761cf7/pandera/api/dataframe/model.py#L116)
I saw that the `.__fields__` attribute should™ contain the data I'm looking for...
but when I viewed `BadModelDF.__fields__`, it was an empty dict `{}`!
And the `.__schema__` value was `None`.
This was frustrating... I defined a valid DataFrameModel, why would its attributes all be empty, uninitialized?

### The deeply unsatisfying solution
The `DataFrameModel` class had a sort of secret `__init__` method.
You needed to call the [to_schema](https://github.com/unionai-oss/pandera/blob/ede8a4354cb41a5ef28218f5fbcf7bd64a761cf7/pandera/api/dataframe/model.py#L210) 
method (or any other method which calls it) first,
because `.to_schema()` populates those data attributes.


## The change

### Motivation

Being deeply unsatisfied with this "spooky action at a distance", I knew how it should™ work.
I expect the value of the attributes to always be correct.

We also don't want to initialize these values when the class is interpreted,
because it is computationally expensive. 
There are cases where we will never use the computed values, 
for example when we set `PANDERA_VALIDATION_ENABLED=False`.
In fact, I don't want to calculate the `__schema__` and all the other data attributes at all,
just to read the `__fields__` data.

### Property-like behavior and classes
IMHO: All™ data attributes on an object should behave like attributes, not methods.
I want to reference a value like `person.age >= 21` not `person.get_current_age() >= 21`.

Python [Properties](https://docs.python.org/3/library/functions.html#property)
provide this behavior for computed values on class instances.

Sadly, properties do not work on classes, they work on instances of the class.
The interpreter reads a property definition like this:
```Python

def MyClass:
    @property
    def value(self):
        return compute_the_value()
```
something like "when I create an instance of this class, make its `value` attribute a property".

Accessing a property of a class returns the `property` object `<property at 0x###>`,
not a computed value as we may have hoped.

Fortunately, [Descriptors](https://docs.python.org/3/howto/descriptor.html)
allow us to write class attributes which behave just like properties!

### The Implementation
A descriptor is a class, which has a `__get__` method with the appropriate signature.
We can assign an instance of the descriptor to a class attribute,
and it will behave just like a property, returning the computed value.

```Python
In [1]: class ValDesc:
   ...:     def __get__(self, obj, objtype=None):
   ...:         return 42
   ...: 

In [2]: class SomeClass:
   ...:     value = ValDesc()
   ...: 

In [3]: SomeClass.value
Out[3]: 42
```


### Changes to the Pandera DataClassModel
I had several goals for improving the Pandera Model interface:
- Eliminate the need to call `.to_schema()` to populate data attribues.
- Data attributes like `__fields__` should™ be idempotent, and always provide valid values.
- Avoid unnecessary computation, by only computing the values we need, when we need them.

To accomplish this, 
I added several Descriptor classes to the `DataClassModel`,
updated the old `get_` and `to_` class methods, and updated other class methods
to access the data attributes directly, while preserving the behavior of the API.


It worked! 
I improved the interface for Pandera's Model class, eliminating a confusing behavior.
This change to a core component of the package was not disruptive, 
and the full test suite passed without modification.
We also achieved some marginal performance improvements.


You can read the [full commit in the PR](https://github.com/unionai-oss/pandera/pull/2136/commits/a530024b0c2bf926333413f424f64cadbed3dada).
This is a brief illustration of some of the changes:
```Python
class _ClassDescriptor:
    def __init__(self):
        self.cache = {}


class _FieldsDescriptor(_ClassDescriptor):
    """Descriptor which allows __fields__ to act as a class property."""

    def __get__(self, obj, cls) -> TFields:
        if self.cache.get(cls) is None:
            self.cache[cls] = cls._collect_fields()

            for field, (annot_info, _) in self.cache[cls].items():
                if isinstance(annot_info.arg, TypeVar):
                    raise SchemaInitError(
                        f"Field {field} has a generic data type"
                    )

        return self.cache[cls]


class _SchemaDescriptor:
    def __get__(self, obj, cls):
        ...


class DataFrameModel(Generic[TDataFrame, TSchema], BaseModel):
    ...
    __fields__: ClassVar[TFields] = cast(TFields, _FieldsDescriptor())
    __schema__ = _SchemaDescriptor()
    ...

    # Update class methods to use the new descriptors, instead of calling class methods
    @classmethod
    def to_schema(cls) -> TSchema:
        """Create :class:`~pandera.DataFrameSchema` from the :class:`.DataFrameModel`."""
        return cls.__schema__

    @classmethod
    def to_yaml(cls, stream: Optional[os.PathLike] = None):
        """
        Convert `Schema` to yaml using `io.to_yaml`.
        """
        return cls.__schema__.to_yaml(stream)

    ...
```

Thanks for reading,

Lundy
