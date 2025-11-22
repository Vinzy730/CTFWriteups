# COMSCI214 CTF Writeups

This file documents my solutions for several COMSCI214 CTF challenges.

---

## Huge RSA

This challenge is a standard RSA decryption problem: we’re given `n`, `e = 65537`, and a ciphertext `C`, and we need to recover the plaintext flag.

Since this is RSA, I first need to factor `n` into its two prime factors. I used [FactorDB](https://factordb.com/) to do this.

**Given:**

- Modulus `n`:

    134483489259357587193062297294916557281511310492658874099915118851106176841242629412489233879805646303697723631290376922707848149746292340948693757862129548348543497180869087021892384409416014298517539741882667796027036147608670091266898895218841621334695007993716583675755016319734116593324017785790343763209

- Factor 1:

    10738351774266087351282102063418344749874815319835209296472309964567481804556694272269779874180563127835865325832141405899276565174505408178103709685731531

- Factor 2:

    12523662111874600161833138357230746871540990567053354502660401326710160541408793753329772363924830742900510940368962338277549788111438397117750739543628539

To decrypt RSA once we have the prime factors, we compute **Carmichael’s totient function** λ(n), which is:

- λ(n) = lcm(p − 1, q − 1)

So:

- λ(n):

    22413914876559597865510382882486092880251885082109812349985853141851029473540438235414872313300941050616287271881729487117974691624382056824782292977021587514421601840030262317775327293387398813785275475553144777219290811661054020963145549277767252656804045269575063762001806582230138441586453655223519067190

Next we compute the **private exponent** `d`, which is the modular multiplicative inverse of `e` modulo λ(n):

- `d = e⁻¹ mod λ(n)` with `e = 65537`

Result:

- `d`:

    18299606989945746723070388588021779632019128356324667430774875933610373285941197013325962019436249031773283535355229865071966550570606080756269376845468881443065087655133729737967933190217427534510409919136092857388537672758106375019836564157735208934000855245726123208468966611156873485114765334544529574563

Now the decryption equation is:

- `M = C^d mod n`

Where:

- Ciphertext `C`:

    102687950593631611439795983752527792474757459249112359566891231885071300563957355572372854325640742460221240091756824050729982284987694623185002468120085229505133037450786304818981737812221292836925419605029408917407642417732585756645359207207411677906338212716279539545603373179299650287710970218685927463998

So:

- `M = C^d mod n` =

    2845553777439546892329853931531422784917202432521245016403176235045790177086427562216425036289231708373840069289872582118914492911796974571413747437034221

Finally, convert this large integer to bytes and then to a string:

- Use `Crypto.Util.number.long_to_bytes(M)`  
- Then `.decode()` to get ASCII.

This yields the flag:

> `COMSCI214{B1G_but_f4ct0rDB_c4n_h3lP_m3_S0lV3}`

---

## Password Attack

We’re given an encrypted `.docx` file and a hint to use **Hashcat**. The plan is:

1. Convert `encrypted_flag.docx` into a hash with `office2john.py` from the John the Ripper suite.
2. Crack the hash with Hashcat using the `rockyou.txt` wordlist.

**Step 1 – Extract hash**

I used `office2john.py`:

- This produced a line like:

    encrypted_flag.docx:$office$*2013*...

I cleaned the output and kept only the hash portion in `hash.txt`.

**Step 2 – Crack with Hashcat**

Hashcat command:

- For some reason, although it’s a 2013 file, I had to use mode `9600` instead of `9700`:

    hashcat -m 9600 -a 0 hash.txt rockyou.txt

Hashcat recovered the password:

- `COMSCI214711923`

I used this password to open the document (I uploaded to Google Drive and opened it there) and got the flag:

> `COMSCI214{greycat>hashcat_provemewrong}`

---

## Plaintext Attack

We’re given a ZIP archive `sus_package.zip` containing two files:

- `Assignment 1.pdf`
- `flag.txt`

We already have a copy of `Assignment 1.pdf`, so this is a classic **known-plaintext attack** on ZIP encryption.

I used **bkcrack** to exploit this.

**Step 1 – Inspect contents**

    ./bkcrack -L sus_package.zip

This confirms the two files in the archive.

**Step 2 – Recover keys using known plaintext**

We know the encrypted `Assignment 1.pdf` and its plaintext version, so:

    ./bkcrack -C ../sus_package.zip -c "Assignment 1.pdf" -p "../Assignment 1.pdf"

This recovers the ZIP encryption keys:

- Keys: `7e827b05 98ea3b23 33a5bfc8`

**Step 3 – Decrypt archive with keys**

    ./bkcrack -C ../sus_package.zip -k 7e827b05 98ea3b23 33a5bfc8 -D no_password.zip

This produces `no_password.zip`, which can be unzipped without a password.

I then unzipped and read `flag.txt`, which contained:

> `COMSCI214{but_what_if_zipcrypto_deflate_is_used_hmm...}`

---

## Secret Message

This challenge is a **monoalphabetic substitution cipher**. The core idea is to use letter frequency and patterns (frequency analysis / “HOR” style reasoning) to recover the substitution, with a focus on the flag.

Given ciphertext snippet (flag-shaped):

- `JFPYJB214{QO4SDY1Y_GRTCV3OE1D_QM56J}`

We’re told this is in the `COMSCI214{...}` flag format, so I started by mapping likely parts:

- `JFPYJB214` → `COMSCI214`  
  So `J → C`, `F → O`, `P → M`, `Y → S`, `B → C`, `J → I`.

Through frequency and pattern analysis:

- I noticed `QOZ` appears often → looks like `"AND"`  
  So `Q → A`, `O → N`, `Z → D`.

After some substitutions, I got:

- `COMSCI214{AN4SDS1S_GRTCV3OE1D_AM56C}`

Continuing with frequency and common English patterns:

- `QE` appears a lot → likely `"AT"`  
  So `E → T`.
- A common pattern `EKT` (now `T?E`) suggests `"THE"`:
  - `K → H`, `T → E`.

After more substitutions and using context from the paragraph (e.g., `"THE SECRET"`, `"SITS NEAR THE END OF THIS..."`), I gradually filled in:

- `COMSCI214{AN4SDS1S_FRECU3NT1D_AM56C}`

From here, I recognized leetspeak:

- `AN4SDS1S` → `AN4SYS1S` → `"ANALYSIS"`  
  This gives:
  - `D → Y` (from ANALYSIS spelling)
- `FRECU3NT1D` → `FREQU3NT1Y` → `"FREQUENTLY"`  
  This gives:
  - `C → Q`
- `AM56C` was the last bit; after testing plausible flags and adjusting:
  - `C → B` to match `"AB56C"` → `"AB5BC"` etc., and eventually got the correct mapping by trial on the CTF site.

Final decrypted flag:

> `COMSCI214{AN4SYS1S_FREQU3NT1Y_AB56C}`

(The challenge is about analyzing and frequently “ab”uses/leets letters.)

---

## Rolling Thunder

The code for this challenge implements a **rolling XOR** encryption:

- Core line:

    data[i] ^= KEY[i % len(KEY)]

Each byte of the original image is XORed with the corresponding byte of the key, and the key repeats.

To **decrypt**, we simply XOR the encrypted data with the same key again.

We’re told the file is a PNG, and we know that all PNG files start with the same 8-byte signature:

- `89 50 4E 47 0D 0A 1A 0A`

**Step 1 – Recover the key**

I opened the encrypted PNG (`flag.png.ec`) in a hex editor and took the first 8 bytes of the encrypted file.

Let:

- `enc[0..7]` = first 8 encrypted bytes
- `sig[0..7]` = PNG signature bytes

Then:

- `KEY[i] = enc[i] XOR sig[i]` for i = 0..7

Doing this gave:

- Key bytes (hex): `65 6c 65 70 68 61 6e 74`

Which in ASCII is:

- `"elephant"`

So the XOR key is the string `"elephant"`.

**Step 2 – Decrypt the file**

I modified the given Python script to:

- Read `flag.png.ec` in binary.
- XOR each byte with `"elephant"` (repeating).
- Write the result to `new_flag.png`.

Pseudo-code:

    key = b"elephant"
    data = bytearray(open("flag.png.ec", "rb").read())
    for i in range(len(data)):
        data[i] ^= key[i % len(key)]
    with open("new_flag.png", "wb") as g:
        g.write(data)

Opening `new_flag.png` reveals the flag:

> `COMSCI214{r0LL1ng_x0R_k4y_98bfa}`

---

## PDF Collisions?!

We need to create **two PDFs** that:

- Are visually identical to `Assignment 1.pdf`.
- Have the **same MD5 hash**.
- Have **different byte prefixes**.

This is a classic **MD5 collision** attack.

From the given `app.py` (in the provided zip), we can see:

- The site uses `diff_pdf_visually` to ensure all three PDFs (the master and the two we submit) are visually the same.
- Then it checks that the PDFs have the **same MD5** but **different bytes** at the start (different prefixes).

The strategy:

1. Find or generate two different binary prefixes `P` and `P'` that have the **same MD5 hash**.
2. Prepend them to the same PDF body (`Assignment 1.pdf`) to get two PDFs:
   - `a.pdf = P || Assignment 1.pdf`
   - `b.pdf = P' || Assignment 1.pdf`
3. These will:
   - Look visually identical (same PDF body).
   - Have the same MD5 hash (because `P` and `P'` collide).
   - Have different leading bytes (different prefixes).

### Initial Attempts – HashClash

I tried using **HashClash** and its chosen-prefix collision scripts:

- Command form I tried:

    ./cpc.sh P.bin Pprime.bin out1.bin out2.bin

I guessed that `P.bin`/`Pprime.bin` were input prefixes and `out1.bin`/`out2.bin` were the outputs. The script kept stopping after ~20 attempts, and I got nowhere.

I also attempted the 2-argument version:

    ./cpc.sh P.bin Pprime.bin

Still no success after long runs.

I tried using 64-byte prefixes (since MD5 digests 512-bit blocks), changing only 1 byte between `P` and `P'`, and padded them properly. I tested with:

    md5sum P.bin Pprime.bin

Since the hashes never matched, no collision.

### Using Known MD5 Colliding Prefixes

Instead of generating my own, I looked up known MD5 collision examples online. I found a documented pair of colliding prefixes (in hex) and used them as `P` and `P'`.

Let `h1` and `h2` be the two hex strings:

```text
h1 = (
"d131dd02c5e6eec4693d9a0698aff95c"
"2fcab58712467eab4004583eb8fb7f89"
"55ad340609f4b30283e488832571415a"
"085125e8f7cdc99fd91dbdf280373c5b"
"d8823e3156348f5bae6dacd436c919c6"
"dd53e2b487da03fd02396306d248cda0"
"e99f33420f577ee8ce54b67080a80d1e"
"c69821bcb6a8839396f9652b6ff72a70"
)

h2 = (
"d131dd02c5e6eec4693d9a0698aff95c"
"2fcab50712467eab4004583eb8fb7f89"
"55ad340609f4b30283e4888325f1415a"
"085125e8f7cdc99fd91dbd7280373c5b"
"d8823e3156348f5bae6dacd436c919c6"
"dd53e23487da03fd02396306d248cda0"
"e99f33420f577ee8ce54b67080280d1e"
"c69821bcb6a8839396f965ab6ff72a70"
)

open("P.bin","wb").write(bytes.fromhex(h1))
open("Pprime.bin","wb").write(bytes.fromhex(h2))

Finally they passed the md5sum check; so then I prepended them to the Assignment 1.pdf file by using the following commands:

cat P.bin "Assignment 1.pdf" > a.pdf
cat Pprime.bin "Assignment 1.pdf" > b.pdf

Which still passes the md5sum command check, so I had to check that the initial bytes were different so I ran the following command:

cmp -l a.pdf b.pdf | head
Which did give me different bytes, so I submitted them to the website and got the following flag: COMSCI214{no_md5_for_you!!!}