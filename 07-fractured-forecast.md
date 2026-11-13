# Fractured Forecast — Writeup (Cryptography, 392 pts)

## Challenge Info
- **Category:** Cryptography (custom block cipher + RNG prediction)
- **Given:** A remote service to connect to, plus two files: `chall.py` and `cipher.py`
- **Goal:** Break a custom encryption algorithm, use that to predict random numbers, and use *that* to figure out the AES key protecting the flag

## What the Server Does
1. Creates a secret 336-bit master key.
2. Builds a custom 7-round **Feistel cipher** (a specific style of block cipher) called `ForecastCipher` using that key.
3. Seeds two Python random number generators (**MT19937**, the algorithm behind Python's `random` module).
4. Generates 5632 bytes of output using those random generators.
5. The first 5600 bytes are given to us (encrypted) as a "leak". The last 32 bytes become the AES-256 key that encrypts the actual flag.

## What We Can Do
- Send our own chosen data to be encrypted (up to 8192 blocks total).
- Request the encrypted 5600-byte leak (after this, the connection closes).

**Overall goal:** recover the master key → decrypt the leak → use the leak to figure out the random generator's internal state → predict the AES key → decrypt the flag.

## Step 1 — Understand the Cipher's Structure
The 336-bit master key is split into 7 pieces of 48 bits each — one piece used per round. Each round shuffles the data using a "round function" made of three parts:
1. **Expansion** — stretches 32 bits into 48 bits using a fixed linear operation.
2. **S-boxes** — the expanded value is mixed with the round key, then passed through small substitution tables (this is where most of the cipher's security normally comes from).
3. **Linear mixing** — another fixed operation to spread the result around.

In round 1 specifically, there's an extra safety feature: a cryptographic hash (BLAKE2s) is mixed in, meant to make the cipher resistant to advanced mathematical attacks.

## Step 2 — Find the Weakness
Two small design flaws combined into a serious backdoor:

**Flaw 1:** The hash-based safety feature in round 1 ignores one specific bit (bit 3) of its input before hashing. So if two inputs differ *only* in that one bit, the hash output is identical for both — meaning the safety feature does nothing in that case.

**Flaw 2:** There's a specific fixed value (`delta`) that, after going through the cipher's "expansion" step, produces a difference in *exactly* that one ignored bit. In other words, this specific value sails right past the hash protection.

**Why this matters:** Using `delta` as a chosen difference between two inputs, we found that it also causes only **one** S-box (out of eight) to actually change anything — and for that particular S-box, there's an unusually strong rule: the same specific difference happens about 1-in-4 times, far more often than the normal expectation (~1-in-16) for a well-designed cipher. This kind of exploitable pattern is sometimes called a **differential characteristic**.

This pattern can repeat itself round after round — forming a predictable "trail" through several rounds of the cipher. Out of the 7 rounds, 3 of them cost nothing to pass through, and the other 4 each have roughly a 1-in-4 chance of continuing the pattern. Overall, with 8192 chosen plaintexts, this produces roughly 15 confirmable "good pairs" — i.e. input/output pairs where we know exactly what happened inside the cipher, without knowing the key yet. We can verify each good pair just by checking a simple relationship in the ciphertext (no guessing needed).

Since this attack only directly reveals information about 4 of the 7 round-key pieces, the remaining 3 needed extra, more careful analysis in later steps.

## Step 3 — Recover the Round Keys (Working Backwards)
Each round key is made of 8 small 6-bit pieces (one per S-box). Keys were recovered starting from the **last round and working backward** — this is a classic trick with Feistel ciphers, because the last round's inputs can be partly reconstructed directly from the output ciphertext.

- **Last round key:** recovered using the direct relationships from the good pairs, plus statistical counting for the trickier parts.
- **Next rounds:** each time, we "peel off" one more round using the key we already recovered, then repeat similar statistical analysis on what's left.
- **Trickiest rounds:** needed a special consistency check (nicknamed the "Psi-collapse test" in the original notes) — for the *correct* key guess, a certain calculated value stays exactly the same across all our data; for a *wrong* guess, it scatters into many different values. This made it easy to tell correct guesses from incorrect ones even when simple counting gave close results.
- **First two rounds:** by this point enough of the cipher was already "undone" that these keys could be found with simple, direct checks — every single test data point agreed.

Once all the round keys were recovered, they were combined into the full 336-bit master key, and double-checked by re-testing it against every single one of the 8192 captured plaintext/ciphertext pairs — all of them matched.

## Step 4 — Break the Random Number Generator
With the master key known, the 5600-byte "leak" could be decrypted. Each byte of the leak reveals information (as simple yes/no style equations) about the internal state of the two MT19937 random generators:
- 5600 bytes × 8 bits = 44,800 equations
- Unknowns: the full internal state of both generators (roughly 19,937 bits each)

Using **Gaussian elimination** (a standard method for solving large systems of linear equations, done here using binary/XOR math), the full internal state was solved — which let us predict the *next* 32 bytes the generator would produce. Those 32 bytes are exactly the AES key used to protect the flag.

## Step 5 — Decrypt the Flag
With the predicted AES key (plus the given nonce, ciphertext and authentication tag), the flag was decrypted using **AES-256-GCM**. A successfully verified authentication tag confirms the recovered key was correct.

## Flag
```
flag{the_forecast_was_linear_after_all1794348730mFnohVO04669076ea655ad7ce7de74f80860242ef772a263_133}
```

## Tools / Scripts Used
- `capture2.py` — connects to the server and captures all the chosen-plaintext data and the leak
- `solver3.py` — the main attack: recovers all the round keys
- `mt_solver.py` — solves the random generator's internal state using linear algebra
- `solve.py` — uses the recovered master key to solve the generator and decrypt the final flag

## Lessons Learned
1. **A hash-based safety feature isn't automatically secure** — always check whether it ignores any input bits, and whether an attacker can route around exactly those bits.
2. **Combining multiple "random-looking" linear operations can accidentally create a predictable pattern** — each piece may look fine alone, but their combination can leak structure.
3. **A cipher's substitution table (S-box) should never have an unusually strong difference pattern** — attackers actively search for exactly this kind of weakness, since it enables efficient key recovery.
4. **In Feistel ciphers, attacks are often easiest by working from the last round backward**, since the final round's inputs can be partly reconstructed from the ciphertext alone.
5. **A global consistency check can be more reliable than simple vote-counting** when results are close.
6. **Capture all your data from the live server immediately, then analyze it offline** — this avoids losing access to the challenge instance while you work.
