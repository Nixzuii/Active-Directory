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

**Analogy**: The head office school (`company.com`) and its branch schools (`india.company.com`, `usa.company.com`). They all share the same family name.  

**Key points**:

- Child domains automatically trust their parent (two-way, transitive)
- Share the same root name

## 4. Forest

**What it is**: The top-level container in AD. It's a collection of one or more trees that share a common schema, configuration, and Global Catalog, but may have different naming structures.  

**Example**: 
```bash
Forest
├── Tree 1: company.com
│     ├── india.company.com
│     └── usa.company.com
└── Tree 2: acquiredcorp.com
      └── uk.acquiredcorp.com
```

**Analogy**: The whole education company that owns multiple school families, even if they have different brand names (for example, after buying another company).

**Key points**:

- **The forest is the true security boundary in AD**, not the domain. Admins from one forest don't automatically control another forest.
- All domains in a forest share:

  - **Schema**: the blueprint defining what types of objects and attributes can exist (like a form template)
  - **Configuration partition**: info about the forest structure
  - **Global Catalog (GC)**: a searchable index of objects from all domains

- The first domain created is the forest root domain. It holds two powerful groups: Enterprise Admins and Schema Admins.

## 5. Read Only Domain Controller

**What it is**: A special DC that holds a read-only copy of the AD database. You cannot make changes on it directly.

**Analogy**: A small branch office with a photocopy of the registry book. They can look things up and verify students, but they can't edit the original. Changes only come from head office.

**Why use it**

- For remote branch offices with weak physical security or poor internet.
- If the RODC is stolen or hacked, the attacker can't modify AD or easily steal all passwords
