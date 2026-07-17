# Eternal Fan CRM — Flow & Interconnections

Clear picture of how the CRM connects (from `devhandoff.txt`, Figma, and client answers in `questions.txt`).

**How to view:** open this file in Cursor / GitHub / any Mermaid-capable preview.

**Last aligned:** answers through **16-07-2026** in `questions.txt`.

---

## 0. One-screen mental model

```text
User (Admin / Standard)
    └── works in CRM screens
            ├── Dashboard          ← reads Account + Phase + Reminder + Activity
            ├── Accounts           ← Account list (filters/badges on LATEST phase; no nav counter)
            ├── Pipeline           ← same status rules as Accounts (LATEST phase)
            ├── Archived Deals     ← archived phases only
            └── Alerts & Reminders ← Reminder tasks (creator-only; also create from Account)

Account (one property / venue)
    ├── Contacts
    ├── Documents (Contracts per phase; MMR docs shared)
    ├── Activity timeline (system blue + MANUAL yellow)
    ├── Reminders (visible only to creator)
    └── Phases (max 5)  ← each has Stage + Financial calculator + % to Close
            └── only ONE active (non-closed) phase at a time
```

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
| Phase | Follow-on engagement; hard max **5**; start at **Prospect** (no Unstarted) |
| Active phase | Only **one** non-closed phase at a time |
| Add Phase N | Only after previous phase is **Closed Won**; create account always starts at **Phase 1** |
| Financial Model | **Manual** on create/edit (Pro or College). **No** auto-logic from Classification |
| Address | Single Google Places field; list filters (Country / State / Domestic) **parse from address** |
| Vendor address | Same pattern — **one** address input |
| Misc Fees | Sum of expense log (no manual Misc field) |
| MMR docs | Shared across phases; **Contracts** have Associated Phase |
| MMR Rights Fee | **One-time, account-level**; deducted from **Phase 1** profit only (not later phases) |
| Avatar | Initials only (no profile photo upload) |
| Nav | Alerts may show due count; **Accounts has no counter** |

---

## 2. Phase lifecycle (Stage state machine)

```mermaid
stateDiagram-v2
    [*] --> Prospect: Create Account (Phase 1)\nor Add Phase after prior Closed Won

    Prospect --> Interest
    Interest --> Meeting
    Meeting --> OnSiteMeeting: On-site Meeting
    OnSiteMeeting --> Money
    Money --> Decision
    Decision --> Contract
    Contract --> ClosedWon: Closed Won

    Prospect --> Archived: Closed Lost /\nCanceled By Client /\nCanceled By EF
    Interest --> Archived
    Meeting --> Archived
    OnSiteMeeting --> Archived
    Money --> Archived
    Decision --> Archived
    Contract --> Archived

    ClosedWon --> [*]: Terminal — cannot move\nback OR to archive
    Archived --> [*]: Leaves active pipeline\n→ Archived Deals

    note right of ClosedWon
      Stays on account / Master Total
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
    Note over P1: Only active phase<br/>MMR Rights Fee hits Phase 1 profit

    U->>P1: Move stages → Closed Won
    Note over P1: Terminal for Phase 1<br/>Cannot archive from Closed Won<br/>Still on account / Master Total

    U->>A: Add Phase (allowed now)
    A->>P2: Create Phase 2 (Stage = Prospect)
    Note over P1,P2: P1 Closed Won + P2 active<br/>Never two active pursuit phases<br/>MMR fee not re-deducted on P2+
```

### Opportunity UI defaults

- Opening an account lands on **highest / most recently added** phase.
- Master Account Total shows when **2+ phases** exist; rollup = **non-archived** phases only (includes Closed Won).

---

## 3. Status, reminders & activity (cross-module)

This is where Accounts, Pipeline, Alerts, Activity, and Dashboard meet.

```mermaid
flowchart TB
    subgraph Inputs
        R[Reminder]
        ACT[Activity entry]
        ST[Phase Stage]
    end

    subgraph Derived["Derived on LATEST PHASE"]
        FD["Follow-Up Due\nreminder due TODAY"]
        OV["Overdue\nreminder past due, not done"]
        NF["Needs Follow-Up\n= Follow-Up Due OR Overdue"]
        P100["Past 100 Days\nLast Activity > 100 days"]
        ACTV["Active filter\nNOT Past 100 Days\nAND NOT Closed Won\n(includes Needs Follow-Up)"]
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
    NF --> BADGE
    P100 --> BADGE

    NF --> AL
    P100 --> AL
    ACTV --> AL
    BADGE --> AL
    NF --> PL
    P100 --> PL
    ACTV --> PL
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
| **Closed Won** | Shown under **All Accounts** only (not Active) |
| **Follow-Up Due** | Reminder due **today** |
| **Overdue** | Reminder past due, not completed |
| **Badge priority** | Follow-Up Due / Overdue **>** Past 100 Days |
| Pipeline | **Same** status rules; evaluated on account’s **latest** phase |

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
| Older completed | Leave Alerts “Completed this month”; remain on **Activity Timeline** (unless undone) |
| Undo | **Same day** as marked done; restores to due-date bucket |
| Activity on complete | Creates **manual** (yellow) Activity with **same string** as the reminder |
| Activity on undo | Related Activity entry is **removed** |

### What writes to Activity / Last Activity

| Auto (system blue) | Manual (brand yellow) |
|--------------------|------------------------|
| Meeting notes added | **Log Activity** button |
| Stage changes | **Reminder completions** (same reminder text) |
| Document uploads (incl. MMR) | |
| Phase additions | |
| Contact additions | |

`Last Activity` drives **Past 100 Days** (account / latest-phase evaluation per client).

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
        TABS[Account tabs]
    end

    CA --> ACC
    OP --> ACC
    OP --> FIN
    DOC --> ACC
    REM --> RM
    LOG --> AT
    MEM --> US

    ACC --> LIST
    ACC --> PIPE
    ACC --> ARCH
    ACC --> TABS
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
| **Admin** | Full access + permanent delete + Members page |
| **Standard** | Day-to-day CRM work; **no delete** |

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
| Vertical | Broad category (Sports, Lifestyle, …) |
| Classification | Nested under Vertical (College Sports, Pro Football, …) |
| Financial Model | User **must specify** Pro or College on create; **full control** regardless of Classification |
| Switching models | Allowed on edit; when moving to College, Revenue Share % defaults to **40%** (same as new college deal) |
| Schema | Always store Financial Model on the account (do not derive-only from Classification) |

---

## 6. Open items that still affect these flows

Track in `questions.txt` — do not invent in code:

1. **Revenue Share %** — client said “typically 40%”; confirm if always editable per deal after default  
2. **MMR Rights Fee UI placement** — fee is account-level / Phase 1 profit; confirm edit surface (account field vs Phase 1 calculator only; Phase 2+ read-only or hidden)

Everything else in §6 of the prior FLOW (reminder visibility, Needs Follow-Up OR, badge priority, Closed Won tabs/archive, classification auto-model) is **resolved** — see above.

---

## Related docs

| Doc | Purpose |
|-----|---------|
| `devhandoff.txt` | Business + financial formulas |
| `questions.txt` | Client Q&A (incl. 15-07 / 16-07 answers) |
| `PLAN.md` | Build phases / stack |
| `estimation/Eternal-Fan-CRM-Timeline.xlsx` | Timeline for team |
