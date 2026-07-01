# Rule-Based Fraud Detection System
**Tech Stack:** MySQL | SQL Triggers | Stored Procedures

A database-level fraud detection system built as part of a DBMS portfolio project. The goal was to implement fraud-detection logic entirely inside the database — no application layer needed — using triggers and stored procedures that fire automatically on every transaction.

---

## What It Does

Every time a transaction is inserted, an `AFTER INSERT` trigger automatically calls a stored procedure that runs 4 fraud-detection rules. If any rule fires, it logs a flag in the `FraudFlags` table and updates the user's risk score — all in real time, without any external code.

---

## Schema (6 Tables)

Users → Accounts → Transactions
                       ↓
Users → Devices ───────┘   (used to detect new device activity)
Accounts → Cards → Transactions
Transactions → FraudFlags

| Table | Purpose |
|---|---|
| `Users` | Stores user info and a running `risk_score` |
| `Accounts` | Each user's bank account with balance and status |
| `Cards` | Cards linked to accounts (hashed number, no plain text) |
| `Devices` | Tracks known devices per user via fingerprint |
| `Transactions` | Every transaction record — the core of the system |
| `FraudFlags` | Log of which rule fired, severity, and review status |

> **Why 6 tables?** The `Devices` table is what makes Rule 3 (new device + high value) possible. Without it, there's no way to know if a device has been seen before. It's not padding — it's load-bearing.

---

## Fraud Detection Rules

### Rule 1 — Velocity Check *(severity: medium)*
If the same account makes more than 3 transactions within 10 minutes, it gets flagged. Rapid repeated transactions are a common pattern in stolen card abuse.

### Rule 2 — Amount Anomaly *(severity: high)*
If a transaction is more than 5× the account's average transaction amount, it's flagged. A sudden spike in spending from a normally low-activity account is suspicious.

### Rule 3 — New Device + High Value *(severity: high)*
If a device has never been used before AND the transaction amount exceeds ₹10,000, it gets flagged. First-time device doing a big transaction is a common account-takeover signal.

### Rule 4 — Structuring Pattern *(severity: low)*
Transactions between ₹9,000–₹9,999 are flagged. "Structuring" is a real AML (Anti-Money Laundering) tactic where people deliberately keep amounts just under reporting thresholds.

---

## How It Works (Flow)

INSERT INTO Transactions (...)
        ↓
trg_after_transaction_insert  [AFTER INSERT trigger]
        ↓
EvaluateFraudRules(txn_id)    [stored procedure]
        ↓
Run Rule 1 → flag if triggered
Run Rule 2 → flag if triggered
Run Rule 3 → flag if triggered
Run Rule 4 → flag if triggered
        ↓
UPDATE Users.risk_score += (flags raised × 10)


## Sample Results

**Flagged Transactions**

| User | Amount | Rule | Severity |
|---|---|---|---|
| Aarav Sharma | ₹700 | Velocity Check | medium |
| Aarav Sharma | ₹600 | Velocity Check | medium |
| Aarav Sharma | ₹800 | Velocity Check | medium |
| Riya Mehta | ₹9,800 | Structuring Pattern | low |
| Karan Patel | ₹25,000 | New Device High Value | high |

**Risk Score Leaderboard**

| User | Risk Score |
|---|---|
| Aarav Sharma | 30 |
| Riya Mehta | 10 |
| Karan Patel | 10 |

**Rule Frequency**

| Rule | Times Triggered |
|---|---|
| Velocity Check | 3 |
| Structuring Pattern | 1 |
| New Device High Value | 1 |

## Design Decisions

- **AFTER INSERT (not BEFORE INSERT):** The trigger runs after the row is committed so the `txn_id` exists and can be referenced in `FraudFlags`. A BEFORE INSERT trigger wouldn't have the ID yet.
- **Stored procedure over inline trigger logic:** Keeping the rules in a procedure makes each rule easy to find, test, and modify independently. Inline trigger logic would become unreadable fast.
- **Hashed card numbers:** The `Cards` table stores a hash, not the real number. Even in a demo project, storing plain card numbers is bad practice worth avoiding.
- **Risk score on Users, not Accounts:** Fraud risk belongs to the person, not the account. A user could have multiple accounts — the score should aggregate across all of them.


## What I'd Improve Next

- Add a geo-impossibility check (same card used in two cities within minutes)
- Move severity scoring to a weighted system instead of flat +10 per flag
- Add an `is_resolved` workflow to FraudFlags so analysts can mark cases closed


## Files
fraud_detection_project.sql   — full schema, procedure, trigger, and sample data
README.md                     — this file
