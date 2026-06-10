# Azure Key Vault Managed HSM — Plain English Explanation

> This document explains the Contoso Managed HSM deployment architecture in simple, non-technical language. Use this to understand the "what" and "why" before diving into the detailed technical document.

---

## The Big Picture — What Are We Doing and Why?

**The problem:** Contoso has data in Azure (databases, VMs, storage). Right now, Microsoft encrypts that data using Microsoft's own keys. Contoso wants to use **their own keys** to encrypt their data — this is called **Customer-Managed Keys (CMK)**.

**Why?** Two reasons:

1. **Control** — If Contoso holds the keys, they can revoke access to their own data at any time. Microsoft can't read it without Contoso's permission.
2. **Compliance** — PCI-DSS (payment card industry standard) requires demonstrable control over encryption keys.

**The solution:** Azure Key Vault Managed HSM — a dedicated, tamper-proof, **FIPS 140-3 Level 3 validated** hardware vault in the cloud that stores and protects Contoso's encryption keys.

---

## How It All Fits Together — The Complete Picture

```
1. Create HSM
2. Generate 5 Security Domain key pairs → store offline → DR only
3. Create CMK keys → live inside HSM → never leave
4. Apply CMK to services:

   SQL MI:
   ┌──────────┐      ┌─────┐      ┌─────┐
   │ Database │─DEK──│ CMK │──in──│ HSM │
   │ data     │      │     │      │     │
   └──────────┘      └─────┘      └─────┘
   (SQL MI talks to HSM directly)

   VM Disks:
   ┌──────────┐      ┌─────┐      ┌─────┐      ┌─────┐
   │ Disk     │─DEK──│ DES │──────│ CMK │──in──│ HSM │
   │ data     │      │brdg │      │     │      │     │
   └──────────┘      └─────┘      └─────┘      └─────┘
   (DES connects the disk to the HSM)

   Storage:
   ┌──────────┐      ┌─────┐      ┌─────┐
   │ Blobs/   │─DEK──│ CMK │──in──│ HSM │
   │ Files    │      │     │      │     │
   └──────────┘      └─────┘      └─────┘
   (Storage talks to HSM directly — like SQL MI)

5. All three use DEK for fast encryption/decryption
6. All three use CMK (in HSM) to protect the DEK
7. The ONLY difference is VM disks need DES as an
   extra intermediary — SQL MI and Storage don't
```

---

## The Three Separate Things You Create

| Thing | Who Creates It | When | Where It Lives | Used For |
|---|---|---|---|---|
| **5 RSA keys + Security Domain** | Security Officers (one-time) | HSM activation | Offline (safe deposit box) | Only for HSM disaster recovery |
| **CMK keys** (Contoso-SQLMI-CMK, etc.) | Key Management Team (Crypto User role) | After HSM is active, before Phase 1 | Inside the HSM hardware (never leaves) | Encrypting/decrypting data every day |
| **Service connection** (TDE protector, DES, etc.) | Key Management Team or Infra Team | Phase 1 (when integrating services) | Azure resource configuration | Telling Azure services "use this HSM key" |

---

## What Lives Where — The Complete Map

| Thing | Where It Lives | What It Is |
|---|---|---|
| **CMK** (Contoso-SQLMI-CMK, etc.) | **Inside HSM hardware** — never leaves | The master encryption key that protects DEKs |
| **DEK** (Data Encryption Key) | **Next to the data** (database/disk/blob) — in **encrypted** form | The key that actually encrypts your data |
| **DEK (plaintext)** | **In memory only** — temporarily, when data is accessed | Unwrapped version of DEK, used for decryption, never written to disk |
| **DES** (Disk Encryption Set) | **Azure Resource Manager** — it's an Azure resource like a VM | A bridge/config object that connects VM disks to the HSM key |
| **Security Domain** | **Offline** — safe deposit box | Master key to rebuild the entire HSM |

> **Key point: Only CMK lives in the HSM.** DEK lives next to your data (but encrypted). DES lives in Azure as a resource (just a bridge for VM disks). The HSM is purely a key protection vault — it wraps/unwraps DEKs on request but never stores them.

```
WHAT'S INSIDE THE HSM:
└── CMK keys (and ONLY CMK keys)
    • Contoso-SQLMI-CMK
    • Contoso-DISK-CMK
    • Contoso-STORAGE-CMK
    That's it. Nothing else lives in the HSM.


WHAT'S STORED WITH YOUR DATA (encrypted):
└── DEK (wrapped/encrypted by CMK)
    • SQL MI stores its encrypted DEK with the database
    • VM disks store their encrypted DEK with the disk
    • Storage stores its encrypted DEK with the blob/file


WHAT'S AN AZURE RESOURCE (like a VM):
└── DES (Disk Encryption Set)
    • Lives in Azure Resource Manager
    • Is a config object that points to the HSM key
    • Only needed for VM disks (SQL MI and Storage don't need it)
    • Must be created in EACH region separately


WHAT'S OFFLINE:
└── Security Domain file + 5 RSA keys
    • Only for HSM disaster recovery
```

---

## How DEK Actually Works — At Rest vs When Accessed

```
AT REST (on disk):
┌──────────────┐     ┌───────────────────────┐
│ Encrypted    │     │ Encrypted DEK         │
│ Data         │     │ (wrapped by CMK)      │
│              │     │                       │
│ (can't read  │     │ (can't use without    │
│  without DEK)│     │  CMK in HSM)          │
└──────────────┘     └───────────────────────┘
Both sit together on disk. Both useless without the HSM.


WHEN DATA IS ACCESSED (in memory):
┌──────────────┐     ┌───────────────────────┐     ┌─────────┐
│ Encrypted    │     │ Encrypted DEK ────────┼────►│   HSM   │
│ Data         │     │                       │     │         │
│              │     │ Plaintext DEK ◄───────┼─────│ Unwraps │
│              │◄────│ (in memory only)      │     │ using   │
│ Decrypted!   │     │                       │     │   CMK   │
└──────────────┘     └───────────────────────┘     └─────────┘
Plaintext DEK exists ONLY in memory, temporarily. Never written to disk.
```

