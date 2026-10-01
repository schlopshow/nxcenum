# nxcenum

A single bash wrapper around [nxc](https://github.com/Pennyw0rth/NetExec) (NetExec, the maintained
successor to CrackMapExec/`cme`) that runs a full Active Directory enumeration sweep. It automatically
runs an **unauthenticated recon phase**, and — if you give it credentials — follows up with a
**full authenticated enumeration phase**, including vulnerability scans and popular attack modules.

## Requirements

- [`nxc`](https://github.com/Pennyw0rth/NetExec) installed and on your `PATH`
  (`pipx install git+https://github.com/Pennyw0rth/NetExec`)
- `hashcat` if you want to crack any AS-REP/Kerberoast hashes the script pulls
- `responder` only if you use the manual hash-stealing steps printed in aggressive mode
- Bash on Linux/macOS (tested against a Kali-style attack box)

## Setup

```bash
chmod +x nxc_enum.sh
```

## Usage

```
./nxc_enum.sh -t <target> [-u <user> (-p <pass> | -H <hash>)] [options]
```

### Required

| Flag | Description |
|------|-------------|
| `-t TARGET` | IP, DNS name, CIDR range, or a file containing one target per line |

### Credentials (omit all three to run unauthenticated recon only)

| Flag | Description |
|------|-------------|
| `-u USER` | Username, a comma-separated list, or a file of usernames |
| `-p PASS` | Password, a list, or a file of passwords |
| `-H HASH` | NTLM hash, used instead of `-p` |

### Optional

| Flag | Description |
|------|-------------|
| `-d DOMAIN` | Domain name. Used for domain auth, and required by `-k` / some LDAP checks |
| `-l` | Use local account authentication instead of domain auth (`--local-auth`) |
| `-k` | Force Kerberos authentication (target must be given as an FQDN) |
| `-a` | **Aggressive mode.** Only applies to the authenticated phase. Also runs secret dumps (SAM/LSA/NTDS/LSASS), command execution, and prints manual steps for hash-stealing LNK/searchConnector drops. Off by default. |
| `-x CMD` | Command to execute on the target via SMB/WinRM (requires `-a`) |
| `-s SHARE` | Share name to spider for interesting/credential-bearing files |
| `-U USERFILE` | Username list used for the unauthenticated AS-REP roast attempt. Defaults to `./users.txt` if present |
| `-o OUTDIR` | Output directory. Defaults to `./nxc_enum_<target>_<timestamp>` |

## Examples

```bash
# Unauthenticated recon only — null sessions, anonymous LDAP, unauth vuln checks
./nxc_enum.sh -t 10.10.10.5

# Unauth recon, then full authenticated enumeration + vuln scans
./nxc_enum.sh -t 10.10.10.5 -u jsmith -p 'Passw0rd!' -d corp.local

# Same, but with an NTLM hash instead of a password
./nxc_enum.sh -t 10.10.10.5 -u jsmith -H 8846f7eaee8fb117ad06bdd830b7586c -d corp.local

# Full enumeration plus intrusive actions: secret dumps + a command exec check
./nxc_enum.sh -t 10.10.10.5 -u jsmith -p 'Passw0rd!' -d corp.local -a -x whoami

# Target a whole subnet, spider a specific share, custom output folder
./nxc_enum.sh -t 10.10.10.0/24 -u jsmith -p 'Passw0rd!' -d corp.local -s FINANCE$ -o ./acme_engagement
```

## What it does

### Phase 1 — Unauthenticated recon (always runs)

- SMB host fingerprinting (OS, SMB version, hostname, domain)
- SMB-signing / relay-candidate mapping (`--gen-relay-list`)
- Null-session and `guest` logon attempts
- Null-session share, user, group, and password-policy enumeration
- RID brute force
- Anonymous LDAP bind, user/group enumeration, domain SID
- AS-REP roast against a supplied username list (`-U`), no password needed
- Unauthenticated vulnerability checks: Zerologon, PetitPotam, MS17-010
- RDP login-prompt screenshot if NLA is disabled

If no credentials are supplied, the script stops here.

### Phase 2 — Authenticated enumeration (runs when `-u` + `-p`/`-H` are given)

- Credential verification
- **SMB:** sessions, shares, disks, computers, logged-on users, users, groups, local groups, RID brute, password policy
- **LDAP:** users, groups, `PASSWD_NOTREQD` accounts, delegation-trusted accounts, `adminCount=1` objects, domain SID, GMSA passwords
- **Roasting:** AS-REP roast + Kerberoasting, with the matching `hashcat` commands printed for offline cracking
- **RDP:** NLA and full-desktop screenshots
- **MSSQL:** privilege-escalation path check (`mssql_priv`)
- **GPP/autologon credential hunting:** `gpp_password`, `gpp_autologin`
- **Popular modules:** `get-network`, `laps`, `maq`, `keepass_discover`
- **Vulnerability scans:** Zerologon, PetitPotam, MS17-010, noPac, DFSCoerce, ShadowCoerce
- **File operations:** targeted share spider (`-s`) and a full read-only `spider_plus` catalog of all shares

### Aggressive mode (`-a`, authenticated phase only)

Off by default because these actions modify target state, dump credential material, or execute code.
Only enable against hosts you're explicitly authorized to test this way:

- SAM, LSA, and NTDS.dit dumps; LSASS memory dump via `lsassy`
- Command execution via `-x` (CMD, PowerShell, and WinRM)
- Printed (not auto-run) instructions for enabling RDP and for the Responder + `slinky`
  hash-stealing LNK/searchConnector workflow, since those require your attack IP and a listener

Pivoting via Chisel is also just printed as a manual reminder at the end of the run.

## Output

Every run creates a timestamped directory (or the one you pass via `-o`):

```
nxc_enum_<target>_<timestamp>/
├── run.log                 # full transcript of every command and its output
├── relay_list.txt           # hosts that don't require SMB signing
├── asreproast_unauth.out    # AS-REP hashes found pre-auth (if a userlist was available)
├── asreproast.out           # AS-REP hashes found authenticated
└── kerberoasting.out        # Kerberoastable SPN account hashes
```

Crack the roast output offline:

```bash
hashcat -m 18200 asreproast.out    /usr/share/wordlists/rockyou.txt --force
hashcat -m 13100 kerberoasting.out /usr/share/wordlists/rockyou.txt --force
```

## Notes

- Only run this against systems you are explicitly authorized to test (e.g. your own HTB/lab
  environment or an engagement with signed authorization).
- Some modules (MSSQL, LAPS, GMSA, KeePass, etc.) will simply return nothing if the target doesn't
  run that service or the account lacks the relevant rights — that's expected, not a script bug.
- `nxc` module names and options change between releases; if a command errors with "unknown module"
  or "unknown option," check `nxc <protocol> -L` / `nxc <protocol> -M <module> --options` against
  your installed version.
