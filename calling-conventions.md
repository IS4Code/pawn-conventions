# Calling conventions
This document describes various calling conventions that are used for communication with the AMX host.

## Standard AMX calling convention (AMX-call)
A native function with this calling convention has this C signature:
```c
cell AMX_NATIVE_CALL f(AMX *amx, cell *params);
```
The used types and definitions are platform-specific and are defined by the AMX SDK.

`amx` is an opaque pointer to the AMX machine that executed the call. Its fields should not be directly accessed, and all communication with the AMX should be performed using the AMX API. `params` contains the number of all arguments that are provided to the function.

`params[0]` is the total number of bytes of arguments that were provided to the function. It is always a multiple of `sizeof(cell)`. The first argument is located at `params[1]`, the last is at `params[params[0]/sizeof(cell)]`.
The function should not assume anything about the state of the AMX machine, or its memory layout. Specifically, it must be prepared for a case where `params` is not located on the stack of the AMX machine.

The elements of `params` must not be modified.

The function can either exit successfully, returning a cell value, or with a call to `amx_RaiseError` providing a non-zero error code. The function should check if the number of non-optional parameters is not larger than the number of provided arguments. For optional parameters with a constant default value, the value should be taken instead of the parameter, if its value is not provided in the call. If the function cannot provide default values for missing parameters, it should fail with an error.

## Standard Pawn calling convention (Pawn-call)
This convention derives from AMX-call, specifying a Pawn call site. In this case, the types stay the same, but further assumptions about them can be made. First, `params` is located in an AMX stack frame which can be identified by inspecting the fields of `amx`, as they contain meaningful values in this case.

The contents of `params` may be modified by the function, but the fields of `amx` may not (only indirectly via calls to the AMX API).

## Optional arguments calling convention (optcall)
The optcall convention derives from AMX-call, intended to be used for languages that allow specifying a "missing" argument without having to provide the value from the declaration of the function.

In this case, the first parameter (`params[1]`), denoted `nil`, is a special value that is guaranteed not to be present in any argument whose value is directly specified. Since the caller has full access to the arguments, it can choose a value for `nil` (preferably `cellmin`) that is distinct from any provided argument, and use this one to represent missing values. `nil` is not a fixed value.

`params[0]` still contains the total number of bytes provided to the function, counting the `nil` parameter. The number may be less than the total number of parameters, in which case the rest is filled with their default values as if they were `nil`.

`nil` is in effect only for the elements inside `params`. Elements in variables or arrays provided to the function indirectly (via an address) are not considered. Reference parameters contain the `nil` value directly, if they are unspecified.

Similarly to Pawn-call, the function is allowed to modify the contents of `params`, but only the arguments that were originally `nil` (and not the actual `nil` parameter). This guarantees that a value the caller may rely upon is never modified.

Since Pawn doesn't directly support this calling convention, a Pawn-like language is used as an example, together with a standard Pawn fallback:
```pawn
native SetOptions(option1 = _, option2 = _, option3 = _); // default values not provided at compile-time
// native SetOptions@o(nil, option1, option2, option3);

SetOptions(10, _, 12);
// SetOptions(cellmin, 10, cellmin, 12);
SetOptions(_, _, cellmin);
// SetOptions(cellmin+1, cellmin+1, cellmin+1, cellmin);
```