---

## Important: DEK vs DES — They Sound Similar but Are Completely Different

| | DEK | DES |
|---|---|---|
| **Full name** | Data Encryption Key | Disk Encryption Set |
| **What it is** | An actual encryption **key** | An Azure **resource** (a configuration object) |
| **Where it lives** | **Next to the data** (encrypted form) + **in memory** (plaintext, temporarily) | **Azure Resource Manager** (like a VM or Storage Account) |
| **Purpose** | Encrypts/decrypts the actual data | **Connects** a VM disk to the CMK in the HSM |
| **Who creates it** | Azure creates it automatically (you never see it) | **You** create it manually |
| **Lives inside HSM?** | ❌ No — lives with the data (encrypted) | ❌ No — lives in Azure Resource Manager |
| **Needed for all services?** | ✅ Yes — SQL MI, VM Disks, and Storage all have DEKs | ❌ No — only VM Disks need DES. SQL MI and Storage don't. |
| **Per-region?** | Created automatically per resource | ✅ Must be created in EACH region separately |
| **Analogy** | The actual padlock on the box | The cable that connects the padlock to the safe |

---

## Why This Design Is Secure

Even if an attacker gets access to:

| What They Steal | Can They Read Data? | Why? |
|---|---|---|
| The encrypted data | ❌ No | They don't have the DEK |
| The encrypted DEK | ❌ No | They can't unwrap it without the CMK |
| The encrypted data + encrypted DEK | ❌ No | Still need CMK (in HSM) to unwrap DEK |
| The HSM endpoint URL | ❌ No | They need Azure AD credentials + RBAC permission |
| Azure AD credentials without RBAC | ❌ No | RBAC denies access to the key |

**The only way to read the data:** Have valid Azure AD credentials + correct RBAC role + network access (Private Endpoint) + the CMK is available in the HSM. All four conditions must be true simultaneously.

---

## The Full Lifecycle — Simplified

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Security Domain (5 RSA keys)                               │
│  = Master key to the VAULT ITSELF                           │
│  = Only used if you need to REBUILD the vault               │
│  = Sits offline, never touched in normal operations         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                         │
                         │ (used ONLY for disaster recovery)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  HSM Vault (Contoso-PROD-HSM)                               │
│  Contains ONLY:                                             │
│    • Contoso-SQLMI-CMK                                      │
│    • Contoso-DISK-CMK                                       │
│    • Contoso-STORAGE-CMK                                    │
│                                                             │
│  These CMK keys wrap/unwrap DEKs on request.                │
│  The DEKs themselves do NOT live here.                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                         │
                         │ (wraps/unwraps DEKs on request)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Your Resources (each has its own encrypted DEK)            │
│                                                             │
│    • SQL MI databases                                       │
│      └── Encrypted DEK stored with database                 │
│      └── When queried: DEK sent to HSM → unwrapped →        │
│          data decrypted in memory → result returned         │
│                                                             │
│    • AD VM disks                                            │
│      └── Encrypted DEK stored with disk                     │
│      └── DES (Azure resource) bridges disk to HSM           │
│      └── When VM boots: DEK sent to HSM via DES →           │
│          unwrapped → disk readable → OS loads               │
│                                                             │
│    • Storage blobs/files                                    │
│      └── Encrypted DEK stored with blob/file                │
│      └── When downloaded: DEK sent to HSM → unwrapped →     │
│          data decrypted → file returned                     │
│                                                             │
│  Data is ALWAYS encrypted at rest.                          │
│  Data is decrypted on-the-fly when accessed.                │
│  Users/apps never see the encryption — it's transparent.    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Section by Section — Plain English

### Terminology & Assumptions

**Two Azure regions in India:**

- **South India (Chennai)** — where Contoso runs their main production workloads
- **Central India (Pune)** — the backup/disaster recovery site

We're **assuming** this setup, but Contoso hasn't confirmed yet — that's why we have the "pending confirmation" questions.

---

### Section 1: High-Level Architecture

**Think of it like this:**

Imagine Contoso has a **safe** (the HSM) where they keep their encryption keys. This safe is located in their Chennai office (primary). But they also want a **copy of the safe** in their Pune office (backup), so if Chennai has a problem, Pune can keep running.

**Key points:**

- There's only **ONE safe** (one HSM resource called `Contoso-PROD-HSM`), but it has a **mirror** in Pune that stays automatically in sync
- Any key you put in the Chennai safe **automatically appears** in the Pune safe
- Any access permissions you set in Chennai **automatically apply** in Pune
- You don't manage two separate things — you manage one thing that exists in two places

**Cost note:** Even though it's ONE logical resource, Microsoft provisions dedicated hardware in **both** regions. You pay for both pools: ~$2,304/month × 2 = ~$4,608/month.

**The services that use keys from this safe:**

- **SQL MI** — the database (uses the key to encrypt all database data)
- **AD Server VMs** — the domain controller servers (uses the key to encrypt their hard drives)
- **Storage, Cosmos DB, etc.** — will be added in Phase 2

---

### Section 2: What Microsoft Does Behind the Scenes

**When Contoso creates the HSM, here's what happens invisibly:**

1. Microsoft takes a physical piece of hardware called a **Marvell LiquidSecurity HSM** — a tamper-proof chip that is **FIPS 140-3 Level 3 validated** and specifically designed to store cryptographic keys securely
2. Microsoft gives Contoso **3 of these chips** (called partitions), placed on **different server racks** so that if one rack loses power or a chip fails, the other two keep working
3. The whole thing runs inside a **Confidential Compute envelope** — think of it as a room within a room. Even Microsoft's own datacenter operators can't peek inside
4. Microsoft exposes a web address (like `contoso-prod-hsm.managedhsm.azure.net`) that Contoso's services call to use the keys

