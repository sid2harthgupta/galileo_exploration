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

### Deep lookup to explore if dependent functions changed

#### The problem

The approach to hashing described above does not account for changes to the code of helper functions. Ex-

```python
def helper(x):
    return x + 10

@function_fingerprint
def foo(x):
    return helper(x) * 2

print(f"Hash for foo(x): {foo.get_hash()}")

def helper(x):
    return x + 12

@function_fingerprint
def foo(x):
    return helper(x) * 2

print(f"Hash for foo(x): {foo.get_hash()}")
```

Output:
```
Hash for foo(x): a80509d0b24e455b3adad7f6f809423d
Hash for foo(x): a80509d0b24e455b3adad7f6f809423d
```

#### Possible solution: Deep Lookup

```python
import hashlib
import inspect
import dis
from functools import wraps

def function_fingerprint(func):
    """
    Decorator that adds a get_hash() method to a function.
    Includes both bytecode and source code of external functions it calls.
    """
    original_func = getattr(func, '__wrapped__', func)
    
    def get_hash(verbose=False):
        # Use a cache to prevent infinite recursion
        cache = {}
        result = _get_deep_hash(original_func, cache, verbose)
        return result
    
    def _get_deep_hash(fn, cache, verbose=False):
        # Return cached hash if already processed
        fn_id = id(fn)
        if fn_id in cache:
            return cache[fn_id]
        
        # Get function's bytecode signature
        code = fn.__code__
        bytecode = code.co_code
        constants = code.co_consts
        
        # Get source code if possible
        try:
            source = inspect.getsource(fn)
            print(f"Found source code: `{source}`")
        except Exception:
            source = f"<source unavailable for {fn.__name__}>"
            print(f"Source code not available: {source}")
        
        # Find all function calls
        called_functions = []
        for instruction in dis.get_instructions(fn):
            if instruction.opname in ('LOAD_GLOBAL', 'LOAD_NAME'):
                name = instruction.argval
                if name in fn.__globals__ and callable(fn.__globals__[name]):
                    called_fn = fn.__globals__[name]
                    # Skip builtins and already processed functions
                    if not inspect.isbuiltin(called_fn) and id(called_fn) not in cache:
                        called_functions.append((name, called_fn))
        
        # Calculate hashes for called functions
        called_hashes = []
        for name, called_fn in called_functions:
            called_hash = _get_deep_hash(called_fn, cache, verbose)
            called_hashes.append(f"{name}:{called_hash}")
            
            if verbose:
                print(f"Including dependency: {name} with hash {called_hash}")
        
        # Build the fingerprint
        fingerprint = (
            bytecode +
            str(constants).encode() +
            '|'.join(called_hashes).encode()
        )
        
        # Calculate and cache hash
        hash_value = hashlib.md5(fingerprint).hexdigest()
        cache[fn_id] = hash_value
        
        if verbose:
            print(f"Hash for {fn.__name__}: {hash_value}")
            
        return hash_value
    
    @wraps(original_func)
    def wrapper(*args, **kwargs):
        return original_func(*args, **kwargs)
    
    wrapper.get_hash = get_hash
    wrapper.__wrapped__ = original_func
    
    return wrapper
```

Test:
```python
def helper(x):
    return x + 10

@function_fingerprint
def foo(x):
    return helper(x) * 2

print(f"Hash for foo(x): {foo.get_hash()}")

def helper(x):
    return x + 12

@function_fingerprint
def foo(x):
    return helper(x) * 2

print(f"Hash for foo(x): {foo.get_hash()}")
```

Output:
```
Found source code: `@function_fingerprint
def foo(x):
    return helper(x) * 2
`
Found source code: `def helper(x):
    return x + 10
`
Hash for foo(x): 785c818c8f50516c81219fef87815911
Found source code: `@function_fingerprint
def foo(x):
    return helper(x) * 2
`
Found source code: `def helper(x):
    return x + 12
`
Hash for foo(x): 69a38c6b116089aeed7d047a2add4e25
```

##### Deeper dive

The `instruction`s provide a lot of useful details:
```
Instruction(opname='RESUME', opcode=151, arg=0, argval=0, argrepr='', offset=0, starts_line=4, is_jump_target=False, positions=Positions(lineno=4, end_lineno=4, col_offset=0, end_col_offset=0))
Instruction(opname='LOAD_GLOBAL', opcode=116, arg=1, argval='helper', argrepr='NULL + helper', offset=2, starts_line=6, is_jump_target=False, positions=Positions(lineno=6, end_lineno=6, col_offset=11, end_col_offset=17))
Instruction(opname='LOAD_FAST', opcode=124, arg=0, argval='x', argrepr='x', offset=12, starts_line=None, is_jump_target=False, positions=Positions(lineno=6, end_lineno=6, col_offset=18, end_col_offset=19))
Instruction(opname='CALL', opcode=171, arg=1, argval=1, argrepr='', offset=14, starts_line=None, is_jump_target=False, positions=Positions(lineno=6, end_lineno=6, col_offset=11, end_col_offset=20))
Instruction(opname='LOAD_CONST', opcode=100, arg=1, argval=2, argrepr='2', offset=22, starts_line=None, is_jump_target=False, positions=Positions(lineno=6, end_lineno=6, col_offset=23, end_col_offset=24))
Instruction(opname='BINARY_OP', opcode=122, arg=5, argval=5, argrepr='*', offset=24, starts_line=None, is_jump_target=False, positions=Positions(lineno=6, end_lineno=6, col_offset=11, end_col_offset=24))
Instruction(opname='RETURN_VALUE', opcode=83, arg=None, argval=None, argrepr='', offset=28, starts_line=None, is_jump_target=False, positions=Positions(lineno=6, end_lineno=6, col_offset=4, end_col_offset=24))
```

Explanation:
```
RESUME - Starts function execution
LOAD_GLOBAL with argval='helper' - Loads the global helper function
LOAD_FAST with argval='x' - Loads the parameter x
CALL with argval=1 - Calls helper(x) with 1 argument
LOAD_CONST with argval=2 - Pushes constant value 2
BINARY_OP with argrepr='*' - Multiplies the result of helper(x) by 2
RETURN_VALUE - Returns the final value
```
