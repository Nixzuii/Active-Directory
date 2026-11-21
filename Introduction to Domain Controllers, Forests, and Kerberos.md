## Part 1: Domain Controllers (DCs)

Think of a company with 5,000 employees. Someone has to keep the master list of who works there, what their passwords are, and what they're allowed to access. In Active Directory, that someone is the Domain Controller.  

A DC is just a **Windows Server with the "AD DS"** role installed. Installing that role turns an ordinary server into the "security office" of the network.  

Jobs done by DC:

| Job | Plain-English meaning | Why it matters |
| :--- | :--- | :--- |
| **Stores the AD database** | Keeps one file, `NTDS.dit`, listing every user, computer, group, and password hash | Attackers who steal this file get everyone's credentials, which is why DCs are the ultimate target |
| **Authenticates logins** | Checks "are you really who you say you are?" using Kerberos | Without it, nobody can log in |
| **Replicates** | Copies changes to other DCs | Keeps all DCs identical |
| **Delivers Group Policy** | Pushes rules (e.g. "password must be 12 characters") to all computers via a shared folder called SYSVOL | Admins manage thousands of PCs from one place |
| **Usually runs DNS** | Acts as the network's phone book | Computers find the DC through DNS |

### Why have more than one DC?

If there's one DC and it breaks, nobody can log in. So companies run 2 or more DCs that sync with each other. If you create a user on DC1, DC2 gets a copy moments later. That's called **replication**.  

**Analogy**: two bank branches that both hold the same customer records. If one burns down, the other keeps working.

## Part 2: Tree and Forest Structure

**Domain** is a single security territory, e.g. `company.local`, **Tree** is a group of domains sharing one naming pattern, **Forest** is one or more trees together.
