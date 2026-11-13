# Tornwall — Writeup (Forensics / Log Analysis / DNS Exfiltration, 200 pts)

## Challenge Info
- **Category:** Forensics / Log Analysis / DNS Exfiltration
- **Flag format:** `FLAG{...}`
- **Given:** An archive `artifacts_12.tgz`
- **Story:** The SOC said "It's clean, go home", but two attack stories and a "friendly marketing pixel" were hiding inside the resolver logs.
- **Hint:** *"Saltmarsh doesn't leak. Something on the inside is quietly disagreeing."*

## What Was Inside the Archive
- **`logs/`** — dnsmasq (about 290k lines), auth, conn, proxy_http, vpn, mail, nginx, firewall
- **`evidence/`** — ransom note, SOC notes, IR report, bash_history, cron, creds export, and more
- **`flag_checksum.txt`** — SHA-256 hash of the real flag

## Step 1 — Eliminate the Decoys
The `evidence/` folder contained **9 different `FLAG{...}` strings** (for example a base64 token in the ransom note, one in the SOC notes, one in a kdbx export, and one in espresso telemetry). Instead of guessing, I took the SHA-256 of every candidate and compared it with `flag_checksum.txt`.

**Result:** none of them matched, so all 9 were decoys (red herrings).

## Step 2 — Find the Real Lead
`SOC_triage_notes.txt` said that DNS lookups for `saltmarsh-cdn.net` were "ALL known-good, marketing tracker". The challenge description also mentioned a "marketing pixel". That was the SOC's blind spot, so that is where I looked.

In `dnsmasq.log`, only **8 lookups out of 290k lines** were for `saltmarsh-cdn.net`:
```
+1712340000 ... 10.42.7.31 -> QNAME of5xm4cm.saltmarsh-cdn.net  NXDOMAIN
+1712340090 ... 10.42.7.31 -> QNAME iqbvwq22.saltmarsh-cdn.net  NXDOMAIN
... (6 more)
```

**Red flags:**
- All queries came from the **same host** (`10.42.7.31`)
- They repeated every **~60–90 seconds** (a beacon-like pattern)
- All answers were **NXDOMAIN** (the domain isn't a real service)
- The subdomains were **random-looking 8-character strings**

## Step 3 — Decode the Subdomains (Base32)
The labels only used the characters `a-z` and `2-7`, which is exactly the **Base32 alphabet**. Joining the labels in chronological order gave:
```
of5xm4cmiqbvwq22ancuix3ilfhvgb22amdfs2cvaqbvib2znbje6bs3jilrofyx
```
That is 64 characters, which decodes to 40 bytes. After Base32 decoding the result still looked like gibberish:
```
q{vpL\x03[CZ\x03ED_hYOS...
```

## Step 4 — Find the XOR Key (Known Plaintext)
We know the flag starts with `FLAG{`, so we can XOR the first 5 bytes of the gibberish with that known text:
```
'q' ^ 'F' = 0x37
'{' ^ 'L' = 0x37
'v' ^ 'A' = 0x37
'p' ^ 'G' = 0x37
'L' ^ '{' = 0x37
```
Every result is `0x37`, so the key is a **single-byte XOR with 0x37**. The last 0x17 bytes decoded as `0x17 ^ 0x37 = 0x20`, which is just space padding.

## Step 5 — Solver
```python
import base64
s = "of5xm4cmiqbvwq22ancuix3ilfhvgb22amdfs2cvaqbvib2znbje6bs3jilrofyx".upper()
flag = bytes(b ^ 0x37 for b in base64.b32decode(s)).decode().strip()
print(flag)
```

## Step 6 — Verify
```
sha256(flag) = 2ebacfc7d89bb14e0d6eb45f3e3e8c092696f43bffeac78070a81bce13c34c78
```
This **exactly matches** `flag_checksum.txt`, so the flag is confirmed.

## Attack Chain (Summary)
```
Infected host → tiny DNS lookups to a "marketing" domain → data hidden in the subdomain
→ NXDOMAIN replies (ignored by SOC) → Base32 + XOR 0x37 → flag
```

## Flag
```
FLAG{s4ltm4rsh_nxd0m41n_b34c0n_ex1l}
```

## Tools Used
`tar`, `grep`, `awk`, `sha256sum`, `base32`, Python

## Lessons Learned
1. **DNS exfiltration hides data inside subdomains.** NXDOMAIN only means "no answer was found", but the data already reached the attacker's DNS server inside the query.
2. **Detection signs:** low volume, regular intervals, random-looking labels, and one single host.
3. **Known-plaintext XOR is easy to break.** If you know the flag format (`FLAG{`), a single-byte XOR key falls out immediately.
4. **Whitelisting is risky.** The SOC marked the domain "known-good" and ignored it, and the attacker exploited exactly that trust.
5. **Always verify decoys with a checksum.** When a checksum is given, test every candidate against it first so you don't waste time on the wrong path.

## Roman Urdu Revision Note
Chor ne chowkidar (SOC) ko bola "main to bas delivery wala hoon" (marketing pixel). Phir wo roz galat number par call karta raha (NXDOMAIN), aur har call ke number mein hi paigham chhupa tha. Chowkidar ne sirf ye dekha ke "call connect nahi hui", lekin number (domain name) ko parha nahi. Humne numbers jode (base32), taala kholi (XOR 0x37), aur flag mil gaya.
