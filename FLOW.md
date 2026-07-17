# Eternal Fan CRM — Flow & Interconnections

Clear picture of how the CRM connects (from `devhandoff.txt`, Figma, and client answers in `questions.txt`).

**Last aligned:** answers through **16-07-2026** in `questions.txt`.

---

## 0. One-screen mental model

```text
User (Admin / Standard)
    └── works in CRM screens
            ├── Dashboard          ← reads Account + Phase + Reminder + Activity
            ├── Accounts           ← list (filters/badges on LATEST phase; no nav counter)
            ├── Pipeline           ← same status rules as Accounts (LATEST phase)
            ├── Archived Deals     ← archived phases only (Closed Lost / Canceled *)
            ├── Alerts & Reminders ← creator-only reminders (+ due count in nav)
            ├── Search             ← global + results page
            ├── Members            ← Admin only (invite / remove)
            └── Settings           ← profile (initials avatar) + password

Account (one property / venue)
    ├── Contacts
    ├── Documents (Contracts per phase; MMR docs shared)
    ├── Activity timeline (system blue + MANUAL yellow)
    ├── Reminders (visible only to creator)
    └── Phases (max 5)  ← each has Stage + Financial calculator + % to Close
            └── only ONE active (non-closed) pursuit phase at a time
```

### Terminology (avoid mixing these up)

| Term | Meaning |
|------|---------|
| **Stage** | Editable pipeline field on a phase (Prospect → … → Closed Won, or archive reasons) |
| **Status** | Derived UI only — list tabs, row badges, Pipeline filters (not a separate DB field) |
| **Active tab** (Accounts/Pipeline) | Filter: NOT Past 100 Days AND NOT Closed Won (can include Needs Follow-Up) |
| **Active pursuit phase** | Only one phase at a time that is not Closed Won / not archived |
| **Active phase** (lifecycle) | Non-archived phase (Prospect through Closed Won) — includes Closed Won phases on the account |

---

## Module index (for team walkthroughs)

Jump to the numbered sections below for full detail.

| Module | See section | Key rules |
|--------|-------------|-----------|
| Auth & roles | **4** — Screen map | Admin vs Standard; no public sign-up |
| Members | **4** — Screen map | Admin invite/remove; hidden from Standard |
| Accounts list | **3** — Status & reminders | Tabs/badges on **latest** phase; filters parse address |
| Create / Edit account | **1** + **5** | Manual financial model; Places address; duplicate names blocked |
| Opportunity / Phases | **2** — Phase lifecycle | Max 5; Prospect start; one pursuit phase; Closed Won terminal |
| Financial calculator | **5** — Financial model | Pro vs College; Misc = expense sum; MMR fee on Phase 1 only |
| Documents / MMR | **1** — Domain map | MMR docs shared; Contracts per phase |
| Activity | **3** — Status & reminders | System blue vs manual yellow; reminder complete = manual |
| Reminders | **3** — Status & reminders | Creator-only; same feature from Account or Alerts page |
| Pipeline | **3** — Status & reminders | Same status rules as Accounts |
| Archived Deals | **2** — Phase lifecycle | Archived stages only |
| Dashboard | **3** + **4** | Past 100, reminder strip, top pursuits |
| Settings | **1** — Domain map | Initials avatar; password UI (no plain-text display) |

---

