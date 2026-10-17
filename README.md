# FortiGate Log Analyzer

A single-file Windows GUI (HTA) for triaging **FortiGate / FortiOS logs exactly as they come off the appliance** - the Log & Report "Download" files from the GUI, syslog captures, or FortiAnalyzer raw exports (key=value lines, plain or `.gz`). Point it at a folder of mixed log types and it answers the questions an incident responder asks of a firewall:

- **Who logged in to it?** Every admin login attempt per source and username, each successful session, and whether the management interface is reachable from the Internet.
- **What changed?** Every configuration change (path, object, old -> new attributes, who, from where), with admin-account, VIP, policy and logging changes flagged.
- **Who came in over VPN?** SSL-VPN logins per user and source country, password guessing, and IPsec peers (including Internet IKE scanning).
- **What was exposed, and what left?** Inbound sessions accepted from the Internet, internal admin-port fan-out (lateral movement), the largest uploads, and Internet destinations by bytes out.
- **What did the security profiles catch?** Antivirus, IPS, application control (remote-access tools flagged), web filter and SSL inspection.

Part of the DFIR toolkit family (Artifact Finder, the EZ-tool wrappers, Hayabusa, Hawk, UAC, vSphere and Data Domain triage).

## How it works

The `.hta` is the launcher and the viewer. A PowerShell engine is embedded in it; on **Process** it is written next to the output and run. The engine's C# core (compiled in-process with Add-Type) streams every line once - roughly **15 seconds per GB** - and writes:

| Output | What it is |
|---|---|
| `<host>-FortiGate-<stamp>.json` | Every table as a preview of up to 6000 rows, with true totals and full-data outcome counts - what the viewer loads. Priority rows (allowed malware, risky config changes, admin logins from the Internet, accepted inbound sources...) are always kept first. |
| `<stamp>_<table>.csv` | Every table in full (Open full CSV in the viewer). Event lists stream to disk while the logs are read, so nothing is dropped. |
| `runinfo.json` | Family provenance entry (app, host, input, files, triage summary) |

Output goes to `_Processed\<host>\FortiGate-Log-Analyzer\` next to the app. Flag rules and findings live in the viewer, so they can change without reprocessing.

## Manual

The full field manual - getting logs off the FortiGate, every tab, the flag reference, a worked example and troubleshooting, illustrated with synthetic data - is [docs/FortiGate-Log-Analyzer-Manual.html](docs/FortiGate-Log-Analyzer-Manual.html) (one self-contained HTML file; download it and open it in a browser). The app's **Instructions** button downloads and opens the latest copy.

## Quick start

1. Download `FortiGate-Log-Analyzer.hta` from the latest release (right-click > Properties > Unblock if Windows marks it).
2. Run it, pick the folder of exported logs (every file is sniffed; non-FortiGate files are skipped) or one log file.
3. Check **Target hostname** (filled from `devname=` when the logs carry it), then **Process -> analyze**.
4. Start on **Overview**: the findings list is generated from every table - click a finding to open the table behind it.
5. Narrow with the filters: **Pivot** on an IP / CIDR / user / host across every tab (each tab shows how many rows match), **High only**, **Internet IPs only**, **Allowed / succeeded only**, **Off-hours only** (firewall local time), **Country** and **Volume**. Hover any control, tab, column or flag for what it means.

## Notes

- All times are UTC, from FortiOS `eventtime` (seconds, ms, us or ns are all recognised); `date` + `time` + `tz` is the fallback.
- Traffic direction comes from the interface roles (`wan` = Internet, `lan` / `dmz` = internal). An `undefined` role on a VPN session counts as internal, so IPsec or SD-WAN overlays with non-RFC1918 addressing are classified correctly; other undefined roles fall back to the address class, and Overview says how many sessions needed that. Bytes use the per-interval deltas, so long sessions are not double-counted.
- **Input completeness is reported.** Every file under the input gets a status (read, partial, failed, not FortiGate); multi-member gzip files are followed member by member; any unreadable or partly decoded file raises a critical finding.
- Security-profile outcomes are mapped per record (blocked / allowed / unknown), so one blocked request never hides an allowed one, and an unrecognised action is never assumed blocked.
- "Succeeded after repeated failures" requires failures in the 24 h before the success, not just both in the totals.
- **Coverage gaps are called out.** A GUI download stops at a row limit, so a busy log type (forward traffic) can cover hours while the others cover days. Overview shows the span of each log type; export busy types day by day.
- Uploads through a forward proxy / SASE (Zscaler, Netskope and similar) are tagged and not scored as exfiltration.
- Only traffic that crosses the firewall is logged; same-subnet traffic and policies with logging off leave no trace.
- Not read in this version: FortiAnalyzer CSV exports and CEF-format syslog.

## Command line

```
mshta "FortiGate-Log-Analyzer.hta" "<input>" ["<outDir>"] [/auto] [/from:yyyy-MM-dd] [/to:yyyy-MM-dd]
```

`<input>` is a log folder or file (processed with `/auto`), a run `.json`, or an output folder (reloads its newest run). `/from` / `/to` set the case window: it filters the views and is recorded in runinfo, never scores.

## License

MIT (c) 2026 Ben Morris
