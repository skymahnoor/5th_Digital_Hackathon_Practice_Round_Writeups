# Jackpot — Writeup (Forensics, 120 pts)

## Challenge Info
- **Category:** Digital Forensics (ATM "jackpotting" incident)
- **Flag format:** `flag{SERIAL:IP:PORT}`
- **Given:** An archive containing 5 evidence folders from a compromised ATM

## Step 1 — Extract the Evidence
```bash
tar xzf artifacts_11.tgz
```
This produced 5 folders:
- **E1** — ATM host logs
- **E2** — Cash dispenser capture
- **E3** — EPP (PIN pad) audit logs
- **E4** — Network uplink packet capture
- **E5** — SD card dump

## Step 2 — Check the Approved Device List
`E1_atm_host/DISPENSER/approved_peripherals.txt` listed only 3 approved hardware serial numbers (card reader, PIN pad, and dispenser controller). Anything else connected to the ATM would be unauthorized.

## Step 3 — Find the Rogue USB Device
Looking at `E1_atm_host/USB/setupapi.dev.log` (Windows USB install log), a new device was installed at a specific timestamp:
```
USBSTOR\Disk&Ven_Kingst&Prod_DataTrav\SER-978C4D81
```
This serial number is **not** in the approved list, and it's a USB flash drive — not a legitimate ATM peripheral at all. This is our unauthorized device → **SERIAL = SER-978C4D81**.

## Step 4 — Confirm with the Dispenser Journal
`DISPENSER/dispenser_journal.csv` showed two cash dispenses happening just after the rogue device was installed, tagged with a session ID matching `978C4D81` — different from the normal session ID format used by legitimate transactions. This confirms the rogue device triggered real cash withdrawals.

## Step 5 — Find the Suspicious Network Connection
`NET/firewall.log` showed the ATM normally only talks to one trusted processing center IP and port. But right around the time of the rogue dispenses, there was **one connection to a different IP and port** — just 2 seconds before the cash was dispensed.

## Step 6 — Verify with the Packet Capture
Parsed `E4_uplink/uplink_48h.pcap` with Python and found:
- A DNS lookup resolving to that same suspicious IP
- A TLS certificate for that server that was created suspiciously recently (compared to the real processing center's much older certificate)
- Commands (`auth`, `present`, `dispense`) in that session tagged with the same `978c4d81` ID, and the dispensed amounts matched the journal exactly

This confirms the full picture: the attacker plugged in an unauthorized USB device, which connected the ATM to a fake/rogue server, which then issued commands to dispense cash — a classic **ATM jackpotting attack**.

## Flag
```
flag{SER-978C4D81:203.0.113.111:15153}
```

## Lesson Learned
In forensic investigations, cross-referencing multiple independent evidence sources (device logs, transaction journals, firewall logs, and packet captures) is key — each source alone gives a clue, but together they confirm the full attack story with confidence.
