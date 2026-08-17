# XorGate Crackme

| | |
|---|---|
| **Source** | [crackmes.one](https://crackmes.one/crackme/6a768ab608712c1a17cbacdd) |
| **Difficulty** | 1.4 / 6.0 |
| **Platform** | Linux (ELF) |

## Steps

```bash
file XorGate
rabin2 -z XorGate      # flag string IS visible plain, but only gets printed
                        # AFTER validation passes -> not the actual challenge
r2 -A XorGate
afl                     # only main found, no separate check()/xor() function
pdf @ main
```

## Analysis

Unlike the muhemed crackme (a fixed password hidden as hex on the stack), the
password here is **not constant** — it is derived from the username the user
types in. The challenge is to reverse the derivation formula, not to find a
single hardcoded string.

The program reads two inputs, `Username` and `Password`, then builds an
**expected password** in three steps:

1. **XOR each byte of the username** with a single-byte key `0x23`, found in
   the disassembly as:
   ```asm
   mov byte [var_456h], 0x23   ; key = 0x23
   ...
   movzx eax, byte [rax]       ; username[i]
   xor al, byte [var_456h]     ; username[i] ^ 0x23
   ```
2. **Hex-encode each XOR result** as a 2-digit lowercase hex string via
   `snprintf(..., "%02x", ...)`. E.g. a XOR result of `0x6b` becomes the text
   `"6b"` (two ASCII characters), not the raw byte `0x6b`.
3. **Append the constant string `"@password"`** to the end. This string is
   stored on the stack the same way the muhemed crackme stored its password
   — as a little-endian hex literal loaded via `movabs`:
   ```asm
   movabs rax, 0x726f777373617040   ; little-endian -> "@passwor"
   mov word [var_412h], 0x64        ; + "d"
   ```

The resulting string is then compared against the user's password input —
first by length, then byte-by-byte — before granting access.

```
expected_password = hex_lowercase(byte(c) XOR 0x23 for each c in username) + "@password"
```

### A note on the disassembly

The binary reuses a single large stack buffer for two different roles: bytes
`0–255` hold the user's password input, and bytes `256+` hold the
program-computed expected password. r2 assigns these two regions different
local variable names (`var_310h`, `size`, ...) even though they are the same
physical array offset by 256 bytes. This is a good reminder not to trust
auto-generated variable names — always check actual addresses/offsets, since
a binary can deliberately reuse memory to make analysis harder.

## Example

Username: `Hi`

| char | byte (hex) | XOR 0x23 | hex-encoded |
|------|-----------|----------|-------------|
| `H`  | `0x48`    | `0x6b`   | `6b`        |
| `i`  | `0x69`    | `0x4a`   | `4a`        |

Expected password: `6b4a` + `@password` = **`6b4a@password`**

```
$ ./XorGate
Welcome to SoulReaper Crackme
Username: Hi
Password: 6b4a@password

[+] Access granted!
[+] FLAG{SoulReaper_XOR_Crackme}
```

## Verification

The derived formula was cross-checked through two independent paths:

1. **Native binary** (WSL2, Linux ELF) — ran `./XorGate` directly with
   `Username: Hi` / `Password: 6b4a@password` -> `Access granted!`
2. **Reconstructed C source** (credit: `#rvze`, see Credits below) —
   recompiled with `gcc test.c -o test` on Windows, same input/output ->
   confirms the disassembly-derived formula matches the original source
   logic exactly.

## Methodology note

Instead of reading the full `pdf @ main` output line by line, targeted
`grep` searches were used to jump straight to the relevant instructions:

- `grep "xor"` -> filter out the `xor reg,reg` zeroing idiom (e.g.
  `xor eax, eax`, used by compilers to set a register to 0 — not encryption)
  from real data transformations like `xor al, [var]`
- `grep "movabs"` -> locate hidden string constants loaded as little-endian
  hex literals
- `grep "cmp"` -> locate the length check and the byte-by-byte comparison
  loop

**Tip:** `r2` outputs ANSI color codes by default, which insert hidden
characters *inside* colored words (e.g. between `xor` and `al`). This
silently breaks `grep` matches like `"xor al"` even though the text looks
correct on screen. Disable color before piping to grep:
```bash
r2 -q -e scr.color=0 -c 'aaa; pdf @ main' XorGate | grep -n "xor"
```

## Credits

The derivation above was worked out independently from `radare2`
disassembly. It was then cross-verified against a community C
reconstruction of this crackme by **`#rvze`** (Telegram:
[@internetrvze](https://t.me/internetrvze)), used solely for verification —
not as the source of the original analysis.

## Category

XOR-based dynamic password derivation (single-byte key `0x23`) + hex
encoding. Unlike a fixed hardcoded password, the correct password is a
function of user-controlled input: `f(username) = password`. Any valid
username has a corresponding correct password that can be computed with the
formula above.
