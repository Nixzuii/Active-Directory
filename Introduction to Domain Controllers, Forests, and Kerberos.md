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

A child domain is a sub-division, and its name extends its parent's:

```bash
company.local          (parent)
 ├─ sales.company.local   (child)
 └─ hr.company.local      (child)
```

Notice the children's names contain the parent's name. That shared, building-on-each-other naming is called a contiguous namespace. Everything in that group is a tree.

### Trusts: How domains agree to cooperate

A **trust** means "domain A is willing to accept domain B's users." When you create a child domain, AD **automatically** sets up a trust with its parent. It's **two-way** and **transitive**. 

**Two-way**: both sides trust each other. (Compare to one-way: "I trust you, but you don't trust me.") 

> If A trusts B, and B trusts C, then A trusts C.

Example: `sales` trusts `company` (its parent), and `company` trusts `hr`. Because trust is transitive, `sales` and `hr` can work together even though they never directly connected. So a `sales` user can be given access to an `HR` resource if permissions allow.

**Important**: trust doesn't mean automatic access. A trust only means "you're allowed to be considered." The actual permissions still have to be granted.

### Forest

A forest is one or more trees that share three things:

- The same schema: the "rulebook" defining what kinds of objects can exist (user, computer, printer, and their attributes).
- The same Global Catalog.
- The same forest root domain: the very first domain ever created.

A second tree can live in the same forest even with an unrelated name. For example, `company.local` and `partner.com` could be two trees in one forest. They don't share a naming pattern, but they share the schema and catalog.

## Kerberos Authentication

Without Kerberos, every time we open a file share, our PC would send our password across the network. Anyone listening could grab it. Kerberos lets us prove who we are without sending our password each time. Lets connect it with a theme park analogy:

1. At the entrance, we show our ID and get a day wristband. This is the TGT (Ticket Granting Ticket).
2. At each ride, we show the wristband and get a ride ticket. This is a service ticket.
3. We hand the ride ticket to the operator and get on. We never show our ID again.

**The three players**:

- Client: we, on our PC.
- KDC (Key Distribution Center): the ticket office. It lives on the DC and has two parts:
  - AS (Authentication Service): checks our ID and issues the TGT
  - TGS (Ticket Granting Service): swaps our TGT for service tickets
- Service: the thing we want (file server, website, etc.)
