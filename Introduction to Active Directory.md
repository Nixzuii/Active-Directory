
# Introduction to Active Directory.

Imagine a company with 500 employees and 500 computers. Without **AD (Active Directory)**, every computer keeps it's own list of accounts, like 500 separate notebooks. If an employee named Panam changes her password, we’d walk to all 500 machines to update it.  

**Active Directory (AD)** replaces those 500 notebooks with one master notebook that every computer checks. Change something once, and it applies everywhere.  

**Analogy**: AD is the company’s HR database and security desk combined. HR knows who works there, and the security desk decides which doors our badge opens.

## The 2 Jobs: Authentication & Authorization.

|  | Question | Real-life example | AD example |
| --- | --- | --- | --- |
| Authentication | Who are you? | Showing your ID at the airport | Logging in with username and password |
| Authorization | What are you allowed to do? | Your boarding pass only lets you onto your flight | You can open the HR folder but not Finance |

**Order matters**: you always authenticate first, then you get authorized. You can’t decide what someone may access before knowing who they are.

## The 5 Components of an Active Direcory.

### Component 1: Domain

A domain is a logical group of users, computers, and resources that share one AD database and one set of security rules. Example: `company.local`. Think of it as one company’s private territory. Everyone inside follows the same rules and is listed in the same directory.

### Component 2: Domain Controller (DC)

The DC is the Windows Server that runs AD. It does two things:

- **Stores the database** of every user, computer, and group (the “records office”).
- **Handles every login request** (the “security guard”).

When we type our password in our PC, the PC doesn’t decide whether it’s correct. It asks the DC, and the DC replies yes or no.

> Companies usually run several DCs that copy each other’s data (called replication). If one fails, the others keep everyone working.

### Component 3: Tree

A tree is a group of domains that share one naming structure (namespace).

- Parent: `company.local`
- Children: `sales.company.local` and `hr.company.local`

Notice that the children’s names end with the parent’s name. That’s what makes it a tree. Think of a company headquarters with regional branches, all under one brand name.
