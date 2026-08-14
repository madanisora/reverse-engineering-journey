# muhemed crackme

| | |
|---|---|
| **Source** | [crackmes.one](https://crackmes.one) |
| **Difficulty** | 1.0 / 6.0 |
| **Platform** | Linux (ELF) |

## Steps

```bash
file crackme
rabin2 -z crackme    # no password found -> data hiding, not a plain string
r2 crackme
aaa
afl                    # found dbg.main and sym.imp.strcmp
pdf @ dbg.main
```

The password is constructed from 3 chunks of 64-bit hex on the stack, compared directly
with `strcmp` against the user input.

## Password

```
wvohXN8X7C14jrq1F*!j
```

## Category

Data hiding without encryption (not XOR/hash).
