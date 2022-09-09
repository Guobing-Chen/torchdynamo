## TorchDynamo Deeper Dive

If you want to understand better what TorchDynamo is doing, you can set:
```py
torchdynamo.config.debug = True
```

which triggers useful (but spammy) printouts.

For example, the printouts for the first graph in the `toy_example`
above are:
```
__compiled_fn_0 <eval_with_key>.1
opcode         name     target                                                  args              kwargs
-------------  -------  ------------------------------------------------------  ----------------  --------
placeholder    a        a                                                       ()                {}
placeholder    b        b                                                       ()                {}
call_function  abs_1    <built-in method abs of type object at 0x7f9ca082f8a0>  (a,)              {}
call_function  add      <built-in function add>                                 (abs_1, 1)        {}
call_function  truediv  <built-in function truediv>                             (a, add)          {}
call_method    sum_1    sum                                                     (b,)              {}
call_function  lt       <built-in function lt>                                  (sum_1, 0)        {}
output         output   output                                                  ((truediv, lt),)  {}

ORIGINAL BYTECODE toy_example example.py 9
 10           0 LOAD_FAST                0 (a)
              2 LOAD_GLOBAL              0 (torch)
              4 LOAD_METHOD              1 (abs)
              6 LOAD_FAST                0 (a)
              8 CALL_METHOD              1
             10 LOAD_CONST               1 (1)
             12 BINARY_ADD
             14 BINARY_TRUE_DIVIDE
             16 STORE_FAST               2 (x)

 11          18 LOAD_FAST                1 (b)
             20 LOAD_METHOD              2 (sum)
             22 CALL_METHOD              0
             24 LOAD_CONST               2 (0)
             26 COMPARE_OP               0 (<)
             28 POP_JUMP_IF_FALSE       38

 12          30 LOAD_FAST                1 (b)
             32 LOAD_CONST               3 (-1)
             34 BINARY_MULTIPLY
             36 STORE_FAST               1 (b)

 13     >>   38 LOAD_FAST                2 (x)
             40 LOAD_FAST                1 (b)
             42 BINARY_MULTIPLY
             44 RETURN_VALUE

MODIFIED BYTECODE
  9           0 LOAD_GLOBAL              3 (__compiled_fn_0)
              2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 CALL_FUNCTION            2
              8 UNPACK_SEQUENCE          2
             10 STORE_FAST               2 (x)
             12 POP_JUMP_IF_FALSE       24
             14 LOAD_GLOBAL              4 (__resume_at_30_1)
             16 LOAD_FAST                1 (b)
             18 LOAD_FAST                2 (x)
             20 CALL_FUNCTION            2
             22 RETURN_VALUE
        >>   24 LOAD_GLOBAL              5 (__resume_at_38_2)
             26 LOAD_FAST                1 (b)
             28 LOAD_FAST                2 (x)
             30 CALL_FUNCTION            2
             32 RETURN_VALUE

GUARDS:
 - local 'a' TENSOR_MATCH
 - local 'b' TENSOR_MATCH
 - global 'torch' FUNCTION_MATCH
```

At the top you can see the FX graph (which we already shared above).
Next you see the original bytecode of the function, followed by the
modified bytecode generated by TorchDynamo.  Finally, you see the guards
which we covered above.

In the modified bytecode `__compiled_fn_0` is the return value
of `my_compiler()` (the compiled graph). `__resume_at_30_1` and
`__resume_at_38_2` are both generated continuation functions that pick up
execution after a graph break (at bytecode offsets 30 and 38).  Each of
these functions take the form:
```
__resume_at_<offset>:
    ... restore stack state if needed ...
    JUMP_ABSOLUTE <offset> into toy_example
    ... original bytecode of toy_example ...
```

By generating these resume_at function we force the remainder of the
function to be executed in a new Python frame which recursively will
trigger TorchDynamo to re-start its capture once execution reaches that
point for the first time.