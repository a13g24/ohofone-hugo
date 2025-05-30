---
date: '2025-05-24T05:05:28-06:00'
draft: false
title: 'Robust Testing of the Template Method Pattern in Python with Pytest and Abstract Factories'
katex: true
summary: 'A Safe and Elegant Approach to Testing the Template Method Pattern'
tags: 
  - Design Patterns
  - Software Testing
  - Python
  - Pytest
  - Template Method Pattern
  - Abstract Factory Pattern
---

## Introduction

Testing abstract anything has been a challenge for developers since the dawn of object-oriented programming. The Template Method Pattern (TMP) is a common design pattern that allows you to define the skeleton of an algorithm in a method, deferring some steps to subclasses. This pattern is particularly useful when you want to allow subclasses to redefine certain steps of an algorithm without changing its structure.

The difficulty with testing the TMP is that the *Template Method* itself (the special method that contains the main algorithm), occupies a peculiar position of being both concrete but relying on abstract methods elsewhere in the abstract class that encloses it. Bijlsma et al. refer to these kinds of methods as **semi-abstract methods**:

> A semi-abstract method is a concrete method that depends, directly or indirectly, on one or more abstract operations, or calls a method of an abstract class defined in a class hierarchy elsewhere.

The *Template Method* itself is a special kind of semi-abstract method because it relies on a concrete method, but that concrete method is within an abstract class, and that concrete method depends on other abstract methods within that abstract class. 

## A Conventional Approach

A common approach for testing the TMP is to test the concrete subclasses of the TMP abstract class. However, there are two major problems with this approach:

1. It's error-prone since the output of the *Template Method* itself differs for each subclass.
2. Even if you managed to do an exhaustive test for each concrete subclass of the abstract TMP class, you're likely to forget to test new subclasses since the *Template Method* itself is concrete, so your tests will always pass—you have to remember to implement a new test for each new concrete implementation.

## The 3PACT Solution

