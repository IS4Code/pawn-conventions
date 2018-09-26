# Guidelines, conventions, and assumptions about the AMX API intended for consumers and extenders
This document shall function as a guide to plugin creators to ensure compatibility between various plugins which need to extend AMX API functions (via hooks) to work as intended. It is also useful to AMX API consumers as it specifies how to write code compatible with any implementation or extension of the AMX API (and how to write proper code in general).

# For AMX API consumers
## Stick to the AMX API whenever possible
Unless absolutely necessary, use the AMX API functions to operate with the AMX instance. Mainly, when your plugin only exports its API via native functions or callbacks, do not assume anything about the AMX instance pointer (`AMX*`) or the members of the structure, and treat it as an opaque pointer for use in the AMX API.

AMX API extenders need to rely on modifications to the AMX API, like changing its state or the way the code is executed. When the AMX API is bypassed, the extending plugins cannot ensure their correct functionality. The AMX API is a useful abstraction over its internals, and using it increases the compatibility of your plugins (with other plugins or future versions of AMX).

An example of a failure to rely on the AMX API can be found in SA-MP:
```pawn
cell* get_amxaddr(AMX *amx,cell amx_addr)
{
    return (cell *)(amx->base + (int)(((AMX_HEADER *)amx->base)->dat + amx_addr));
}
```
This function mimics `amx_GetAddr`, but returns the pointer directly. Even though it is easier to use, it lacks any input checks, which can lead to accessing invalid memory from Pawn. It also completely disregards the `AMX::data` member that can point to a separate data block of the AMX instance, making separating the code and the data impossible. Plugins that hook `amx_GetAddr` to extend the AMX address space will not work correctly in natives using `get_amxaddr`.

## Process error codes
Every AMX API function can fail. Even though there may be no indication of a possible failure in the sources, every AMX API function that returns an error code can fail. Always be prepared for any function to fail, and do not rely on values returned from a function that may fail without checking the error code.

Neglecting this point can lead to hard-to-detect bugs, like memory corruption. 

## Raise errors on failure
When a native function fails in a way that is unexpected or cannot be handled from Pawn, use `amx_RaiseError` to stop the execution of the script. This prevents any further code from manipulating incorrect values, and allows plugins like crashdetect to process the error and provide useful debugging information.

## Check the number of arguments
A registered native function can be called with any number of arguments, even from Pawn. If you expect a certain number of arguments, always check the actual number is not smaller. If a range of parameters has constant default values, be prepared to fill them in at runtime rather than relying on them being supplemented from Pawn.

Your plugin may be invoked from a script compiled for an older version of your API. When it is possible, ensure binary compatibility together with source compatibility.

Note that plugins that adapt native functions to other languages may remove sanity checks imposed on the parameters in Pawn. Be prepared for invalid values in the arguments.

## Ensure proper buffer size
Always make sure the output buffer can contain the data you copy there. Do not rely on any fixed size and use information from Pawn/AMX to dynamically calculate the size. Also keep in mind that if your native function is provided with a large string or array, using `alloca` (even indirectly via `amx_StrParam`) to allocate the buffer takes up the space from the stack, so if you expect very large strings or recursion, using conventional allocation is better (at least from a certain size) to avoid stack overflow.

When querying names from the AMX instance, use `amx_NameLength` every time to obtain the proper size of the output buffer.

## Handle packed strings
A packed string is an equally valid string compared to an unpacked one. You can work with character data directly via the pointer you obtain from `amx_GetAddr` (without `amx_GetString`/`amx_SetString`) but make sure you check if the string is packed or not. Since there is no function in the AMX API to check if a string is packed, and the layout of packed strings is not an implementation detail, you can compare the first cell in the string to `UNPACKEDMAX` (as `ucell`!). Larger values are guaranteed to be part of a packed string (since the first character in a packed string corresponds to the highest octet in the first cell).

