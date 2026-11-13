# Gitpack — Writeup (Network Forensics / OAuth Phishing, 50 pts)

## Challenge Info
- **Category:** Network Forensics / OAuth Phishing
- **Flag format:** `flag{CLIENT_ID:ROUTE_ID:BACKEND_DOMAIN}`
- **Given:** Two packet capture (pcap) files — one small and focused (`capture_incident.pcap`), one large and noisy (`capture_network.pcap`)

## Challenge Summary
At a specific time, a victim's OAuth authorization token was stolen — no password was typed, and it happened quietly. The expected attack chain was:
```
notification email → fake GitHub Pages site → OAuth consent → callback → attacker's backend
```

## Step 1 — Read the Phishing Email (SMTP Traffic)
Filtered the incident capture for plaintext SMTP (email) traffic:
```bash
tshark -r capture_incident.pcap -Y "tcp.port==25" -T fields -e mail.from -e mail.subject
```
Found a realistic-looking "GitHub security alert" email. Its link pointed to a **fake GitHub Pages site** (a decoy) instead of the real GitHub domain.

## Step 2 — Trace the Victim's Connections (SNI)
Even though the actual traffic was encrypted with TLS, the **Server Name Indication (SNI)** field in the TLS handshake reveals which domain the victim was connecting to — this isn't encrypted:
```bash
tshark -r capture_incident.pcap -Y "tls.handshake.type==1" -T fields -e tls.handshake.extensions_server_name
```
This showed the victim's browser visiting, in order:
1. The decoy GitHub Pages site
2. The real `github.com` (for OAuth consent — no password needed since the victim was already logged in)
3. An attacker-controlled "callback" page (also hosted on GitHub Pages, to look trustworthy)

## Step 3 — Find the Actual Stolen Data (Plaintext HTTP)
The callback page forwarded the stolen OAuth data to the attacker's server over plain, **unencrypted HTTP**:
```bash
tshark -r capture_incident.pcap -Y "http.request" -T fields -e http.host -e http.request.uri
```
This request contained everything needed for the flag:
- `client_id=...` → the OAuth application's client ID
- The request path contained a unique state/route value, matching the `state` parameter → the route ID
- The `Host` header → the attacker's backend domain

## Step 4 — Confirm with the Larger Capture
Checked the bigger, noisier capture to confirm the same story and rule out a decoy:
```bash
tshark -r capture_network.pcap -Y "dns.qry.name contains \"<backend-domain>\""  -T fields -e dns.qry.name -e dns.a
```
This confirmed the backend domain's IP address, and showed the attacker's server later connecting to `github.com` itself — using the stolen token. A second, unrelated phishing email (which had already been quarantined by spam filters) turned out to be a **red herring**, unrelated to this specific attack chain.

## Attack Chain (Summary)
```
Phishing email → Fake GitHub Pages decoy → Real OAuth consent screen 
→ Attacker's callback page → Plaintext POST to attacker's backend → Token abused
```

## Flag
```
flag{Ov23li114b2bce64:bb83cfd6:gitc05b.servepkt.net}
```

## Lesson Learned
Even when a login page is 100% real (like the genuine GitHub OAuth screen), the overall flow can still be malicious if the *redirect* destination is attacker-controlled. Analysts should always check:
- The **SNI** field — reveals real destinations even over encrypted TLS
- Any **plaintext HTTP** traffic — often reveals exactly what data was stolen and where it went