## 1. Domain map — how objects connect

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : "Account Lead (internal)"
    USER ||--o{ REMINDER : "creates / owns (creator-only)"
    USER ||--o{ ACTIVITY : "manual log / actor"

    ACCOUNT ||--|{ PHASE : "has 1..5"
    ACCOUNT ||--o{ CONTACT : "has"
    ACCOUNT ||--o{ DOCUMENT : "has"
    ACCOUNT ||--o{ ACTIVITY : "timeline"
    ACCOUNT ||--o{ REMINDER : "linked to"
    ACCOUNT ||--o| FINANCIAL_MODEL : "manual Pro or College"
    ACCOUNT ||--o| MMR_RIGHTS_FEE : "one-time account-level"

    PHASE ||--|| FINANCIALS : "calculator per phase"
    PHASE ||--o{ EXPENSE : "expense log → Misc Fees"
    PHASE ||--o{ DOCUMENT : "Contracts tagged to phase"
    PHASE }o--|| STAGE : "pipeline stage"

    ACCOUNT {
        string name UK
        string vertical
        string classification
        string financial_model
        string account_lead_internal_or_external
        string address_places
        date last_activity
    }

    PHASE {
        int number
        string stage
        int percent_to_close
        date est_close
    }

    REMINDER {
        date due_date
        string note
        bool completed
        date completed_at
    }

    ACTIVITY {
        string type_system_or_manual
        string message
        datetime at
    }
```

### Rules (confirmed)

| Rule | Detail |
|------|--------|
| Account | One property (e.g. University of Arkansas); **duplicate names blocked** |
| Account Lead | **Internal** = CRM member dropdown; **External** = free-text consultant name |
| Phase | Follow-on engagement; hard max **5**; start at **Prospect** (no Unstarted) |
| Active pursuit phase | Only **one** non-closed phase at a time |
| Add Phase N | Only after previous phase is **Closed Won**; create account always starts at **Phase 1** |
| Financial Model | **Manual** on create/edit (Pro or College). **No** auto-logic from Classification (pivot **16-07-2026**) |
| Address | Single Google Places field; list filters (Country / State / Domestic) **parse from address** |
| Vendor address | Same pattern — **one** address input |
| Dropdowns | **Fixed** for v1 — client requests changes through dev |
| Misc Fees | Sum of expense log (no manual Misc field) |
| MMR docs | Shared across phases; **Contracts** have Associated Phase |
| MMR Rights Fee | **One-time, account-level**; deducted from **Phase 1** profit only (not later phases) |
| Avatar | Initials only (no profile photo upload) |
| Nav counters | Alerts may show due count; **Accounts has no counter** |
| Delete | **Admin only** (accounts, documents, etc.) |

---

## 2. Phase lifecycle (Stage state machine)

**Stage** is the sales pipeline field. **Status** badges/tabs are derived from stage + reminders + last activity (see §3).

```mermaid
stateDiagram-v2
    [*] --> Prospect: Create Phase 1 or add after Closed Won

    Prospect --> Interest
    Interest --> Meeting
    Meeting --> OnSiteMeeting: On-site Meeting
    OnSiteMeeting --> Money
    Money --> Decision
    Decision --> Contract
    Contract --> ClosedWon: Closed Won

    Prospect --> Archived: Archive
    Interest --> Archived: Archive
    Meeting --> Archived: Archive
    OnSiteMeeting --> Archived: Archive
    Money --> Archived: Archive
    Decision --> Archived: Archive
    Contract --> Archived: Archive

    ClosedWon --> [*]: Terminal
    Archived --> [*]: Off active pipeline

    note right of Archived
      Closed Lost
      Canceled By Client
      Canceled By EF
      → Archived Deals list
    end note

    note right of ClosedWon
      Stays on account / Master Total
      Cannot revert or archive
      Account List: All Accounts tab only
      (not Active tab)
    end note

    note right of Prospect
      Full financial calculator
      visible from day 1
    end note
```

### Multi-phase sequence

```mermaid
sequenceDiagram
    participant U as User
    participant A as Account
    participant P1 as Phase 1
    participant P2 as Phase 2

    U->>A: Create Account + choose Financial Model
    A->>P1: Create Phase 1 (Stage = Prospect)
    Note over P1: Only active pursuit phase<br/>MMR Rights Fee hits Phase 1 profit

    U->>P1: Move stages → Closed Won
    Note over P1: Terminal for Phase 1<br/>Cannot archive from Closed Won<br/>Still on account / Master Total

    U->>A: Add Phase (allowed now)
    A->>P2: Create Phase 2 (Stage = Prospect)
    Note over P1,P2: P1 Closed Won + P2 active pursuit<br/>Never two active pursuit phases<br/>MMR fee not re-deducted on P2+
```

### Opportunity UI defaults

- Opening an account lands on **highest / most recently added** phase.
- Master Account Total shows when **2+ phases** exist; rollup = **non-archived** phases only (includes Closed Won).

---

## 3. Status, reminders & activity (cross-module)

Evaluated on the account’s **latest** phase unless noted.

```mermaid
flowchart TB
    subgraph Inputs
        R[Reminder]
        ACT[Activity entry]
        ST[Phase Stage]
    end

    subgraph Derived["Derived status (LATEST PHASE)"]
        FD["Follow-Up Due\nreminder due TODAY"]
        OV["Overdue\nreminder past due, not done"]
        NF["Needs Follow-Up\n= Follow-Up Due OR Overdue"]
        P100["Past 100 Days\nLast Activity > 100 days"]
        ACTV["Active tab\nNOT Past 100 Days\nAND NOT Closed Won"]
        CW["Closed Won\nlatest phase stage"]
        BADGE["Row badge priority\nFollow-Up Due / Overdue\n> Past 100 Days"]
    end

    subgraph Surfaces
        AL[Account List tabs + badges]
        PL[Pipeline — same rules]
        DB[Dashboard Past 100 / Reminder strip]
        AR[Alerts & Reminders page]
        TL[Account Activity timeline]
    end

    R --> FD
    R --> OV
    FD --> NF
    OV --> NF
    ACT --> P100
    ST --> ACTV
    ST --> CW
    NF --> BADGE
    P100 --> BADGE

    NF --> AL
    P100 --> AL
    ACTV --> AL
    CW --> AL
    BADGE --> AL
    NF --> PL
    P100 --> PL
    ACTV --> PL
    CW --> PL
    P100 --> DB
    R --> AR
    R -->|Mark Done| ACT
    ACT --> TL
```

### Account List / Pipeline status rules

| Tab / concept | Rule (on **latest** phase) |
|---------------|----------------------------|
| **All Accounts** | Latest phase is **not** archived (includes Closed Won) |
| **Active** | NOT Past 100 Days AND NOT Closed Won (includes Needs Follow-Up) |
| **Needs Follow-Up** | Follow-Up Due **OR** Overdue |
| **Past 100 Days** | Last activity more than 100 days ago |
| **Closed Won** | Latest phase stage = Closed Won; **All Accounts** tab only (not Active) |
| **Follow-Up Due** | Creator’s open reminder due **today** |
| **Overdue** | Creator’s open reminder past due |
| **Badge priority** | Follow-Up Due / Overdue **>** Past 100 Days |
| Pipeline | **Same** status rules as Accounts |

> Reminder-driven status uses the **creator’s** reminders (creator-only visibility). Other users do not see those reminders but status derivation follows the same rules for the account’s latest phase context.

### Reminder feature (one feature, two entry points)

```mermaid
flowchart LR
    A[Account page\nSet Reminder] -->|pre-fills account| M[Reminder record\ncreator-only]
    B[Alerts & Reminders page\nCreate Reminder] -->|user picks account| M
    M --> Buckets[Overdue / Today / Upcoming]
    M -->|Mark Done| Done[Completed this month]
    Done -->|same day Undo| Buckets
    Done -->|creates MANUAL Activity\nyellow — same reminder text| Timeline[Activity Timeline]
    Buckets -->|Undo removes related Activity| Timeline
```

| Reminder rule | Detail |
|---------------|--------|
| Same feature | Account pre-fills; Alerts page selects account |
| Visibility | **Creator only** — not shared with team |
| Completed list | **Current calendar month** only |
| Older completed | Hidden from Alerts after month; **still on Activity Timeline** (unless undone) |
| Undo | **Same day** as marked done; restores to due-date bucket |
| Activity on complete | Creates **manual** (yellow) Activity with **same string** as the reminder |
| Activity on undo | Related Activity entry is **removed** |

### What writes to Activity / Last Activity

| Auto (system blue) | Manual (brand yellow) |
|--------------------|------------------------|
| Meeting notes added | **Log Activity** button |
| Stage changes | **Reminder completions** (same reminder text) * |
| Document uploads (incl. MMR) | |
| Phase additions | |
| Contact additions | |

\* Reminder completion logging as **manual/yellow** confirmed **16-07-2026** (supersedes earlier “auto-log” wording).

`Last Activity` drives **Past 100 Days** (account / latest-phase evaluation).

---

## 4. Screen map — who reads / writes what

```mermaid
flowchart TB
    subgraph Write["User writes"]
        CA[Create / Edit Account]
        OP[Opportunity — stage + financials]
        DOC[Documents upload]
        REM[Reminders]
        LOG[Log Activity / Meeting Notes]
        MEM[Members — Admin only]
        SET[Settings — profile / password]
    end

    subgraph Core["Core data"]
        ACC[(Account + Phases)]
        FIN[(Financials + Expenses + MMR fee)]
        RM[(Reminders)]
        AT[(Activity)]
        US[(Users / Roles)]
    end

    subgraph Read["User reads"]
        LIST[Accounts list]
        PIPE[Pipeline]
        ARCH[Archived Deals]
        DASH[Dashboard]
        ALERTS[Alerts & Reminders]
        SRCH[Search]
        TABS[Account tabs]
    end

    CA --> ACC
    OP --> ACC
    OP --> FIN
    DOC --> ACC
    REM --> RM
    LOG --> AT
    MEM --> US
    SET --> US

    ACC --> LIST
    ACC --> PIPE
    ACC --> ARCH
    ACC --> TABS
    ACC --> SRCH
    FIN --> OP
    FIN --> DASH
    RM --> ALERTS
    RM --> LIST
    RM --> PIPE
    RM --> DASH
    AT --> TABS
    AT --> LIST
    AT --> DASH
```

### Roles (quick)

| Role | Can |
|------|-----|
| **Admin** | Full access + permanent delete + **Members** page |
| **Standard** | Day-to-day CRM work; **no delete**; **no Members** page |

Members: Admin invite (email + role) → claim link / set password; remove member. No public sign-up. Confirmed **15-07-2026**.

---

## 5. Vertical, Classification & Financial Model

```mermaid
flowchart LR
    V[Vertical] --> C[Classification]
    U[User on Create / Edit] -->|required choice| FM[Stored Financial Model\nPro or College]
    C -.->|no auto-trigger| FM
    FM --> CALC[Phase financial calculator]
    MMR[MMR Rights Fee\naccount-level] -->|deduct Phase 1 only| CALC
    EXP[Expense log] -->|sum| MISC[Misc Fees logged]
    MISC --> CALC
```

| Rule | Detail |
|------|--------|
| Vertical | Broad category (Sports, Lifestyle, …) — used for Classification cascade |
| Classification | Nested under Vertical (College Sports, Pro Football, …) — **does not** set financial model |
| Financial Model | User **must choose** Pro or College on create; **full control** regardless of Classification (**16-07-2026**) |
| Switching models | User may switch Pro ↔ College on edit; switching **to** College defaults Revenue Share % to **40%** |
| Schema | Always store Financial Model on the account (never derive-only from Classification) |

> **Pivot note:** Earlier answers described auto-trigger from Classification. Client confirmed **16-07-2026** that this is removed — manual selection only.