**What FIPS 140-3 Level 3 means in plain English:** It's the U.S. government's certification that the hardware is tamper-evident, tamper-resistant, and that the keys physically cannot be extracted from the chip. This is the level required for most regulated industries (banking, healthcare, government). PCI-DSS auditors look for this.

**What Contoso never has to do:**

- Buy hardware ❌
- Install anything ❌
- Update firmware ❌
- Replace broken parts ❌
- Configure high availability ❌

Microsoft does all of this. Contoso just creates keys and sets permissions.

**Important note about South India:** This region doesn't have Azure Availability Zones (separate buildings/power grids). Instead, the 3 partitions are on **different racks in the same datacenter**. Central India (Pune) does have Availability Zones. Either way, the HA protection works — just through a different mechanism.

---

### Section 3: The Deployment Steps (Phase 0)

This is the **setup phase** before any service starts using CMK. Think of it as building the safe before putting anything in it.

---

#### Step 1: Create the HSM (~20-30 minutes)

You run a command or click through Azure Portal. Microsoft automatically provisions the hardware. Two critical flags:

- **Soft-delete** (on by default, **cannot be disabled** for Managed HSM) — if someone accidentally deletes the HSM or a key, it goes to a "recycle bin" for a retention period (7-90 days) before being permanently destroyed
- **Purge protection** (opt-in at creation time, **irreversible once enabled — cannot be disabled or overridden by anyone, including Microsoft**) — prevents anyone from emptying the "recycle bin" early

**Why is purge protection so important?**

- **SQL MI won't work without it.** Azure SQL literally checks "does this HSM have purge protection?" and refuses to use it if the answer is no.
- **PCI-DSS needs it.** Auditors want to see that keys can't be permanently destroyed without going through the full retention period.
- **Billing catch:** If someone deletes the HSM, it sits in the recycle bin for the full retention period AND you keep paying for it during that time, because purge protection prevents early cleanup.

---

#### Step 2: Activate the Security Domain (~15 minutes)

This is the **most important step** and the moment Contoso takes true ownership.

**Analogy:** Imagine the safe has a master combination. Except instead of one combination, it's split into 5 pieces, and you need any 3 of the 5 pieces to open it. Each piece is given to a different security officer at Contoso.

**How it works:**

1. Five Contoso security officers each generate a key pair (like a digital lock and key)
2. They download the "Security Domain" file from the HSM — this is the encrypted master secret
3. The file is encrypted so that you need any 3 of the 5 officers' keys to decrypt it

**This is Contoso's root of trust.** After this step:

- Microsoft can **never** access Contoso's keys — they physically can't
- If Contoso loses the Security Domain file AND 3+ of the 5 officer keys → **all keys are permanently, irrecoverably lost.** There's no "call Microsoft support" recovery. Gone forever.

> 💡 That's why the document emphasizes storing these offline in separate secure locations (think: safe deposit boxes at different banks).

**The quorum (3 of 5) explained:**

| Combination | Can Decrypt Security Domain? |
|---|---|
| Any 3 officer keys out of 5 | ✅ Yes |
| Any 4 officer keys out of 5 | ✅ Yes |
| All 5 officer keys | ✅ Yes |
| Only 2 officer keys | ❌ No |
| Only 1 officer key | ❌ No |

**Why 3 of 5?**
- If up to 2 officers leave the company or lose their keys → the remaining 3 can still recover ✅
- If 1 or 2 rogue officers try to steal keys → they can't, they don't meet quorum ✅
- Stronger separation of duties than 2-of-3, and matches the recommended posture in the technical guide

---

#### Step 3: Create the CMK Keys

This is when the **actual encryption keys** are created. The person with the **Crypto User** role runs a simple command:

```bash
az keyvault key create \
  --hsm-name Contoso-PROD-HSM \
  --name Contoso-SQLMI-CMK \
  --kty RSA-HSM \
  --size 3072
```

**What happens:** The HSM's tamper-proof chip generates the key **internally**. The key never existed anywhere else. It never leaves the HSM. You just gave it a name and told the HSM "create it."

Three keys are created (just once, in the primary region — they auto-replicate):

- `Contoso-SQLMI-CMK` — for database encryption
- `Contoso-DISK-CMK` — for VM disk encryption
- `Contoso-STORAGE-CMK` — for storage encryption (Phase 2)

No `-SI` or `-CI` suffixes needed — there's only one key per purpose, replicated to both regions.

> 💡 **Key size note:** RSA 3072 is the modern PCI-DSS / NIST recommended minimum and matches the technical deployment guide. RSA 2048 is still supported but is being phased out for new deployments.

---

#### Step 4: Set up RBAC (permissions)

This is about **who can do what** with the HSM. Here's where it gets counterintuitive:

| Role Name | What You'd Think It Does | What It Actually Does |
|---|---|---|
| **Managed HSM Administrator** | "Controls everything" | Only manages Security Domain, backup/restore, and role assignments. Can't touch keys at all. |
| **Crypto Officer** | "Manages keys" | **WRONG.** This is a governance/compliance role. Can purge/recover deleted keys, export keys, and manage roles. **Cannot** create, rotate, or delete live keys. |
| **Crypto User** | "Just uses keys" | **WRONG.** This is actually the **key management** role. Can create, rotate, delete, import keys AND use them for crypto. |
| **Crypto Service Encryption User** | "For Azure services" | Correct. This is the minimal role that Azure services (SQL MI, DES, Storage) need — just wrap and unwrap keys. |

**Why this matters:** If Contoso assigns their Key Management team the "Crypto Officer" role (because the name sounds right), that team **won't be able to create or rotate keys.** The names are misleading and this is the #1 mistake people make.

---

#### Step 5: Enable multi-region replication

You add Central India (Pune) as an "extended region." From that moment:

