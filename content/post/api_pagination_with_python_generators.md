---
date: "2025-02-22T23:29:55-07:00"
draft: false
title: "Api Pagination With Python Generators"
katex: true
summary: "Python generators are an underutilized tool.  In this article, we demonstrate how they can be used for pagination on API calls."
tags: ["python", "api", "pagination", "generators"]
---

Python generators are an underutilized tool.  In this article, we demonstrate how they can be used for pagination on API calls.

## What is a Generator?

The best way to understand generators is in contrast to normal functions. In Python, you define a function as follows:

```python
def my_function(optional_params):
	# do something
	return something
```

When a function hits the `return` statement, it'll immediately exit and return whatever you've supplied as its argument. However, any statements after `return` will not be executed.

```python
def concat(a: str = "Hello", b: str = "World"):
	return "returned too soon"
	result = a + " " + b # this will never run

# Prints "returned too soon" rather than "Hello World"
print(concat())
```

Generators are interesting in that they can return a value early to the caller, and be called again, remembering where they left off.

```python
def get_next_int():
	i = 1
	while True:
		yield i # Generators use yield to return a value
                # The function is suspended, and returns
                # the value yielded. 

		# Everything below this comment will be ran
		# after the next call. Unlike return statements
		if i == 4: 
			break
			
		i += 1
```

You can actually make this function produce infinitely many integers by removing the conditional `if i == 4` and `break`ing from the loop when the counter is 4. How is that possible? Because generators are *lazy*, meaning they don't compute and store all values at once. Rather they generate values *on the fly*, yielding them one at a time and only when requested to do so. 

There's some powerful implications to this:
1. They're memory efficient because they don't store all of the computed data into memory, but only retrieve the next item and return it to the caller. This is great when dealing with large datasets or infinite sequences.
2. The computation is on-demand since it must be called with `next()` or some iterator that *drives* it (e.g. a `for` or `while` loop or comprehension of some sort). This allows for interesting alterations in the flow of control.

More on this *driver* point. Unlike functions that can simply be called and immediately executed, generators need to be driven in order to retrieve the next value.

```python
def func():
	print("I'm executed when called")


# Immediately prints "I'm executed when called"
func()

def generator():
	for s in ("I'm", "executed", "when", "driven"):
		yield s

g = generator()
print(next(g)) # "I'm"
print(next(g)) # "executed"
print(next(g)) # "when"
print(next(g)) # "driven"

# Note that when generators reach the end of
# their sequence (if it's finite), it yields a 
# StopIteration exception. This is not the case
# when driven by while/for loops or comprehensions
print(next(g()))

# Or more succinctly
for s in generator():
	print(s)

# There's also a neat trick called generator delegation
# which delegates the iteration process to another iterator.
# You use `yield from` to do this. So we can rewrite 
# our generator() above more succinctly as,
def generator():
	yield from ("I'm", "executed", "when", "driven")

for s in generator():
	print(s)
```

## Using Generators for API Pagination

### What is Pagination?

Imagine you have an API endpoint for an e-commerce store that returns thousands of products. For a variety of reasons, you can't retrieve all *n* thousands of products at once. So, API developers break it up by subdividing the whole into pages of some lower number of products. You then retrieve all of these products by writing some code that will obtain the next page of products, and iterate over all of the pages until you've retrieved them, accumulating and storing each page of products as you go.  

### Pagination and Generators

There are several ways to implement pagination in Python. One is using recursive calls, which I'd argue is an anti-pattern, especially because Python has a default limit of 1000 recursive calls, which, as you can already tell, if you have 6,000 products, you'll exceed that limit rather quickly. Note that you can increase this default, but it's bad practice due to the risk of stack overflow. 

Another approach is to use iteration, like a `for` or `while` loop. This eliminates the recursion limit problem. Indeed, you can collect the products and add them to a list as you go, or better yet, use a generator, which will only ever retrieve the next item, and then a *driver* like Python's  `list()` to assemble them in a list.

We'll be using the Python [responses](https://github.com/getsentry/responses) library to mock the pages. This is a great way to practice API calls locally.

```python
import responses
import requests
from typing import Generator

ENDPOINT = "/products"
URL = f"https://e-commerce.com/{ENDPOINT}"


def get_data_with_pagination() -> Generator[dict, None, None]:
    # -------------------------------------------------------
    # Helpers
    # -------------------------------------------------------
    def next_page(data: dict) -> int | None:
        """
        As a reminder, the data structure returned by the API is:

        {
            "pagination": {
	            "previous": 1, 
	            "current": 2, 
	            "next": 3
	        },
            "data": [
	            {
		            "page_2_data": "there"
		        }, 
		        {
			        "and_page_2_data": "there"
			    }
			],
        }

        Note that previous, current, and next vary depending 
        on the page, including being None when there is no 
        previous or next page, which is how we determine when
        to stop iterating.
        """
        return data.get("pagination", {}).get("next")
    
    def has_next_page(data: dict) -> bool:
        return bool(next_page(data))

    # -------------------------------------------------------
    # Main
    # -------------------------------------------------------
    page = 1
    while True:
        response = requests.get(URL, params={"page": page})

        data = response.json()

		# Since the caller is using `list()` to drive & store the
		# resulting data, but the data is a list of dictionaries,
		# if we only used `yield` rather than `yield from`, then
		# we would end up with a list of list of dictionaries. 
		# This would require us to unpack and flatten each so
		# that we have only a single list of all of the collected
		# dictionaries. Rather than do that, we use generator 
		# delegation with `yield from` to flatten the lists 
		# in the data, thus only returning the dicts and allowing
		# the caller to have a single list of dicts.
        yield from data.get("data", [])

		# Exits the loop when there's no next page
        if not has_next_page(data):
            break

        # Modifies the URL each iteration to get the next page
        # e.g. https://e-commerce.com/products?page=1 
        # e.g. https://e-commerce.com/products?page=2 
        page += 1


@responses.activate
def test_responses():
    """
    Mocks the API responses for the pagination.
    """
    # Page 1
    responses.get(
        URL,
        json={
            "pagination": {"previous": None, "current": 1, "next": 2},
            "data": [{"page_1_data": "here"}, {"and_page_1_data": "here"}],
        },
        status=200,
    )

    # Page 2
    responses.get(
        URL,
        json={
            "pagination": {"previous": 1, "current": 2, "next": 3},
            "data": [{"page_2_data": "there"}, {"and_page_2_data": "there"}],
        },
        status=200,
    )

    # Page 3
    responses.get(
        URL,
        json={
            "pagination": {"previous": 2, "current": 3, "next": None},
            "data": [{"page_3_data": "everywhere"}, {"and_page_3_data": "everywhere"}],
        },
        status=200,
    )

    # Collects the data from the generator.
    # Note how we use `list()` to drive the generator
    data = list(get_data_with_pagination())

	# OUTPUTs:
	# {'page_1_data': 'here'}
	# {'and_page_1_data': 'here'}
	# {'page_2_data': 'there'}
	# {'and_page_2_data': 'there'}
	# {'page_3_data': 'everywhere'}
	# {'and_page_3_data': 'everywhere'}
    for datum in data:
        print(datum)

if __name__ == "__main__":
    test_responses()
```

## Conclusion

The result of using generators is a more elegant and memory efficient solution to pagination. It's also more flexible, allowing for more control over the flow of data and how you collect it.