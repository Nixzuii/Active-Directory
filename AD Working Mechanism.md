# Working Mechanism of Active Directory

Before touching how AD works lets first get familiar with the AD Components. Let's keep the school analogy going, but scale it up: imagine a big education company that runs many schools in different cities.

## 1. Domain

**What it is**: A domain is a logical boundary, a group of users, computers, and resources that share the same AD database and the same security rules. It has a name like `company.com`.  
**Analogy**: One school. It has its own student list, its own rules, and its own principal.  

**Key points**:

- Every object (user, computer, group) lives inside a domain
- The domain is the main unit of administration and security
- Has its own security policies (password rules, lockout rules)
- Domain admins control only their own domain by default

## 2. Domain Controller (DC)

**What it is**: The actual server that runs Active Directory for a domain. It stores a copy of the AD database (a file called NTDS.dit) and answers login requests.  
**Analogy**: The school's security gate and office staff. Everyone must go through them to get verified.

**Key points**:

- **Authenticates** users (checks your identity)
- **Authorizes** access (decides what you can use)
- Stores the AD database
- Applies Group Policy
- Runs services like DNS and Kerberos (KDC)

## 3. Tree

**What it is**: A collection of domains that share a continuous naming structure (a common DNS namespace), connected in a parent-child hierarchy.  

**Example**:
```bash
company.com            (parent)
├── india.company.com  (child)
└── usa.company.com    (child)
```