- All keys sync automatically
- All RBAC permissions sync automatically
- All policies sync automatically
- You now have one HSM that works from two cities

---

#### Step 6: Set up Private Endpoints

This creates a **private doorway** from Contoso's virtual network to the HSM. Instead of traffic going over the public internet, it stays entirely inside Azure's backbone network. This is needed in **both regions** because workloads in both Chennai and Pune need to reach the HSM.

---

### Section 4: SQL MI Integration (Phase 1a)

**How database encryption works — simplified:**

```
Contoso's Database Data
     │
     ▼
Encrypted by a DEK (Data Encryption Key)
     │
     ▼
The DEK itself is encrypted ("wrapped") by the CMK in the HSM
     │
     ▼
CMK lives in the HSM and NEVER comes out
DEK lives with the database in encrypted form
```

**It's a two-key system:**

1. **DEK (Data Encryption Key)** — the key that actually encrypts database rows. SQL MI generates this automatically. It's stored **with the database** in encrypted form.
2. **CMK (Customer-Managed Key)** — the key that protects the DEK. This lives in the HSM and **never leaves**.

**Why two keys?** Performance. Encrypting every database row with an HSM key would be extremely slow (HSM operations are slower than in-memory operations). So the DEK does the heavy lifting in memory, and the HSM only needs to wrap/unwrap the DEK (a single operation at startup or key rotation).

**The setup is:**

1. Give SQL MI an identity (like an employee badge)
2. Tell the HSM "this SQL MI identity is allowed to use the `Contoso-SQLMI-CMK` key for wrapping/unwrapping only"
3. Tell SQL MI "use this HSM key as your TDE protector"
4. Done — SQL MI handles the rest automatically

**What happens when data is accessed:**

```
USER/APP                    SQL MI                       HSM
    │                         │                         │
    │  "Give me customer      │                         │
    │   records"              │                         │
    │────────────────────────►│                         │
    │                         │                         │
    │                         │  "Unwrap my DEK please" │
    │                         │  (sends encrypted DEK)  │
    │                         │────────────────────────►│
    │                         │                         │
    │                         │  "Here's your DEK"      │
    │                         │  (plaintext, in memory) │
    │                         │◄────────────────────────│
    │                         │                         │
    │                         │  (decrypts data         │
    │                         │   in memory using DEK)  │
    │                         │                         │
    │  "Here's your data"     │                         │
    │◄────────────────────────│                         │
```

**If the HSM goes down:** SQL MI can't unwrap its DEK → can't decrypt data → **database goes offline.** This is why HA and multi-region replication matter so much.

---

### Section 5: AD Server Disk Encryption (Phase 1b)

**Same concept as SQL MI, but with one extra piece.** Instead of database encryption, we're encrypting the **hard drives** of the AD domain controller VMs.

The mechanism uses a **Disk Encryption Set (DES)** — think of it as a bridge that connects Azure Disks to the HSM key. SQL MI doesn't need this bridge (it talks to HSM directly), but VM disks do.

```
SQL MI (no bridge needed):
Database data ←── DEK ←── CMK (in HSM)
                    ↑
              SQL MI calls HSM directly


VM Disks (needs DES as bridge):
Disk data ←── DEK ←── DES ←── CMK (in HSM)
                        ↑
                  DES connects disk to HSM
                  (Azure Compute needs this intermediary)

Where things live:
• DEK → stored with the disk (encrypted form)
• DES → Azure resource (like a VM — NOT inside the HSM)
• CMK → inside the HSM (never leaves)
```

**Why the difference?** SQL MI is a managed database service — Microsoft built HSM integration directly into it. VM Disks are managed by Azure Compute, a different service that uses DES as the connector. It's an architectural difference, not a functional one. **Both use a DEK for fast encryption, both protect the DEK with a CMK in the HSM.**

**Critical difference from SQL MI:** The DES is a **regional Azure resource** — it doesn't auto-replicate and it does NOT live inside the HSM. If Contoso uses Azure Site Recovery (ASR) to replicate VMs to Pune for DR, they need a **separate DES in Pune** pointing to the same HSM key. The key replicates automatically, but the DES must be manually created in both regions.

**Operational risk:** Changing disk encryption on an existing VM may require **shutting down the VM.** For AD domain controllers, this means planning a maintenance window and ensuring another DC is available so the domain stays operational.

---

### Section 6: Network (Private Endpoints)

**Analogy:** The HSM has a front door (public endpoint) that faces the street, and a back door (private endpoint) that connects directly to Contoso's private building through a tunnel.

**With Private Endpoints:**

- The front door is **locked and boarded up** (public access disabled)
- All access goes through the **private tunnel** inside Azure's network
- Nobody on the internet can even see the HSM exists

**Operational catch:** If the front door is locked, Contoso's admins also can't manage the HSM from their laptops at home or from the Azure Portal directly. They need a **jump box or Azure Bastion** inside the VNet — a machine that's inside the "building" and can reach the back door.

---

### Section 7: RBAC (Already Covered Above)

The key takeaway is the **role name confusion** — Crypto Officer ≠ key management, Crypto User = key management. And all service identities (SQL MI, DES, Storage) get the minimal `Crypto Service Encryption User` role, scoped to only the specific key they need.

---

### Section 8: Security Domain (Already Covered Above)

The "split combination" model — 5 officers, need any 3 to recover. Store offline. Lose it = lose everything.

**Remember:** The Security Domain is for the **vault itself** (HSM recovery). The CMK keys inside the vault are what encrypt/decrypt your actual data every day. These are two completely separate things.

---

### Section 9: Disaster Recovery

**Normal operations:**

- Chennai: everything running, keys active
- Pune: everything synced and standing by

**If Chennai goes down:**

