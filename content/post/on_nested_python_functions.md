---
date: '2025-03-09T23:02:13-06:00'
draft: false
title: 'On Nested Python Functions'
katex: true
summary: 'A discussion on the use of nested functions in Python and why you should use them.'
tags: ['python', 'refactoring', 'clean code']
---

## Introduction

There's a lot of opinions on nested (or "inner") functions in Python. [RealPython.com](https://realpython.com/inner-functions-what-are-they-good-for/) already has a fantastic article about them, especially with respect to [closures](https://stackoverflow.com/questions/36636/what-is-a-closure) and [decorators](https://realpython.com/primer-on-python-decorators/), so I won't reinvent the wheel by discussing those issues here. Instead, I want to diverge from their recommendation to put inner functions in the global scope as "private" (indicated by a `_` prefix) functions, and why you ought to use Martin Fowler's *Extract Method/Function* refactoring technique with inner functions (including inside of methods).

## The Problem

First, prepending `_` to indicate so-called private functions/methods doesn't actually enforce their non-access from outsiders to that module or class from the outside world. 

```python
# module_a.py

def _private() -> str:
	return "I'm still accessible to the outside world!"

# module_b.py

from module_a import _private

print(_private()) # Yup, this works

# module_c.py

import module_a

print(module_a._private()) # So does this

# module_d.py

from module_a import * # This does not
```

So, `module_b` and `module_c` can indeed access the `_private` function from `module_a`. The only "exception" is when using `*`  to import everything in a module (a frowned upon practice anyways). Moreover, using `__all__` to control this sort of access in a module does not change the access behavior demonstrated in `module_b`  and `module_c`. This is no different for classes, 

```python
class SomeClass():
	def _private_method(self):
		return "Yup, still accessible"

print(SomeClass()._private_method()) # This works
```

And no, using double `__` (name mangling in Python) doesn't count either.

```python
class SomeClass():
	def __private_method(self):
		return "Yup, still accessible"

print(SomeClass()._SomeClass__private_method()) # So does this
```

I mean sure it makes it less likely to be accessed, but it's still technically accessible.

Now I'm not discouraging the use of any of these techniques. It's a well-known enough pattern in dynamic languages that a competent dev will know not to touch it from without. However, I will suggest that name mangling with class methods is unnecessary since it's mostly meant to be used for inheritance to avoid sub-classes from erroneously overwriting a method of the parent by the same name. This is an extreme edge case in my experience, and besides, if you're making deep inheritance trees, you need to rethink your design anyways. And please, for the love of God, don't use it in a class when you have no intention to inherit from it. A single `_` will do just fine.

## The Solution

Again, there are cases when indicating a private function/method with this convention at a higher scope is fine. But most times, helper functions are one-off things. Further, many make the mistake of thinking you only use functions when you start reusing logic. That's one philosophy. The other, championed by the likes of Uncle Bob, is using them to encapsulate and describe the intent of the author. For example, if you have a complicated boolean expression in a conditional, extract that into a function, and rename it to reveal what the hell it's doing.

```python
def get_order_items(order: dict) -> list[dict]:
	order = []
	
	# The boolean expression is hard to read the intention
	# of the author
	if (order and order["visible"] and order["purchasable"]):
		order = order["order_items"]
		
	return order

# Extract this and name it something more meaningful

def is_order_ready(order: dict) -> bool:
	return (
		order 
		and order["visible"] 
        and order["purchasable"]
	)

# Then the old one becomes

def get_order_items(order: dict) -> list[dict]:
	order = []
	
	if (is_order_ready(order)):
		order = order["order_items"]

	return order
```

As you can see, the second case is much easier to understand at a glance what the author intended. Are we likely to repeat this logic elsewhere and require this specific function? Maybe, but I think one ought to begin as local as possible, and only move it to a higher scope if needed. 

```python
def get_order_items(order: dict) -> list[dict]:
	# -----------------------------------------
	# Helpers
	# -----------------------------------------
	def is_order_ready(order: dict) -> bool:
		return (
			order 
            and order["visible"] 
            and order["purchasable"]
		)
		
	# -----------------------------------------
	# Main
	# -----------------------------------------
	order_items = []
	
	if (is_order_ready(order)):
		order_items = order["order_items"]
		
	return order_items
```

I've rearranged things slightly and added some comments to separate the *helpers* from the main logic. You even have the added benefit of not needing to pass the item being manipulated since the inner function can access the enclosing scope's variables.

```python
def get_order_items(order: dict) -> list[dict]:
	# -----------------------------------------
	# Helpers
	# -----------------------------------------
	def is_order_ready() -> bool:
		# Here order is accessed from the enclosing scope
		return (
			order 
			and order["visible"] 
			and order["purchasable"]
		)
		
	# -----------------------------------------
	# Main
	# -----------------------------------------
	order_items = []

	# It cleans up the call as well 
	if (is_order_ready()):
		order_items = order["order_items"]
		
	return order_items
```

This can have the undesired effect of too much indirection. I usually default to this, but if I really want to emphasize what is being passed, I'll have the inner function set a parameter that takes the object being manipulated.

This pattern of using inner functions to *extract method* adheres to Uncle Bob's keen observation that

>Concepts that are closely related should be kept vertically close to each other [...] their vertical separation should be a measure of how important each is to the understandably of the other. (Martin, p. 80) 

The less mental overhead a developer has to do to understand your code (including your future self), the better for maintenance and extensibility. And if you find you need this helper elsewhere some time later, it's easier to move it to a wider scope than it is the other way around. 

This also doesn't break the [SRP](https://en.wikipedia.org/wiki/Single-responsibility_principle) as some might claim. In fact, it enforces it by ensuring each function, inner and outer, has a single responsibility. The nesting of the functions does not break this, it's simply utilizing the features of the language to organize it.

>If the language supports nested functions, nest the extracted function inside the source function. That will reduce the amount of out-of-scope variables to deal with [...] (Fowler, p. 108) 

This might seem like making mountains out of molehills, but as your module or class grows more complex, you will begin to pollute the global scope (or wider scopes in general) with one-off helper functions, and it will become increasingly difficult to find your way around. 

Remember, this is not limited to functions, but can (and should) be applied to methods as well. 

```python
class SomeClass():
    def get_order_items(self):
        # -----------------------------------------
        # Helpers
        # -----------------------------------------
        def is_order_ready() -> bool:
            return (
                self.order 
                and self.order["visible"] 
                and self.order["purchasable"]
            )
            
        # -----------------------------------------
        # Main
        # -----------------------------------------
        order_items = []

        if (is_order_ready()):
            order_items = self.order["order_items"]
            
        return order_items
```

The benefit of doing this in procedural code, is that it acts as a sort of *quasi* encapsulation. And in object-oriented code, it doesn't pollute the class with one-off methods that are only used in one place, and thus does not obfuscate the class' API.


## Conclusion

Thus concludes my highly opinionated rant on nested functions in Python. I hope you found it useful and that it will help you write cleaner, more maintainable code. Remember, the goal is to make your code as easy to understand as possible, and to make it as easy as possible to extend and maintain. It does no harm nesting functions in the beginning, since you can always move them to a wider scope if needed. But I promise, you will thank yourself a year from when you last touched that code, and you revisit it only to see all of your helper functions in one place, right where you need them. Providing you with immediate context and understanding of what you intended.


## References
- Fowler, M. (2019). Refactoring: Improving the design of existing code (2nd ed.). Addison-Wesley.
- Martin, R. (2009). Clean Code: A handbook of agile software craftsmanship. Prentice Hall. 