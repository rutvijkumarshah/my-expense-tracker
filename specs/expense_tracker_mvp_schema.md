# Expense Tracker MVP — Schema & Architecture Plan

## Project Context

Android app built **from scratch** with goals:
- Learn Jetpack Compose
- Learn Coroutines & Flow
- Learn Room + Firestore offline-first sync

Stack: **Kotlin + Jetpack Compose + Room + Firebase (Auth + Firestore) + WorkManager**

---

## Core Design Principles

- **Room** is the single source of truth — UI only reads from Room, never Firestore directly
- **Firestore** is remote backup/sync only
- All IDs are **client-generated UUIDs** (not auto-increment) — works offline
- All money stored as **Long (cents)** — never Float/Double (e.g. $12.50 → `1250`)
- **Soft deletes** everywhere — `isDeleted` flag instead of actual row deletion
- `syncStatus` enum drives background sync via WorkManager

---

## Room Schema (Local Database)

### Sync Status Enum (shared across all tables)
```
SYNCED           — local matches remote
PENDING_UPLOAD   — created/edited offline, not yet pushed
PENDING_DELETE   — soft-deleted locally, needs remote delete
```

---

### Table: `users`
Anchors Firebase Auth identity locally.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID (PK) | Matches Firebase Auth UID |
| `email` | String | |
| `displayName` | String | |
| `createdAt` | Long (epoch ms) | |
| `lastSyncedAt` | Long? | Null until first sync |

---

### Table: `accounts`
Represents where money lives (cash, bank, credit card).

| Field | Type | Notes |
|---|---|---|
| `id` | UUID (PK) | |
| `userId` | UUID (FK → users) | |
| `name` | String | e.g. "Chase Checking", "Wallet" |
| `type` | Enum | `CASH`, `BANK`, `CREDIT_CARD`, `SAVINGS` |
| `currency` | String | Default `"USD"` |
| `initialBalance` | Long (cents) | |
| `color` | String | Hex color |
| `iconName` | String | Material icon name |
| `isDefault` | Boolean | Pre-selected on new transaction |
| `isArchived` | Boolean | Soft hide |
| `createdAt` | Long | |
| `updatedAt` | Long | |
| `syncStatus` | Enum | 3-state sync enum |

> **MVP UI note:** Seed one default `"My Wallet"` account on first launch. Hide accounts from UI entirely for MVP. Schema stays correct, UI stays simple.

---

### Table: `categories`
Seeded with defaults on first launch. User can add custom ones.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID (PK) | |
| `userId` | UUID? | Null = system/default category |
| `name` | String | e.g. "Food & Dining" |
| `iconName` | String | Material icon key |
| `color` | String | Hex |
| `type` | Enum | `EXPENSE`, `INCOME`, `BOTH` |
| `isDefault` | Boolean | System-seeded = true |
| `sortOrder` | Int | Manual ordering |
| `isArchived` | Boolean | |
| `createdAt` | Long | |
| `updatedAt` | Long | |
| `syncStatus` | Enum | |

---

### Table: `transactions` ← Heart of the app

| Field | Type | Notes |
|---|---|---|
| `id` | UUID (PK) | |
| `userId` | UUID (FK → users) | |
| `accountId` | UUID (FK → accounts) | |
| `categoryId` | UUID (FK → categories) | |
| `type` | Enum | `EXPENSE`, `INCOME` |
| `amountCents` | Long | Always positive; type determines sign |
| `currency` | String | Copied from account at insert time |
| `note` | String? | Optional description |
| `transactionDate` | Long | User-chosen date (NOT createdAt) |
| `createdAt` | Long | Row insert time |
| `updatedAt` | Long | Last edit time |
| `isDeleted` | Boolean | Soft delete flag |
| `deletedAt` | Long? | When soft-deleted |
| `syncStatus` | Enum | |
| `remoteId` | String? | Firestore doc ID if it differs |

> **`transactionDate` vs `createdAt`:** Users frequently log expenses retroactively. These must be separate fields.

---

### Table: `budgets`
Per-category monthly spending limits.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID (PK) | |
| `userId` | UUID (FK → users) | |
| `categoryId` | UUID (FK → categories) | |
| `amountCents` | Long | Monthly limit |
| `period` | Enum | `MONTHLY` only for MVP |
| `startDate` | Long | First day of applicable period |
| `isActive` | Boolean | |
| `createdAt` | Long | |
| `updatedAt` | Long | |
| `syncStatus` | Enum | |

---

## Firestore Schema (Remote)

### Collection Structure
```
/users/{userId}/
    /transactions/{transactionId}
    /accounts/{accountId}
    /categories/{categoryId}       ← user-created only
    /budgets/{budgetId}

/default_categories/{categoryId}   ← top-level, global, read-only
```