1. Pune already has all the keys (auto-replicated) ✅
2. SQL MI in Pune gets promoted to primary
3. AD VMs in Pune boot using the replicated keys (DES was pre-created) ✅
4. **The global FQDN `contoso-prod-hsm.managedhsm.azure.net` is automatically swung to Pune** by Microsoft's internal routing layer (DNS TTL ~5 seconds). **Contoso does NOT deploy or operate any Traffic Manager profile** — the routing is platform-managed and happens behind the single FQDN that all services already use.
5. Everything keeps running — services don't need to be reconfigured to point at a different HSM endpoint

**No emergency key provisioning needed** — that's the whole point of multi-region replication.

**Why this matters:** The same connection string keeps working in both regions. SQL MI, DES, and Storage Accounts don't need to be re-bound to a "Pune HSM" because there is no separate "Pune HSM" from their point of view — there is only `Contoso-PROD-HSM`, and Microsoft routes the request to whichever regional pool is healthy.

**One gotcha:** If someone **removes** the Pune extended region from the HSM (not the same as a regional outage — this is an administrative action), the Pune HSM presence is **purged immediately** — not soft-deleted. Keys in Chennai are fine, but Pune workloads lose access instantly.

---

### Section 10: Key Rotation

Keys should be rotated periodically (annually, quarterly — whatever Contoso's policy is). The basic process is:

1. **Crypto User** creates a new version of the key (same name, new version number)
2. Update services to use the new version
3. Old data re-encrypts automatically with the new version (for most services)
4. Old key version is kept around so existing encrypted data can still be decrypted

#### Manual rotation vs auto-rotation

Managed HSM supports **two ways** to do step 1:

- **Manual** — someone with the Crypto User role runs `az keyvault key rotate` (or creates a new version explicitly) on a schedule the team manages.
- **Auto-rotation policy** — you attach a policy to each key telling the HSM "rotate yourself every N days" or "rotate yourself N days before expiry." The HSM then creates new versions on its own. **Yes, this is fully supported in Managed HSM** (same model as Key Vault Premium).

A typical policy in plain English:

> *"Create a new version of this key 90 days after the current version was created. Each version expires 1 year after creation. Send me an Event Grid notification 30 days before expiry."*

#### The catch — HSM auto-rotation ≠ service auto-pickup

The HSM creating a new key version is only half the story. **Whether the downstream service starts using the new version automatically depends on how that service was bound to the key:**

| Service | Auto-picks up new key version? | What's needed |
|---|---|---|
| **Azure Storage** (`Contoso-STORAGE-CMK`) | ✅ Yes — IF bound with a **versionless** key URI (`.../keys/Contoso-STORAGE-CMK` with no version suffix) | Nothing. Fully hands-off. |
| **SQL MI** (`Contoso-SQLMI-CMK`, TDE protector) | ❌ No — SQL TDE always binds to a **specific key version**, not a versionless URI | Operator must run `az sql server tde-key set --kid <new-version-URI>` after rotation |
| **VM Disks via DES** (`Contoso-DISK-CMK`) | ⚠️ Yes — IF the DES has `autoKeyRotationEnabled = true` and uses a versionless URI. **BUT** if VMs are replicated via ASR, the auto-rotation **breaks ASR replication** — ASR has to be disabled and re-enabled | Auto for plain DES; controlled maintenance window if ASR is in play |

#### Recommended rotation strategy for Contoso

| Key | Suggested policy | Why |
|---|---|---|
| `Contoso-STORAGE-CMK` | **Auto-rotate every 12 months** + versionless URI on every Storage Account | Zero-touch. Storage handles it natively. |
| `Contoso-SQLMI-CMK` | **Notify-only at 11 months** (Event Grid alert, no auto-rotate) | Forces a planned maintenance window where the ops team updates the TDE protector explicitly. Prevents drift between the HSM and SQL MI. |
| `Contoso-DISK-CMK` | **Notify-only at 11 months** (no auto-rotate) | If ASR is in use, auto-rotation will silently break replication. Notify-only keeps the operator in the loop and avoids surprises. |

**General principle:** *Auto-rotate where the service can follow along automatically (Storage). Notify-only where the operator must do something (SQL MI, DES with ASR).*

**ASR warning (still applies):** Rotating `Contoso-DISK-CMK` on ASR-replicated VMs always requires **disabling and re-enabling replication**. ASR does not auto-detect new key versions. Plan a maintenance window for every disk-key rotation.

---

### Section 11: Resource Inventory

**What exists once and replicates:**

- 1 HSM resource (two regional presences)
- 3 CMK keys (auto-replicated) — only things that live INSIDE the HSM
- RBAC assignments (auto-replicated)

**What must be created per region:**

- Private Endpoint (one per region)
- Private DNS Zone (one per region)
- Disk Encryption Set (one per region — an Azure resource, NOT inside the HSM, needed for VM disks)
- Jump Box / Bastion (one per region — needed for admin access)

**What Azure creates automatically (you never manage these):**

- DEKs — one per resource (SQL MI, each disk, each storage account), stored with the data in encrypted form

---

### Section 12: Pricing

**The simple version:**

| What | Monthly Cost |
|---|---|
| HSM pool (Chennai) | ~$2,304 |
| HSM pool (Pune — extended region) | ~$2,304 |
| Keys + operations | ~$15-20 |
| **Total** | **~$4,620/month** |

Even though it's ONE logical resource, Microsoft provisions dedicated hardware in **both** regions — so you pay for both.

**Compared to the old way (Dedicated HSM):**

- Old: ~$6,984/month for HA in ONE region, no DR
- New: ~$4,620/month for HA in TWO regions with DR
- **Savings: ~34%** with better coverage

If you compare two-region to two-region (old way would need 4 devices = ~$13,968/month), savings are **~67%**.

---

### Section 13: Open Questions

These are the things we **cannot finalize** until Contoso answers. The biggest ones:

1. **Does Contoso use ASR?** — Affects DES requirements and key rotation planning
2. **Does Contoso have jump boxes in both regions?** — Needed for admin operations with private endpoints

---

## Azure Storage Encryption with CMK (Phase 2)

### How It Compares to SQL MI and VM Disks

Storage follows the **same pattern** but is actually the **simplest** of the three.

```
SQL MI (database):
Database data ←── DEK ←── CMK (in HSM)
                    ↑
              SQL MI talks to HSM directly
              (no intermediary needed)


VM Disks (AD servers):
Disk data ←── DEK ←── DES ←── CMK (in HSM)
                        ↑
                  DES is the bridge
                  (Azure Compute needs this intermediary)


Storage (blobs, files, queues, tables):
Storage data ←── DEK ←── CMK (in HSM)
                    ↑
              Storage Account talks to HSM directly
              (no intermediary needed — like SQL MI!)
```

**Storage works like SQL MI** — it talks directly to the HSM. No DES needed. No intermediary. It's actually the simplest integration of the three.

---

### How Azure Storage Encryption Works — Simplified

```
Contoso's Storage Data (blobs, files, etc.)
     │
     ▼
Encrypted by a DEK (Data Encryption Key)
     │    (DEK stored with the blob/file in encrypted form)
     ▼
The DEK itself is encrypted ("wrapped") by the CMK in the HSM
     │
     ▼
CMK lives in the HSM and NEVER comes out
```

**Exactly the same two-key system as SQL MI:**

1. **DEK** — Azure Storage generates this automatically. One DEK per storage service (blob, file, queue, table). Stored **with the data** in encrypted form. You never see it.
2. **CMK** — `Contoso-STORAGE-CMK` in the HSM. Protects the DEK. Never leaves the HSM.

---

### What Happens When Someone Accesses a Blob

```
USER/APP                 STORAGE ACCOUNT              HSM
    │                         │                         │
    │  "Download report.pdf"  │                         │
    │────────────────────────►│                         │
    │                         │                         │
    │                         │  "Unwrap my DEK please" │
    │                         │  (sends encrypted DEK)  │
    │                         │────────────────────────►│
    │                         │                         │
    │                         │  "Here's your DEK"      │
    │                         │  (plaintext, in memory) │
    │                         │◄────────────────────────│
    │                         │                         │
    │                         │  (decrypts blob         │
    │                         │   using DEK)            │
    │                         │                         │
    │  "Here's report.pdf"    │                         │
    │◄────────────────────────│                         │
```

**Identical to SQL MI.** The user never knows encryption is happening. It's completely transparent.

---

### The Setup Steps — Storage

```
Step 1: Give the Storage Account an identity (Managed Identity)
Step 2: Tell the HSM "this Storage Account identity can use Contoso-STORAGE-CMK"
Step 3: Tell the Storage Account "use this HSM key for encryption"
Step 4: Done — automatic from here
```

---

### Comparison — All Three Services

| Aspect | SQL MI | AD VM Disks | Azure Storage |
|---|---|---|---|
| **What gets encrypted** | Database data (rows/tables) | VM hard drives (OS + data disks) | Blobs, files, queues, tables |
| **Encryption method** | TDE (Transparent Data Encryption) | Server-Side Encryption | Storage Service Encryption (SSE) |
| **Has a DEK?** | ✅ Yes (auto-generated) | ✅ Yes (auto-generated) | ✅ Yes (auto-generated) |
| **Where DEK lives** | With the database (encrypted) | With the disk (encrypted) | With the blob/file (encrypted) |
| **DEK protected by** | CMK in HSM | CMK in HSM | CMK in HSM |
| **Needs an intermediary?** | ❌ No — talks to HSM directly | ✅ Yes — needs DES (Disk Encryption Set) | ❌ No — talks to HSM directly |
| **DES lives in HSM?** | N/A | ❌ No — DES is an Azure resource | N/A |
| **CMK key used** | `Contoso-SQLMI-CMK` | `Contoso-DISK-CMK` | `Contoso-STORAGE-CMK` |
| **RBAC role needed** | Crypto Service Encryption User | Crypto Service Encryption User | Crypto Service Encryption User |
| **RBAC scope** | `/keys/Contoso-SQLMI-CMK` | `/keys/Contoso-DISK-CMK` | `/keys/Contoso-STORAGE-CMK` |
| **Identity type** | System-Assigned Managed Identity | DES Managed Identity | System-Assigned Managed Identity |
| **DES needed per region?** | N/A | ✅ Yes (regional Azure resource) | N/A |
| **Downtime required?** | ❌ No | ⚠️ May require VM deallocation | ❌ No |
| **Encryption is transparent?** | ✅ Yes — apps don't know | ✅ Yes — OS doesn't know | ✅ Yes — apps don't know |
| **Auto key rotation support?** | ⚠️ Manual update needed | ⚠️ Manual update + ASR impact | ✅ Can auto-detect new key versions |

---

### Why Storage Is the Easiest

| Reason | Details |
|---|---|
| **No intermediary** | Unlike VM disks (which need DES), Storage talks to HSM directly — like SQL MI |
| **No downtime** | You can switch from Microsoft-managed keys to CMK on a live storage account with zero downtime |
| **No regional DES headache** | No DES means no need to create a separate resource in the DR region for ASR |
| **Automatic key rotation support** | Storage can automatically pick up new key versions — no manual update needed |

---

### One Important Note for DR

Even though Storage doesn't need DES, if Contoso has storage accounts in **both regions**, each storage account needs its own:

- Managed Identity
- RBAC assignment to the HSM key

But since RBAC **auto-replicates** via multi-region replication, and the key auto-replicates too — the only "per-region" work is pointing each storage account to the HSM key. Much simpler than VM disks.

---

## Bank Analogy — The Complete Picture

Think of it like a bank:

1. **Security Domain** = the deed to the bank building itself. You lock it in a vault and never touch it unless the building burns down and you need to prove ownership to rebuild.

2. **CMK keys** = the safety deposit **boxes** inside the bank. They are bolted to the floor (inside the HSM) and never leave the building.

3. **DEK** = the padlock your tenant puts on their stuff. The tenant (SQL MI, VM, Storage) stores the padlock **with their stuff** (next to the data), but in a locked state. To unlock the padlock, they need to bring it to the safety deposit box (CMK in HSM) to get it opened (unwrapped). The opened padlock (plaintext DEK) is only used temporarily in-hand (in memory) and never taken home (never written to disk).

4. **DES** = the cable that connects the padlock to the safety deposit box. SQL MI and Storage walk directly to their box and don't need a cable. VM disks access the box through a different department (Azure Compute), so they need the cable (DES) to make the connection.

5. **Applying CMK to services** = giving each tenant a key card that only opens THEIR specific safety deposit box, and telling them "this is your box, bring your padlocks here."

---

> *"We're setting up a dedicated, FIPS 140-3 Level 3 validated, tamper-proof hardware vault in Azure that Contoso owns and controls. Microsoft provides and maintains the hardware, but can never see the keys inside — not even their own operators. Contoso's databases, VM disks, and storage will be encrypted with keys from this vault. The vault is replicated across Chennai and Pune so if one region goes down, everything keeps running — automatically, behind a single FQDN, with no customer-managed traffic routing. It costs about $4,600/month — 34% less than the old approach — and meets PCI-DSS 4.0 requirements out of the box."*

---

## Appendix — Azure Key Management Options Compared

This table compares the four Azure key management options, with the **Managed HSM** column corrected against Microsoft Learn and aligned with the rest of this document.

| Attribute | Platform-Managed Keys (PMK) | CMK — Azure Key Vault (Standard) | CMK — Azure Key Vault Premium (Shared HSM) | CMK — Azure Key Vault **Managed HSM** (Single-Tenant HSM) |
|---|---|---|---|---|
| **Key Ownership** | Microsoft (platform) — Azure generates and fully controls keys; no customer access or visibility | Customer — manages key lifecycle in Key Vault; Microsoft manages underlying infrastructure | Customer — manages key lifecycle in HSM-backed Key Vault; keys stored in Microsoft-hosted **multi-tenant** HSM hardware | Customer — full and exclusive control (**key sovereignty**) in a **single-tenant Managed HSM cluster**. *Not the same product as the legacy "Azure Dedicated HSM" (SafeNet Luna) service.* |
| **Key Storage** | Internal to Azure service — transparent and inaccessible to customer | Software-protected keys in Azure Key Vault (no HSM hardware protection) | HSM-protected keys in Azure Key Vault **multi-tenant** HSM hardware (shared across tenants) | HSM-protected keys in a **dedicated single-tenant HSM cluster** built on **Marvell LiquidSecurity** adapters. Keys never leave the HSM boundary. |
| **Object Types Stored** | N/A (transparent) | Keys, **secrets, and certificates** | Keys, **secrets, and certificates** | **Cryptographic keys only** — no secrets, no certificates. (Use a separate Key Vault for those.) |
| **Hardware Validation** | N/A | None (software-protected) | FIPS 140-2 Level 3 (Platform 2) / FIPS 140-2 Level 2 (older Platform 1) | **FIPS 140-3 Level 3** validated. Also covers **PCI DSS**, **PCI 3DS**, and **PCI DSS 4.0**. |
| **Control over Key Lifecycle** | None — platform generates, rotates, and retires keys automatically | Full control via Key Vault APIs (create, import, disable, enable, delete, rotate, manage versions) | Full control via Key Vault APIs (HSM-backed) | Full control via Managed HSM APIs. Customer also controls the **root of trust** (Security Domain). |
| **Key Rotation** | Platform-managed — automatic, on a schedule not configurable by the customer | Customer-managed — manual rotation or configurable auto-rotation policies | Customer-managed — manual rotation or configurable auto-rotation policies | Customer-managed — manual rotation or configurable auto-rotation policies. |
| **Access Control / Data-Plane RBAC** | N/A — handled by Azure | **Azure RBAC** (or Key Vault access policies in legacy mode). Subscription Owners/Global Admins have full key access by default. | **Azure RBAC** (or Key Vault access policies). Subscription Owners/Global Admins have full key access by default. | **Local RBAC, isolated from Azure RBAC.** Subscription Owners, Global Admins, and even Microsoft have **zero** data-plane access by default. Roles include Managed HSM Crypto User (key mgmt), Crypto Officer (governance), Crypto Service Encryption User (services). |
| **Audit Logs** | Not exposed — no visibility into key usage, rotation, or access events | Full logging — all key operations logged via Azure Monitor / Key Vault diagnostic logs | Full logging — all key operations logged via Azure Monitor / Key Vault diagnostic logs | Full logging — detailed AuditEvent logs covering all key operations, Security Domain operations, and RBAC changes via Azure Monitor. |
| **Soft-Delete** | N/A | Optional — can be enabled/disabled | Optional — can be enabled/disabled | **Always on. Cannot be disabled.** Retention 7–90 days; HSM continues to accrue hourly charges while soft-deleted. |
| **Purge Protection** | N/A | Optional — can be enabled later, can be disabled if not yet triggered | Optional — can be enabled later, can be disabled if not yet triggered | Opt-in at creation. **Once enabled, cannot be disabled or overridden by anyone, including Microsoft.** Required by SQL MI as a precondition. |
| **Root of Trust (Security Domain)** | N/A | N/A | N/A | **Customer-controlled Security Domain** — encrypted with M-of-N customer RSA key pairs (recommended posture: 3-of-5). Without quorum, all keys are unrecoverable, even by Microsoft. |
| **Compliance Requirements** | Baseline — meets standard Azure encryption-at-rest compliance (service-level default) | FIPS 140-2 Level 1 (software-protected keys) | FIPS 140-2 Level 3 / Level 2 | **FIPS 140-3 Level 3, PCI DSS, PCI 3DS, PCI DSS 4.0.** Explicitly designed for PCI-compliant deployments (banking, ecommerce, regulated industries). |
| **Revocation Capability** | None — customer cannot revoke platform keys; only option is to disable the entire service or resource | Yes — customer can disable or delete keys in Key Vault at any time, revoking encryption access | Yes — customer can disable or delete HSM-protected keys in Key Vault at any time | Yes — customer can disable or delete keys in Managed HSM. **Microsoft has zero access to key material; revocation is fully under customer control.** |
| **Confidential Compute Envelope** | N/A | No | No | **Yes** — control plane runs inside a Confidential Compute envelope so datacenter operators cannot inspect runtime memory. |
| **Use Cases** | Default encryption at rest for all Azure services; suitable when regulatory requirements do not mandate customer key control | Encryption at rest, CMK, custom application encryption. Recommended for early-stage startups and general workloads | Encryption at rest, CMK, custom application encryption. Recommended for production cloud-native apps needing HSM-backed keys | Encryption at rest, TLS offload, CMK, custom workloads. Recommended for **banking, government, healthcare, and highly regulated industries** requiring key sovereignty and single tenancy. |
| **Cost (Typical Azure Pricing)** | No direct cost — included as part of the Azure service | Low — per-transaction pricing only (e.g., $0.03 per 10,000 operations for standard keys) | Moderate — per-key fee (e.g., $1/key/month for RSA 2048) + per-operation fee + optional rotation charges | Higher — **fixed hourly rate for dedicated HSM pool: ~$3.20/hour ≈ $2,304/month per region**. No separate per-key charges. Multi-region replication = 2× pool cost. |
| **Availability Across Azure Services** | Universal — enabled by default across all Azure services for storage and data encryption; no configuration required | Broad — most Azure PaaS/SaaS services support CMK with Key Vault Standard keys | Broad — most services supporting Key Vault CMK also accept HSM-protected keys from Premium tier interchangeably | **Broad and growing** — Azure Storage, Azure SQL DB / SQL MI, Cosmos DB, Managed Disks (via DES), App Service, Service Bus, Event Hubs, M365 Customer Key, Azure Information Protection, Databricks, and others. Parity with KV Premium for the most common CMK scenarios. |
| **Integration with BYOK / HYOK** | No BYOK / HYOK — customer cannot bring or hold keys in this model | BYOK supported — customer can import their own keys into Key Vault (Microsoft hosts keys on software-protected infrastructure) | BYOK supported — customer can import their own keys into Key Vault HSM (Microsoft hosts multi-tenant HSM hardware) | BYOK supported — customer can import keys into their dedicated HSM cluster. **HYOK-like sovereignty** — customer exclusively controls the Security Domain (root of trust); Microsoft has zero key access. |
| **Geo-Replication of Keys** | Transparent — handled internally by the Azure service; customer cannot configure cross-region key placement | Automatic — built-in cross-region disaster recovery managed by Microsoft | Automatic — built-in cross-region disaster recovery managed by Microsoft | **Opt-in, then fully automatic active-active.** Customer runs `az keyvault region add` once to extend the HSM to a paired region. After that, keys, RBAC, and policies sync automatically behind a **single global FQDN** (`<hsm>.managedhsm.azure.net`) with platform-managed routing (DNS TTL ~5 s). **No customer-operated Traffic Manager.** |
| **Backup & Restore of Keys** | N/A — not applicable or accessible to customer | Built-in service backup — Key Vault provides backup/restore of keys to encrypted file | Built-in service backup — Key Vault provides backup/restore of HSM-protected keys to encrypted file | **Service-managed backup to Azure Storage** via User-Assigned Managed Identity. Restore requires the **matching Security Domain**. The Security Domain itself must be backed up offline separately (it is **not** included in the service backup). |
| **Price Range** | Free | Low ($) | Medium ($$) | High ($$$) — dedicated hardware |

### What changed vs the original table

| Cell | Original | Corrected | Why |
|---|---|---|---|
| Key Ownership | "Customer (Dedicated HSM)" | "Customer — single-tenant Managed HSM cluster. *Not the legacy Azure Dedicated HSM service.*" | "Azure Dedicated HSM" is a separate, older bare-metal SafeNet Luna service. Conflating the two is a common source of confusion. |
| Geo-Replication | "Manual — cross-region DR requires customer action" | "Opt-in, then automatic active-active behind a single FQDN with platform-managed routing" | One opt-in command, then fully automatic. The "manual" label belongs to KV Standard/Premium GRS failover, not Managed HSM. |
| Availability Across Azure Services | "Core services" | Expanded list (Storage, SQL MI, Cosmos, Disks, App Service, Service Bus, Event Hubs, M365 Customer Key, AIP, Databricks…) | "Core services" understates current coverage and parity with KV Premium. |
| Data-Plane RBAC | *(missing row)* | Added — local RBAC isolated from Azure RBAC; Owners/Global Admins have zero data-plane access by default | The #1 operational surprise vs Standard/Premium and the most-frequently-misconfigured area. |
| Soft-Delete | *(missing row)* | Added — always on, cannot be disabled for Managed HSM | Hard architectural difference from KV Standard/Premium and a billing gotcha. |
| Purge Protection | *(missing row)* | Added — once enabled, cannot be disabled by anyone including Microsoft; required by SQL MI | Mandatory for SQL MI integration and the irreversibility is unique to Managed HSM. |
| Security Domain | *(missing row)* | Added — customer-controlled M-of-N RSA key quorum, no Microsoft recovery path | The Security Domain is the defining feature of Managed HSM and has no equivalent in Standard/Premium. |
| Confidential Compute | *(missing row)* | Added — control plane runs inside Confidential Compute envelope | Differentiator from Standard/Premium; explains why Microsoft operators have zero key access. |
| Object Types | *(missing row)* | Added — keys only (no secrets, no certificates) | Often surprises teams migrating from Premium; they need a separate Key Vault for secrets/certs. |

