# Receipt Vault — Writeup (Mobile, 50 pts)

## Challenge Info
- **Category:** Mobile / Reverse Engineering
- **Given:** An APK file (`mobile-receipt_vault_tar.gz`) and a server URL
- **Goal:** Get the app to "approve" a receipt and return the flag

## The Vulnerability
The app's visible screen only let you *view* a receipt — there was no "approve" button. But hidden inside the APK was an activity called `StaffApprovalActivity`, which could be triggered by a deep link (`dch://staff/approve`). So the approve feature existed, it was just hidden from the UI. This is called **security by obscurity** — hiding a feature instead of actually protecting it.

The real problem was worse: to approve a receipt, the app needed to send a cryptographic proof (HMAC-SHA256). The **secret key used to create that proof was hardcoded inside the APK**. Anyone who decompiles the app can read that key and generate valid proofs themselves — without even running the app.

## Steps
1. Decompiled the APK using **Androguard**.
2. Found the app's logic:
   - `GET /api/v1/challenge` → server sends back a `receipt_id` and a `nonce`
   - `proof = HMAC-SHA256(key, receipt_id + ":" + nonce)`
   - `POST /api/v1/approve` with `{receipt_id, nonce, proof}` → server returns the flag
3. Found the HMAC key hardcoded in the decompiled code.
4. Wrote a small Python script to repeat this flow manually (no app needed, just direct HTTP calls).
5. Ran the script — the server accepted the proof and returned the flag.

## Flag
```
flag{f4k3_fl4g_f0r_t3st1ng1794125992fifgBAud41a398b79578775b9bb7d1e6ee6c18da3590b630_133}
```

## Lesson Learned
Never store secrets (keys, tokens, passwords) on the client side (apps, browsers, etc.). Anyone can reverse-engineer the app and extract them. Servers should verify a client using something the client does **not** have access to — such as a server-side session or device attestation — never a static key baked into the app itself.