## Do not rely on consistency
The state of the AMX instance may change at any time in the call to `amx_Exec`, or when you return from a native function. Be cautious when caching ids and names of public functions, native functions, public variables, or tags, since AMX API extenders may introduce features that allow modifying the corresponding tables dynamically.

If you need to cache these tables (for higher performance), always provide a fallback when the record cannot be found in your cache, and resolve to the AMX API in that case. Also check that the consistency is upheld if you rely on it.

For example, if you cache the public function index returned by `amx_FindPublic`, use `amx_GetPublic` to check that the public function is still present and has the same name, before calling `amx_Exec`.

## When relying on implementation details, ensure their presence
If you absolutely need to work with AMX implementation details (the AMX structure or the memory layout of the instance), always make sure you are obtaining correct information. Impose bounds checks on arbitrary values you get from the memory to prevent accessing invalid memory.

For example, `getarg` does not verify the memory it accesses in any way, and assumes it conforms to the AMX implementation:
```pawn
static cell AMX_NATIVE_CALL getarg(AMX *amx, cell *params)
{
    AMX_HEADER *hdr=(AMX_HEADER *)amx->base;
    uchar *data=amx->data ? amx->data : amx->base+(int)hdr->dat;
    // This function assumes that FRM is valid and points below the arguments passed to the function
    cell value= * (cell *)(data+(int)amx->frm+((int)params[1]+3)*sizeof(cell));
    value+=params[2]*sizeof(cell);
    // Also no attempt to check that the argument is actually passed by reference and the reference is valid
    value= * (cell *)(data+(int)value);
    // Aside from this, amx_GetAddr should be called on the value when it is first obtained, and the offset
    // should be added to the resulting pointer.
    return value;
}
```
When you are working with the AMX memory directly, always check that you read valid data from a valid block in it. Use `amx->hlw`, `amx->hea`, `amx->stk`, `amx->frm`, and `amx->stp` for checking the validity of addresses.

# For AMX API extenders
## Conform to the contract
When hooking functions in the AMX API, ensure it doesn't break their contract, as observed by consumers. All functions should behave as expected, and return proper values and proper error codes consistently.

When extending a function that can fail, try to minimize the impacts of consumer code not being able to handle the error. For example, `amx_FindPublic` assigns `INT_MAX` to the index if it cannot find the function. If the consumer uses the index (even though they shouldn't), further calls will fail as well, and no incorrect code is executed.

## Do not unregister hooks before unload
The order of registered hooks on a function is important. Hooks must be unregistered in the precisely opposite order they were registered, otherwise removing a hook can remove other hooks as well, or reapply removed ones (at best). Since it is almost impossible to ensure the order during standard execution, do not unregister hooks in other places than `Unload`.

If a plugin dynamically registers and unregisters hooks to various functions, it must be the only plugin that does so (for the given set of affected functions).

## Do not override other features
Features of your plugin should be only apparent when the consumer actually needs them, and should not make using features of other plugins impossible. For example, when you use the `AMX_ERR_SLEEP` code to execute code outside a native function, other plugins may use a similar mechanism as well, so use a way that allows you to identify your feature among the others (by providing a return code or marking the AMX).

## Try to resemble the AMX implementation
Despite these guidelines, you can always find code that relies on the AMX implementation details and breaks when used with your plugin. Take extra measures to ensure that you keep the AMX state consistent with what is expected according to the implementation.

For example, make sure you keep `amx->frm` valid, make `cell *params` actually point to the stack when calling native functions etc.

## Try to be consistent
Other plugins may take shortcuts to increase performance without using the AMX API. If a value you returned from a hooked function suddenly becomes invalid, be prepared for a case when another plugin uses it at any time in the future. If possible, try to return consistent values to minimize the risk of incorrect behaviour.

## Discard irregular values
Your hooks may be called from "weird" places in plugins that use the AMX API for their own purpose. When you cannot handle the call, don't, and call the original function instead. Chances are another plugin will be able to handle the call, or it will go to the base implementation and produce an error.
