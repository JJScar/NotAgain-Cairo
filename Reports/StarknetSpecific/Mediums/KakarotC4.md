# Kakarot Contest on C4
## [M-1] `RIPEMD160` precompile crashes with a Cairo exception for some input lengths
### TL;DR
This is sort of a Starknet Specific and DoS blend of a vulnerability. To keep it short and easy, due to a mismatch of updating a variable the precompiler will crash under specific conditions. 

**What to look for in the codebase?**<br>
When updating a variable, wether it's conducting some calculations, changing it to another value, make sure the code takes it into account across the codebase where this variable is used. 

### Vulnerability Description
When the `RIPEMD160` precompile is called with specific input lengths, it triggers a Cairo exception. This is due to a mismatch in dictionary squashing where two pointers (`start` and `x`) point to different segments, causing an invalid squash.

### Code Sample
```rust
let (x) = default_dict_new(0);
tempvar start = x;
// ...
let (x) = default_dict_new(0); // reinitializes x but start still points to the old one
// ...
default_dict_finalize(start, x, 0); // mismatched segments, leads to crash
```

### Impact
Any L2 contracts depending on this precompile for `RIPEMD-160` hashing may crash or revert under specific conditions, breaking user functionality and exposing apps to denial of service (DoS) scenarios.

### Recommendation
Ensure that start points to the correct dictionary or finalize the correct dictionary before reinitializing. Also, improve testing coverage by adding end-to-end tests for precompiles to catch such bugs.

### References
- Cairo docs on [default_dict_finalize](https://docs.cairo-lang.org/cairozero/reference/common_library.html) (under default_dict_finalize).
- Finding on [Solodit](https://solodit.cyfrin.io/issues/m-01-ripemd160-precompile-crashes-with-a-cairo-exception-for-some-input-lengths-code4rena-kakarot-kakarot-git)