
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
