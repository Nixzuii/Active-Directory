
# Introduction to Active Directory.

Active Directory is a Microsoft directory service. It runs on a server called **Domain Controller** and keeps a central database of users, computers, groups, and policies. It handles authentication, which means verifying who we are, and authorization, which means deciding what we can access. Organizations use it so employee can login once, admins can manage thousands of accounts from one place, and security rules like password policies apply everywhere. Because it controls access to everything, it's also a major target for attackers, so securing it is very important.

**Analogy**: Think of it like a school's office registry. It knows every student and teacher, which rooms they can enter, and what they're allowed to do. Active Directory does the same for people and computers in a company.

## Why use active directory

Without an AD, imagine a company with 500 computers. If a new employee joins the IT person would have to walk to every 500 computers and create an account by hand. If someone forgets their password, same problem. It would be chaos. With AD:

1. **One login for everything**: User login once and user can reach their email, shared folders, and printers. This is called Single Sign-On.
2. **Easy management**: Create a user once, and it works everywhere. When someone leaves, disable one account and they are locked out of everything.
3. **Security rules in one place**: The admin can say "everyone must use a password of 12+ characters" and it applies to every computer.
4. **Groups save time**: Instead of giving 50 people access to a folder one by one, you put them in a group and give access to the group.
5. **Control and tracking**: We can see who logged in, when, and from where.

## Common AD vulnerabilities.

Because AD holds the "keys to the kingdom," attackers love targeting it. If they take over AD, they can control the whole company. Here are the common weaknesses, again in simple terms:

| Vulnerability | Simple explanation |
| :--- | :--- |
| **Weak or reused passwords** | Like using "1234" as your locker code. Attackers guess it (password spraying, brute force). |
| **Too many admin accounts (excess privileges)** | Giving master keys to too many people. If one is hacked, the attacker gets everything. |
| **Kerberoasting** | Attackers request special service tickets and crack them offline to discover passwords of service accounts. |
| **Pass-the-Hash / Pass-the-Ticket** | Instead of stealing your password, the attacker steals the "proof you logged in" and reuses it, like copying your hall pass. |
| **Unpatched Domain Controllers** | Old software with known holes. Examples include Zerologon and PrintNightmare, which let attackers take over servers. |
| **Misconfigured permissions (ACLs)** | A door was left unlocked by mistake. Attackers find paths from a normal user to an admin. |
| **Legacy protocols (NTLM, SMBv1, LLMNR)** | Old, weak security methods that are easy to trick or intercept. |
| **Unconstrained delegation** | A server is trusted too much and can pretend to be any user. Attackers abuse that trust. |
| **Credentials stored in Group Policy** | Passwords accidentally saved in shared files anyone can read. |
| **Weak AD Certificate Services (AD CS) setup** | Misconfigured certificate templates let attackers request an ID card for an admin |
| **No monitoring or logging** | Nobody watching the cameras, so break-ins go unnoticed. |
| **Service accounts with never-expiring passwords** | A key that is never changed, so a stolen copy works forever. |