Bijlsma et al. proposes a solution they call **3PACT** which uses an [Abstract Factory Pattern](https://refactoring.guru/design-patterns/abstract-factory) which is "safe and elegant". Indeed, after implementing it myself in Python, I must say I agree with that description.

Why? Since we need to test *both* the *Template Method* itself as well as how it varies with each concrete instance that inherits from the base TMP abstract class, we still need to test all the variations of the *Template Method* itself. But the problem here is there will be a lot of object creation for possible injected dependencies for a given concrete class; which should not be done in a test class. We therefore delegate that creation to an Abstract Factory to handle the creation of each concrete class. We also create an abstract base test factory that we inherit from with concrete test factories for each concrete subclass of the original TMP abstract class.

### Huh? 

Ya I know, it's confusing at first. An example is much easier to describe this method. I'll be using an example directly from the paper, but implemented in Python and Pytest.


```python
"""
The Trivial Case 
----------------

For sake of completeness, note that testing the classes here are trivial because we 
need only implement 3 test cases testing the methods of each concrete class. They're
simple because there's no abstract class or method defined. But since there's redundant 
code in C and B, we can refactor this to use the Template Method Pattern (TMP) to avoid 
the redundancy.
"""
class A:
    def say_hello(self) -> str:
        return "Hello from A"


class B(A):
    def add_squares(self, base: list[int]) -> list[int]:
        res = []
        for val in base:
            res.append(val * val)

        return res


class C(A):
    def add_pos_vals(self, base: list[int]) -> list[int]:
        res = []
        for val in base:
            if val > 0:
                res.append(val)

        return res
```

Refactoring this to the TMP

```python
"""
Refactored to use the Template Method pattern
"""

from abc import ABC, abstractmethod


class A(ABC):
    def process(self, base: list[int]) -> list[int]:
        """
        The *Template Method* itself that defines the skeleton of an algorithm.
        This is an example of a *semi-abstract* method since it relies on calls
        to the abstract method `use`, which must be implemented by subclasses.
        """
        res = []
        for val in base:
            self.use(res, val)
        return res

    def say_hello(self) -> str:
        return "Hello"

    @abstractmethod
    def use(self, lst: list[int], element: int) -> None:
        pass


class B(A):
    def use(self, lst: list[int], element: int) -> None:
        lst.append(element * element)


class C(A):
    def use(self, lst: list[int], element: int) -> None:
        if element > 0:
            lst.append(element)
```

#### Defining the Abstract Factory (and Factories)

Just like all cases of the Abstract Factory Pattern, we define an interface to interact with.

```python
from abc import ABC, abstractmethod

class AFactory(ABC):
    """
    Our abstract factory class that defines the interface for creating instances
    of the Template Method (TM) subclasses.
    """

    @abstractmethod
    def create(self) -> A:
        pass

    @staticmethod
    def get_factory(name: str) -> "AFactory":
        """
        Static method to get the appropriate factory based on the test
        class.
        """
        factories = {
            "B": BFactory(),
            "C": CFactory(),
        }

        return factories.get(name, None)


# Declare interfaces for a set of distinct but related subclasses of the TM
# to test. In this case, `create` is the only interface method that we use.


class BFactory(AFactory):
    def create(self) -> A:
        return B()


class CFactory(AFactory):
    def create(self) -> A:
        return C()
```

#### Testing with Pytest

Bijlsma et al. use a syntax more appropriate for JUnit testing. Indeed, their example can be copied nearly verbatim if you use Python's standard `unittest` module, but let's implement the tests in pytest instead.

```python
# Test classes are the `clients` of the factory so they only know the interface
# declared by the abstract factory and the name of the class it wants to test.


import pytest


class NotATest:
    """
    This is a Mixin class that prevents pytest from discovering a class that
    inherits from it. However, any subclass of the class you apply this mixin to
    **will** be considered a test class by pytest. This is useful because you
    must set `__test__ = False` in an abstract test class to prevent pytest from
    discovering it as a test class, but you still want to ensure that any
    subclass of that abstract test class is considered a test class by pytest.
    You would otherwise have to set `__test__ = True` in each subclass, which is
    not practical if you have many subclasses.

    Links
    ^^^^^
    - `Source <https://github.com/pytest-dev/pytest/issues/8612>`_
    """

    def __init_subclass__(cls):
        cls.__test__ = NotATest not in cls.__bases__


class BaseTestA(ABC, NotATest):
    @pytest.fixture(autouse=True)
    def setup_instance(self):
        """
        An instance of the class under test is assigned to the `a` attribute by
        calling method `create` of the factory that is received by calling
        `get_factory` of the `AFactory` class.

        In Pytest, we can't use constructors for setup, so we use a fixture with
        `autouse=True` to ensure it runs before each test method.

        This fixture will retrieve the `TESTCLASS` attribute, which should be
        set in each test subclass, and use it to get the appropriate factory
        to create an instance of the class to be tested.
        """
        test_class = getattr(self, "TESTCLASS", None)
        factory = AFactory.get_factory(test_class)
        self.a: A = factory.create()

    def test_say_hello(self):
        """
        We can still test the concrete methods of the TM even though
        they're within an abstract test class
        """
        assert self.a.say_hello() == "Hello"

    @abstractmethod
    def test_process(self) -> None:
        """
        Abstract method to test the `process` method of the TM.
        Test subclasses will implement this method to test specific behaviors.
        """
        pass


class TestB(BaseTestA):
    # This is the class name that will be used to get the factory via the static
    # method that will create an instance of the class to be tested.
    TESTCLASS = "B"

    @pytest.fixture
    def test_data(self):
        """
        Prepares the test data according to the specific functionality of the
        abstract method (`use` in this case) that will be tested.
        """
        # Receive a concrete factory based on the class name
        # Then it can use the factory to create an instance of the class via the
        # interface declared by the abstract factory. Since concrete subclasses of
        # the TM might require different parameters including other objects, to be
        # passed to the constructor, the abstract factory can be used to create the
        # instance without the client (testing class) needing to know the details of
        # the concrete class.
        return [1, 2, -3], [1, 4, 9]

    def test_process(self, test_data) -> None:
        """The TM test is redefined for doing the right test."""
        input_list, expeced = test_data
        assert self.a.process(input_list) == expeced


class TestC(BaseTestA):
    TESTCLASS = "C"

    @pytest.fixture
    def test_data(self):
        return [1, 2, -3], [1, 2]

    def test_process(self, test_data):
        """Again the TM test is redefined for doing the right test."""
        input_list, expected = test_data
        assert self.a.process(input_list) == expected


```

Simply add more concrete subclasses of `BaseTestA` to test other concrete 
subclasses of `A`. Since we delegate the responsibility to create objects of the
classes to be tested to an abstract factory, we don't clutter our test classes
with the creation of objects.

Furthermore, `BaseTestA` allows us to run every test uniformly since it defines
a common interface. Employing this technique alse ensures each test subclass 
implements its own test for the template method itself (`process`) of the TM.

We need only remember 2 things:
1. Create a test class for each subclass of `A`.
2. Create a factory for each subclass of `A`.

This forms a Parallel Architecture of Class Testing (PACT). In this case, there's
three parallel hierarchies:
1. The test hierarchy
2. The factory hierarchy
3. The domain class hierarchy

This is why it's known as the 3PACT hierarchy.

## Conclusion

Using the 3PACT approach allows you to test the Template Method Pattern in a way that is both safe and elegant. By using an abstract factory to create instances of the concrete subclasses, you can ensure that your tests are focused on the behavior of the *Template Method* itself, while also allowing for easy extension when new subclasses are added. This approach minimizes the risk of forgetting to test new subclasses and ensures that your tests remain maintainable and clear. 

## References
- Bijlsma, A., Passier, H.J.M, Pootjes, H.J., & Sturrman, S. (2018). Template Method test pattern. *Information Processing Letters*, 139, 8-12. https://doi.org/10.1016/j.ipl.2018.06.008 access for free [here](https://research.sylviastuurman.nl/wp-content/uploads/2018/08/TEST_TEMPLATE_METHOD.pdf)