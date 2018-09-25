# Name mangling
This document proposes a name mangling scheme for native functions exported to Pawn. Name mangling uses the standard AMX API mechanics to include additional information about a function signature, that is the types of arguments and the calling convention.
Functions decorated this way are imported like regular functions in Pawn, but use the mangled name in the native table.

## Reasons
### Compatibility
Using name mangling can prevent incorrect behaviour caused by changes in implementation of a native function without propagating the change to scripts that import the function. Consider the following function provided by the host:
```pawn
native SetTimer(const funcname[], interval, bool:repeating);
```
A script using the function assumes a signature of `(string, int, bool)`, but has no means to verify it. The host can decide to change the implementation and provide this function instead:
```pawn
native SetTimer(const funcname[], bool:repeating, interval);
```
Scripts compiled for a previous version of the API will initialize without errors, and unless the host has strict sanity checks on the arguments, there is nothing to indicate the use of incorrect function except for incorrect behaviour of the program.
Reordering parameters is usually not a good idea, but the host may extend the parameters in a way that cannot be sanitized at run-time:
```pawn
native SetTimer(const funcname[], Float:interval, bool:repeating);
```
Now there is nothing to indicate at run-time that `interval` was provided with an integer value, for older scripts, since the domains mostly overlap.

If the host provided a function with a name to reflect the formal types of parameters, using an older version of the function will report an error during initialization. Moreover, if the host decides to deprecate a certain function, it could provide it as an overload for backwards compatibility.
In this case, the host is able to provide three functions with a different mangled name:
```pawn
native SetTimer(const funcname[], interval, bool:repeating) = SetTimer@3sib;
native SetTimer(const funcname[], bool:repeating, interval) = SetTimer@3sbi;
native SetTimer(const funcname[], Float:interval, bool:repeating) = SetTimer@3sfb;
```
The mangled name is a valid Pawn identifier specifying the formal types of arguments (and the calling convention).

### Validation
AMX API extenders may decide to add support for dynamic languages or languages that provide metadata necessary to perform type checking of declarations, to be able to interact with the Pawn API. Consider a hypothetical way of using Pawn native functions in C# and Lua:
```csharp
[PawnImport]
int SetTimer(string funcname, int interval, bool repeating);
```
```lua
native.SetTimer("Func", 1000, true)
```
In both languages, the host can determine the correct function based on the types of parameters or arguments. If, by mistake, the mangled name of the requested function differs from the one provided, the host can handle this situation without calling an incorrect function (and in C#, this error can be reported early before the program starts).
If only the single definition of the function is provided, these both pieces of code fail:
```csharp
[PawnImport]
int SetTimer(string funcname, float interval, bool repeating);
```
```lua
native.SetTimer("Func", 1.5, true)
```

### Reflection
The host may provide extended information about any function which is encoded this way. This information can be used by AMX API extenders to automatically convert values to types expected by the function. If a more complex function needs to be called, this may be necessary for languages that do not provide a way to express this information.
```pawn
call_native("SetTimer", "Func", 1000, true);
```
`call_native` has no idea about the types of parameters it received, and thus it doesn't know if it should treat them as references or as values. If there is `SetTimer@3sib` provided, the function can use the type information to determine the types of provided arguments.
```lua
native.SetTimer("Func", 1000.0, true)
```
In Lua, this would probably call `SetTimer` with 1148846080, since 1000.0 would be converted to a cell containing its encoded value. If the mangled function name is available, the host can safely round 1000.0 to 1000 and call the function.

### Overloads
Although Pawn does not support overloaded functions, other languages may profit from mangled names, due to function overloading. In case of Pawn, multiple scripts can use different functions with the same name, without causing collisions, and in the case of other languages, the correct version of the function can be chosen based on the parameters specified in the language.

## Mangling scheme
### Standard calling convention
A mangled name for a function using the standard Pawn calling convention is suffixed with `@` and the encoded signature. The encoded string can contain only characters that can appear in a valid Pawn symbol, i.e. `a-z`, `A-Z`, `0-9`, `_`, and `@`. If the function contains the `@` character in its unmangled name, the first occurence of the character that is followed by a valid signature is chosen.

The signature starts with the number of fixed parameters, encoded as a non-negative decimal number (with no leading zeroes or non-digit characters). This number is used to verify that the signature is correct, based on the number of encoded parameters. If the function accepts no arguments, this must be `0`.

The number is followed by the fixed parameters, each encoded according to its type. The following table specifies how to encode the formal types of simple parameters:

|Type|Code|
|-|-|
|Signed integer|`i`|
|Unsigned integer|`u`|
|Boolean|`b`|
|Float|`f`|
|Character|`c`|
|Handle|`h`|
|String|`s`|
|Variant|`_`|

Parameters encoded in this way are always passed by value, with the exception of `s`. The length of the string (both packed or unpacked) must be determined by calling `amx_StrLen`, and the native function is prohibited from modifying the string. `c` corresponds to a valid cell in any string, or 0. `h` represents an opaque value whose integral value is irrelevant, and the only guarantee is that 0 is not a valid value. `_` permits an argument of any value type.

Arrays are encoded as `a` or `A`. `A` denotes an input array, i.e. an array which cannot be modified by the native function (interning the value is safe). The array is followed by its length specified as a decimal number and the element type, encoded in the same fashion. `0` can be used to represent an unbounded array, in which case the length is determined in another way.

References are encoded as `a1`, followed by the element type.

If the parameter accepts a value of a specific tag, `t` can be used to specify the tag. This is followed by the length of the tag name, followed by the tag name itself. It is possible to chain the tags in case of multiple permitted tags, but they must be sorted in the ascending order (based on the ASCII codes). Zero-length tag name represents an untagged cell. `t` must not be used if the only tag is `Float` or `bool`, or if the parameter is tagless.

Finally, if the default value of a parameter is a `sizeof` or a `tagof` expression, it can be encoded by using `L` or `T`, respectively, followed by the index of the referenced parameter (zero-based). Additionally, `L` can be chained to represent dimensions, e.g. `L0` is `sizeof(arg0)`, `LL0` is `sizeof(arg0[])` etc.

#### Variadic functions
A variadic function (with the `...` part) is encoded by specifying `x` after the parameters. This can be followed by tags like the `t` specifier, or by nothing, in which case a value of any tag is allowed.

#### Return value
Optionally, the signature can be followed by another `@` character and the encoded return type. If this part is not present, no assumptions about the returned value must be made.

### Optcall
Functions using the optcall calling convention take a special parameter that specifies the "default" value for missing arguments. The name of an optcall function must be followed by `@O` and optionally by the encoded signature of the base function (without the initial `@`). The special parameter is not represented in any way in the signature.

## Examples
```pawn
native SetTimer(const funcname[], interval, bool:repeating) = SetTimer@3sib@i;
native SetTimerEx(const funcname[], interval, bool:repeating, const format[], {Float,_}:...) = SetTimerEx@4sibsx05Float@i;
native Float:GetPVarFloat(playerid, const varname[]) = GetPVarFloat@2is@f;
native File:fopen(const name[], filemode:mode=io_readwrite) = fopen@2st8filemode@t4File;
native GetPlayerName(playerid, name[], len=sizeof(name)) = GetPlayerName@3ia0cL1@i;
native bool:GetPlayerHealth(playerid, &Float:health) = GetPlayerHealth@2ia1f@b;
```
