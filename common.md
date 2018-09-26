# Common conventions
## String arguments
In case a string provided to native functions must contain the null character somewhere in the middle, use `0xFFFF00` in its place (in an unpacked string). `amx_StrLen` and `amx_GetString` will skip the character and store the null in the output buffer, respectively (any value with the middle two octets non-zero will do, but some extension functions may interpret as extension-specific metadata).

`amx_StrLen` should be used to obtain the correct length of the string (packed or unpacked). Plugins may hook this function to decrease its complexity or return the true length of the string for a set of known strings. Plugins should rely on the result from this function and not on `strlen`.
