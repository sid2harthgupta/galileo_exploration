Quick experiment to see if it is possible to:
1. Export the source code of a function using a Python decorator,
2. Detect changes to the function code so it can be versioned

```python
def function_fingerprint(func):
    """
    Decorator that adds a get_hash() method to a function to get its current state hash.
    The hash changes if the function definition or closure variables change.
    """
    def get_hash():
        # Get function source code (includes signature and implementation)
        source = inspect.getsource(func)
        print(f"Source- type={type(source)}, content=`{source}`")
        # Could even send this to the Galileo server in a background thread
        
        # Create a hash of the function state
        return hashlib.md5(source.encode()).hexdigest()
    
    # Attach the get_hash method to the function
    func.get_hash = get_hash
    
    return func
```

Test case:
```python
@function_fingerprint
def foo(x):
    return x * 2

print(f"Hash for foo(x): {foo.get_hash()}")
```

Output:

```
Source- type=<class 'str'>, content=`@function_fingerprint
def foo(x):
    return x * 2
`
Hash for foo(x): a396313b6ccb49b61236614ed4600a9b
```


Where the above fails:
Since hashing depends on the string representation of the code, meaningless changes in the code will result in a different hash.
Also, if the funciton depends on external functions and constants, and those functions/constants change, then the hash will NOT change.
For example:

```python
@function_fingerprint
def foo(x):
    return x * 2

print(f"Hash for foo(x): {foo.get_hash()}")

@function_fingerprint
def foo(x):
    return x *  2  # Note the extra space

print(f"Hash for foo(x): {foo.get_hash()}")

@function_fingerprint
def foo(y):
    return y * 2

print(f"Hash for foo(y): {foo.get_hash()}")

# Redefine with different parameters
@function_fingerprint
def foo(x, y):
    return x * y

print(f"Hash for foo(x, y): {foo.get_hash()}")
```

This results in:
```
Hash for foo(x): a396313b6ccb49b61236614ed4600a9b
Hash for foo(x): be5d930775f5f14c482f76f251c493cb
Hash for foo(y): 649b96e865161ddfa7e9e5da9758d6b1
Hash for foo(x, y): 2e7ff612d4ebc1721435710257224b35
```

Let's make it more robust:
```python
def function_fingerprint(func):
    """
    Decorator that adds a get_hash() method to a function to get its behavioral hash.
    Ignores whitespace and variable names to focus on functional behavior.
    """
    # Get original function if already decorated
    original_func = getattr(func, '__wrapped__', func)
    
    def get_hash():
        # Get the code object
        code = original_func.__code__
        
        # Extract bytecode (actual operations)
        bytecode = code.co_code
        
        # Extract constants (literal values)
        constants = code.co_consts
        
        # Get structure information (number of arguments)
        argcount = code.co_argcount
        
        # Create a hash from these components
        fingerprint = (
            bytecode + 
            str(constants).encode() + 
            str(argcount).encode()
        )
        
        return hashlib.md5(fingerprint).hexdigest()
    
    # Create wrapper to preserve original function
    @wraps(original_func)
    def wrapper(*args, **kwargs):
        return original_func(*args, **kwargs)
    
    # Add get_hash method
    wrapper.get_hash = get_hash
    
    # Store original function for reference
    wrapper.__wrapped__ = original_func
    
    return wrapper
```

This results in the output:

```
Hash for foo(x): 0f44306c99312ffde3b6e996365e260d
Hash for foo(x): 0f44306c99312ffde3b6e996365e260d
Hash for foo(y): 0f44306c99312ffde3b6e996365e260d
Hash for foo(x, y): 96aaf524d6271fefc7ba2c5d5c8392eb
```

This makes the hash robust to formatting/naming changes. It still does not explore the changes to functions defined outside.
It might be possible to address those as well by creating a 'deep' fingerprinting system but the takeaway is that:
1. It is be possible to access the code of a function,
2. Functions can also be versioned reasonably well using an abstract syntax tree
Put together there might be interesting ways to get broader context of the code without needing to get user to install add-ons. Furhter, it might be possible to create a versioning system for individual functions as users proceed in their development process.