### Key differences from Room
| Room | Firestore | Why |
|---|---|---|
| `syncStatus` | dropped | Local-only concept |
| `remoteId` | dropped | Firestore doc ID IS the UUID |
| `Long` epoch ms | `Timestamp` | Native Firestore type |
| `userId` field in rows | dropped | Path encodes ownership |

### Security Rules
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null
                         && request.auth.uid == userId;
    }

    match /default_categories/{docId} {
      allow read: if request.auth != null;
      allow write: if false;
    }
  }
}
```

### Document Shapes

**`/users/{userId}`**
```json
{
  "email": "string",
  "displayName": "string",
  "createdAt": "Timestamp",
  "lastSyncedAt": "Timestamp"
}
```

**`/users/{userId}/transactions/{txId}`**
```json
{
  "accountId": "uuid-string",
  "categoryId": "uuid-string",
  "type": "EXPENSE | INCOME",
  "amountCents": 1250,
  "currency": "USD",
  "note": "string | null",
  "transactionDate": "Timestamp",
  "createdAt": "Timestamp",
  "updatedAt": "Timestamp",
  "isDeleted": false,
  "deletedAt": "Timestamp | null"
}
```

**`/users/{userId}/categories/{categoryId}`** (user-created only)
```json
{
  "name": "string",
  "iconName": "string",
  "color": "#hex",
  "type": "EXPENSE | INCOME | BOTH",
  "sortOrder": 1,
  "isArchived": false,
  "createdAt": "Timestamp",
  "updatedAt": "Timestamp"
}
```

**`/default_categories/{categoryId}`**
```json
{
  "name": "Food & Dining",
  "iconName": "restaurant",
  "color": "#FF5722",
  "type": "EXPENSE",
  "sortOrder": 1
}
```

### Seeded Default Categories
| Document ID | name | iconName | color | type | sortOrder |
|---|---|---|---|---|---|
| `food_dining` | Food & Dining | restaurant | #FF5722 | EXPENSE | 1 |
| `transport` | Transport | directions_car | #2196F3 | EXPENSE | 2 |
| `shopping` | Shopping | shopping_bag | #9C27B0 | EXPENSE | 3 |
| `bills` | Bills & Utilities | receipt_long | #607D8B | EXPENSE | 4 |
| `health` | Health | favorite | #E91E63 | EXPENSE | 5 |
| `entertainment` | Entertainment | movie | #FF9800 | EXPENSE | 6 |
| `salary` | Salary | payments | #4CAF50 | INCOME | 7 |
| `other_expense` | Other | category | #9E9E9E | EXPENSE | 8 |

---

## Sync Architecture

```
User Action
    ↓
Room (write immediately)  ← source of truth
    ↓
Mark syncStatus = PENDING_UPLOAD
    ↓
WorkManager SyncWorker (background)
    ↓
Firestore (remote backup)
    ↓
Mark syncStatus = SYNCED
```

**Why `isDeleted` exists in Firestore:**
If device A deletes while device B is offline, device B needs to see the tombstone when it reconnects. Missing document ≠ deleted in distributed systems.

---

## What This Teaches (Learning Goals per Feature)

| Feature | Concept Learned |
|---|---|
| `Flow<List<Transaction>>` from Room DAO | Core Compose + Flow reactive pattern |
| Budget spent vs limit query | `combine()` and `map()` on multiple Flows |
| syncStatus → WorkManager job | Real coroutines with `withContext(Dispatchers.IO)` |
| Soft deletes + `isDeleted` filter | Non-trivial WHERE clauses, state management |
| Cents as Long | Kotlin value classes (`@JvmInline value class Money`) |
| UUID PKs generated on device | Offline-first architecture thinking |
| Categories from Room vs Firestore | Repository pattern, single source of truth |

---

## Intentionally Left Out of MVP
- Recurring transactions
- Multi-currency conversion
- Receipt photo attachments
- Sub-categories
- Split transactions
- Tags / labels
- Export to CSV

---

## Firebase Setup Checklist
- [x] Firebase project created (Google Analytics disabled)
- [x] Android app registered with correct package name
- [x] `google-services.json` placed in `app/` folder
- [x] Email/Password Auth enabled
- [x] Firestore created (started in test mode)
- [x] Security rules published
- [x] `default_categories` seeded via Node.js script

## .gitignore Entries (Critical)
```
google-services.json
serviceAccount.json
*.jks
.gradle/
local.properties
```

---

## Next Steps (Build Order)
1. Android Studio project setup + dependencies
2. Room entities + DAOs
3. Repository layer
4. ViewModels + UI State
5. Compose screens (Transaction list → Add transaction → Categories → Budget)
6. Firebase Auth (Login/Register screens)
7. Firestore sync + WorkManager
