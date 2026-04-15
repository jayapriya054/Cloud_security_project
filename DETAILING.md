# Spiltwise — Project Detailing

## Overview

Spiltwise is a web-based expense splitting and payment tracking application built with Flask. It allows multiple users to share expenses, track who owes whom, settle debts, and manage a personal wallet. The app uses graph-based debt simplification to minimize the number of transactions needed to settle all balances within a group.

---

## Tech Stack

| Layer      | Technology                        |
|------------|-----------------------------------|
| Backend    | Python, Flask 3.0                 |
| Database   | SQLite (local) / PostgreSQL (AWS) |
| ORM        | Flask-SQLAlchemy 3.1              |
| Auth       | Flask-Bcrypt (password hashing), Flask sessions |
| Frontend   | Jinja2 templates, HTML, CSS, vanilla JavaScript |
| Config     | python-dotenv (.env file)         |

---

## Database Schema

### User
| Column       | Type    | Description                          |
|--------------|---------|--------------------------------------|
| id           | Integer | Primary key                          |
| name         | String  | Display name                         |
| email        | String  | Unique login email                   |
| password     | String  | Bcrypt hashed password               |
| avatar_color | String  | Hex color auto-assigned from email   |
| created_at   | DateTime| Account creation timestamp           |

### Expense
| Column      | Type    | Description                          |
|-------------|---------|--------------------------------------|
| id          | Integer | Primary key                          |
| description | String  | What the expense was for             |
| amount      | Float   | Total expense amount                 |
| category    | String  | Food, Travel, Rent, etc.             |
| split_type  | String  | equal or custom                      |
| paid_by     | FK      | User who paid upfront                |
| created_at  | DateTime| When expense was created             |

### ExpenseSplit
| Column     | Type    | Description                           |
|------------|---------|---------------------------------------|
| id         | Integer | Primary key                           |
| expense_id | FK      | Related expense                       |
| user_id    | FK      | User this split belongs to            |
| amount     | Float   | Amount this user owes for the expense |

### Payment
| Column     | Type    | Description                           |
|------------|---------|---------------------------------------|
| id         | Integer | Primary key                           |
| from_user  | FK      | Sender (null for wallet top-ups)      |
| to_user    | FK      | Receiver                              |
| amount     | Float   | Payment amount                        |
| note       | String  | Optional payment note                 |
| status     | String  | completed                             |
| created_at | DateTime| When payment was made                 |

### Wallet
| Column  | Type    | Description                            |
|---------|---------|----------------------------------------|
| id      | Integer | Primary key                            |
| user_id | FK      | One-to-one with User                   |
| balance | Float   | Current wallet balance                 |

### Junction Tables
- **expense_members** — many-to-many between Expense and User (who is part of the expense)
- **expense_settled** — many-to-many between Expense and User (who has settled their share)

---

## Features

### 1. Authentication
- User signup with name, email, and password
- Passwords hashed with Bcrypt before storage
- Session-based login/logout
- All routes protected with a custom `@login_required` decorator
- Auto-assigned avatar color based on email hash

### 2. Dashboard
- Financial snapshot showing:
  - Total amount owed to you
  - Total amount you owe others
  - Net balance (positive or negative)
  - Wallet balance
- Monthly spending bar chart (last 6 months)
- Spending breakdown by category
- 5 most recent expenses
- Top 3 spending partners
- Quick balance summary with all group members

### 3. Expense Management
- Add an expense with description, amount, category, paid-by, and members
- Two split modes:
  - **Equal split** — amount divided equally among all members
  - **Custom split** — manually enter each person's share
- Live member search by name or email (via `/api/users/search`)
- Expense list showing all expenses you are part of, sorted by date
- One-click settle button per expense (marks your share as settled)

### 4. Balances
- **Raw balances** — net amount owed between you and each person, factoring in all expenses and payments
- **Simplified Debts** — graph-optimized view showing the minimum transactions needed to settle everything (see algorithm below)

### 5. Payments
- Send a payment to any user directly from the Balances page
- Payment modal pre-fills the amount you owe
- Payment history showing all sent and received transactions
- Type shown as: Sent, Received, or Top-up
- Wallet balance is automatically updated on each payment

### 6. Wallet
- Each user has a wallet with a balance
- Top up wallet with preset amounts ($100, $200, $500, $1000) or custom amount
- Full transaction history with date, type, and counterparty
- Wallet balance deducted/credited automatically when payments are made

---

## Workflow

### New User Flow
1. User visits `/` → sees landing page
2. Signs up at `/signup` → account + wallet created
3. Redirected to `/dashboard`

### Adding an Expense
1. Click "Add Expense" from Dashboard or Expenses page
2. Enter description, amount, category
3. Select who paid
4. Search and add members
5. Choose equal or custom split
6. Submit → expense saved, splits recorded per member

### Settling a Debt
**Option A — Settle individual expense:**
- Go to Expenses page
- Click "Settle" on an expense row
- Your share is marked as settled for that expense

**Option B — Send a direct payment:**
- Go to Balances page
- Click "Pay Now" next to a person you owe
- Enter amount and note in the modal
- Submit → payment recorded, wallet balances updated

### Viewing Balances
1. Go to `/balances`
2. See Simplified Debts (graph-optimized, minimum transactions)
3. See All Balances table (raw net per person)
4. Use "Pay Now" or "Request" buttons inline

---

## Graph-Based Debt Simplification Algorithm

### Problem
In a group of N people, naive debt tracking leads to O(N^2) transactions. The goal is to reduce this to the minimum number of transactions.

### Example
```
Without simplification:
  A owes B $10
  B owes C $10
  = 2 transactions

With simplification:
  A pays C $10 directly
  B is automatically cleared
  = 1 transaction
```

### Algorithm (Greedy Min/Max Heap)
1. Compute the **net balance** for every user globally across all expenses and payments
2. Separate users into:
   - **Debtors** (net balance < 0) — pushed into a min-heap
   - **Creditors** (net balance > 0) — pushed into a max-heap
3. Repeatedly:
   - Pop the largest debtor and largest creditor
   - The debtor pays the creditor `min(debt, credit)`
   - Push back any remaining balance
4. Stop when all balances are settled
5. Filter the resulting transactions to show only those involving the current user

### Complexity
- Time: O(N log N) — heap operations
- Space: O(N) — one entry per user

This is the classic **"Optimal Account Balancing"** problem, a well-known graph/greedy algorithm used in real-world fintech applications.

---

## API Endpoints

| Method | Route                          | Description                        |
|--------|--------------------------------|------------------------------------|
| GET    | `/`                            | Landing page or redirect to dashboard |
| GET/POST | `/signup`                   | User registration                  |
| GET/POST | `/login`                    | User login                         |
| GET    | `/logout`                      | Clear session and logout           |
| GET    | `/dashboard`                   | Main dashboard                     |
| GET    | `/expenses`                    | List all expenses                  |
| GET/POST | `/expenses/add`             | Add a new expense                  |
| POST   | `/expenses/<id>/settle`        | Settle your share of an expense    |
| GET    | `/balances`                    | View balances and simplified debts |
| GET    | `/payments`                    | Payment history                    |
| POST   | `/payments/send`               | Send a payment to a user           |
| GET    | `/wallet`                      | Wallet page with transaction history |
| POST   | `/wallet/topup`                | Add money to wallet                |
| GET    | `/api/users/search?q=`         | Search users by name or email      |

---

## Configuration (.env)

```
SECRET_KEY=your_secret_key_here

# Local development (SQLite)
DATABASE_URL=sqlite:///spiltwise.db

# AWS production (PostgreSQL RDS)
# DATABASE_URL=postgresql://user:password@rds-endpoint:5432/spiltwise
```

---

## Running the Project

### Local
```bash
cd spiltwise
pip install -r requirements.txt
python app.py
```
App runs at `http://localhost:5000`. The SQLite database file (`spiltwise.db`) is created automatically on first run.

### AWS Deployment
- Set `DATABASE_URL` environment variable to your RDS PostgreSQL connection string
- Use a WSGI server (Gunicorn) instead of Flask's dev server
- No code changes required — SQLAlchemy handles both SQLite and PostgreSQL

---

## Architecture — Monolith with Microservice Boundaries

Spiltwise is currently built as a **monolithic Flask application**. All logic lives in `app.py` and shares a single SQLite/PostgreSQL database. However, the codebase is logically structured around clear service boundaries that map directly to independent microservices if the app were to be decomposed for scale.

### Current Architecture
```
Client (Browser)
      |
   Flask App (app.py)
      |
   SQLite / PostgreSQL (single DB)
```

### Logical Service Boundaries (Potential Microservices)

| Service              | Responsibility                                                                 | Current location in code                                      |
|----------------------|--------------------------------------------------------------------------------|---------------------------------------------------------------|
| **Auth Service**     | Signup, login, logout, session management, password hashing                    | `signup`, `login`, `logout` routes + `User` model            |
| **User Service**     | User profile, avatar color assignment, user search API                         | `User` model + `/api/users/search`                            |
| **Expense Service**  | Create expenses, manage splits (equal/custom), settle individual shares        | `Expense`, `ExpenseSplit` models + `add_expense`, `settle_expense` routes |
| **Balance Service**  | Compute per-user balances, global net balances, graph-based debt simplification | `_compute_balances`, `_compute_global_balances`, `_simplify_debts` |
| **Payment Service**  | Send payments between users, payment history, wallet credit/debit on payment   | `Payment` model + `send_payment`, `payments` routes           |
| **Wallet Service**   | Wallet balance management, top-up, transaction history                         | `Wallet` model + `wallet`, `topup_wallet` routes              |
| **Analytics Service**| Monthly spending trends, category breakdown, top spending partners             | `_monthly_spending`, `_category_breakdown`, `_top_partners`   |

### How Microservice Decomposition Would Work

```
Client (Browser / Mobile)
           |
       API Gateway
    /    |    \    \      \        \
 Auth  Expense Balance Payment  Wallet  Analytics
  DB     DB      DB      DB       DB       DB
```

- Each service owns its own database table(s)
- Services communicate via REST APIs or a message queue (e.g. RabbitMQ / AWS SQS)
- The Balance Service subscribes to events from Expense and Payment services to recompute balances
- Auth Service issues JWT tokens; all other services validate them independently

### Why Monolith First
For the current scope (single deployment, small team), a monolith is the right choice:
- Simpler to develop, test, and deploy
- No network latency between service calls
- Single database transaction across expense + split creation
- Easy migration to AWS with a single Gunicorn + RDS setup

The logical boundaries are already clean in the code, making a future microservices migration straightforward.

---

## CS581 Signature Project — Phase Mapping

| Phase | Requirement | How Spiltwise Covers It |
|-------|-------------|------------------------|
| Phase 1 | Architecture Design (VPC, EKS, Load Balancer) | Flask app deployed to EKS inside private subnets, NGINX ingress as load balancer |
| Phase 2 | EKS Cluster Deployment (eksctl/Terraform) | eksctl cluster config created, IAM node roles configured |
| Phase 3 | Multi-tier app (Frontend + Backend + DB) | Jinja2 templates (frontend) + Flask API (backend) + RDS PostgreSQL (database) |
| Phase 4 | IAM roles, RBAC, IRSA | IAM role with least privilege for EKS nodes, Kubernetes RBAC manifests, IRSA for pod-level AWS access |
| Phase 5 | Network Security (VPC, SGs, NACLs, Network Policies) | Private subnets for worker nodes, security groups restricting ports, Kubernetes NetworkPolicy manifests |
| Phase 6 | Data Security (encryption, Secrets Manager) | RDS encryption at rest, TLS via NGINX ingress, DB credentials stored in AWS Secrets Manager |
| Phase 7 | Container Security (non-root, minimal image, ECR scan) | Dockerfile uses python:3.11-slim, runs as non-root user, pushed to ECR with Trivy vulnerability scanning |
| Phase 8 | Monitoring & Logging (CloudWatch, GuardDuty) | Flask structured JSON logging to CloudWatch, GuardDuty enabled on cluster, Kubernetes audit logs enabled |
| Phase 9 | Threat Simulation & Mitigation (2+ scenarios) | 3 scenarios implemented — brute force, unauthorized access, log deletion attempt |

---

## Security Breach Scenarios (Phase 9)

### Scenario 1 — Brute Force Attack on Login

**What it is:** An attacker repeatedly tries different passwords on the `/login` endpoint to gain unauthorized account access.

**How it is simulated:**
- Script sends 5+ failed login requests to `/login` from the same IP in under 5 minutes

**How it is detected:**
- Flask security middleware tracks failed login attempts per IP
- After 5 failures within 5 minutes, threshold is exceeded

**What happens:**
- Event `BRUTE_FORCE_DETECTED` logged as structured JSON to CloudWatch
- AWS SNS sends an email alert immediately
- IP is blocked from further login attempts

**Log entry example:**
```json
{
  "timestamp": "2025-04-12T10:30:00Z",
  "service": "spiltwise",
  "event_type": "BRUTE_FORCE_DETECTED",
  "severity": "CRITICAL",
  "details": {
    "ip": "203.0.113.45",
    "email": "victim@example.com",
    "attempts_in_window": 6
  }
}
```

---

### Scenario 2 — Unauthorized Access Attempt

**What it is:** An unauthenticated user or bot tries to directly access protected routes like `/dashboard`, `/wallet`, `/expenses` without logging in.

**How it is simulated:**
- Send HTTP requests to protected routes without a valid session cookie

**How it is detected:**
- Flask `@login_required` decorator intercepts the request
- Security middleware logs the attempt with IP and route
- If the same IP hits 10+ protected routes within a minute, route scanning is flagged

**What happens:**
- Event `UNAUTHORIZED_ACCESS` logged to CloudWatch
- If scanning threshold hit, `SCANNING_DETECTED` fires and SNS alert is sent

**Log entry example:**
```json
{
  "timestamp": "2025-04-12T10:31:00Z",
  "service": "spiltwise",
  "event_type": "UNAUTHORIZED_ACCESS",
  "severity": "WARNING",
  "details": {
    "ip": "203.0.113.45",
    "route": "/wallet",
    "method": "GET"
  }
}
```

---

### Scenario 3 — Log Deletion / Audit Tampering (AWS-Level)

**What it is:** An attacker with stolen or overprivileged AWS credentials attempts to delete CloudWatch logs or disable CloudTrail to cover their tracks — a classic anti-forensics technique used in real-world breaches.

**How it is simulated:**
```bash
# Disable CloudTrail logging
aws cloudtrail stop-logging --name spiltwise-trail

# Delete CloudWatch log group
aws logs delete-log-group --log-group-name /spiltwise/app
```

**How it is detected:**
- AWS GuardDuty raises finding: `Stealth:IAMUser/CloudTrailLoggingDisabled`
- CloudTrail records the API call (`StopLogging`, `DeleteLogGroup`, `DeleteTrail`)
- CloudWatch Alarm fires on these specific API events

**What happens:**
- GuardDuty finding appears in AWS console within minutes
- CloudWatch alarm triggers SNS email: "CRITICAL: Someone attempted to delete audit logs"
- The attempt is permanently recorded in CloudTrail (protected by S3 versioning + MFA delete)

**GuardDuty Findings triggered:**

| Finding | Trigger |
|---------|---------|
| `Stealth:IAMUser/CloudTrailLoggingDisabled` | `StopLogging` or `DeleteTrail` API call |
| `Stealth:IAMUser/LogAggregationDisabled` | CloudWatch log aggregation disabled |
| `UnauthorizedAccess:IAMUser/ConsoleLogin` | Unusual console login from new IP |

**Why this is powerful:** Even if an attacker deletes logs, the deletion event itself is permanently recorded in CloudTrail. With S3 versioning and MFA-delete enabled on the CloudTrail S3 bucket, logs cannot be removed without MFA, creating a tamper-proof audit trail.

---

## Security Notification Flow

```
Security Event Occurs (app or AWS level)
               |
  Flask middleware / GuardDuty detects it
               |
    Structured JSON log written
               |
       CloudWatch Log Group (/spiltwise/security)
               |
    CloudWatch Metric Filter (matches CRITICAL severity)
               |
      CloudWatch Alarm triggers
               |
           AWS SNS Topic
          /              \
    Email Alert        SMS Alert
   (Professor)         (Professor)
```

**Events that trigger immediate SNS notification:**

| Event | Trigger Condition |
|-------|-------------------|
| `BRUTE_FORCE_DETECTED` | 5+ failed logins from same IP within 5 minutes |
| `SCANNING_DETECTED` | 10+ unauthorized route hits from same IP in 1 minute |
| `SUSPICIOUS_TRANSACTION` | Payment amount exceeds $5000 |
| `SIGNUP_ABUSE_DETECTED` | 3+ accounts created from same IP |
| `SQL_INJECTION_ATTEMPT` | Suspicious pattern detected in form input |
| `Stealth:IAMUser/CloudTrailLoggingDisabled` | Log deletion attempt detected by GuardDuty |

---

### Scenario 4 — AWS CloudWatch + EventBridge + SNS Pipeline (Infrastructure-Level)

**What it is:** Any suspicious AWS API activity is automatically captured by CloudTrail, filtered by EventBridge rules, and routed as a real-time alert via SNS — completely independent of the Flask application. Works even if the app is down or compromised.

**How it works:**

```
Any AWS API Call
       |
  CloudTrail (records every API call permanently)
       |
  EventBridge Rule (filters for suspicious events)
       |
  SNS Topic
      / \
Email   SMS
  +
CloudWatch Log (permanent record)
```

**Suspicious events wired up via EventBridge:**

| Suspicious Action | EventBridge Rule Catches |
|---|---|
| AWS Console login from unknown IP without MFA | `ConsoleLogin` with `MFAUsed = No` |
| New IAM user created (unauthorized provisioning) | `CreateUser` API call |
| Security group opened port 22 or 3389 (SSH/RDP exposed) | `AuthorizeSecurityGroupIngress` |
| S3 bucket made public (data exposure risk) | `PutBucketAcl` with public access |
| EC2 instance launched in unexpected region | `RunInstances` outside configured region |
| Root account used (highest privilege, should never be used) | `userIdentity.type = Root` |
| CloudTrail logging disabled | `StopLogging` or `DeleteTrail` |

**How to demo it:**
1. Create EventBridge rule matching `ConsoleLogin` without MFA
2. Log into AWS console without MFA enabled
3. Within 60 seconds — SNS fires email: *"IAM login detected without MFA on your AWS account"*
4. CloudWatch logs show the full event JSON permanently

**Sample EventBridge rule (JSON):**
```json
{
  "source": ["aws.signin"],
  "detail-type": ["AWS Console Sign In via CloudTrail"],
  "detail": {
    "additionalEventData": {
      "MFAUsed": ["No"]
    }
  }
}
```

**Sample SNS alert received:**
```
Subject: [Spiltwise Security] CRITICAL: Console Login Without MFA

Severity: CRITICAL
Event: ConsoleLogin without MFA
Time: 2025-04-12T10:45:00Z
Account: 123456789012
User: arn:aws:iam::123456789012:user/admin
Source IP: 203.0.113.45
Region: us-east-1
```

**Why this is impressive:**
- Purely AWS configuration — no code required
- Infrastructure-level monitoring independent of the application
- Root account detection is a real-world compliance requirement (SOC2, PCI-DSS, HIPAA)
- Covers Phase 8 (Monitoring & Logging) and Phase 9 (Threat Detection) simultaneously
- Demonstrates defense-in-depth — security at both application and infrastructure layers

---

## Project Structure

```
spiltwise/
├── app.py                  # Main Flask application
├── requirements.txt        # Python dependencies
├── .env                    # Environment variables (not committed)
├── .env.example            # Example env config
├── DETAILING.md            # This file
├── templates/
│   ├── base.html           # Base layout with nav
│   ├── landing.html        # Public landing page
│   ├── login.html          # Login form
│   ├── signup.html         # Signup form
│   ├── dashboard.html      # Main dashboard
│   ├── expenses.html       # Expense list
│   ├── add_expense.html    # Add expense form
│   ├── balances.html       # Balances and debt simplification
│   ├── payments.html       # Payment history
│   └── wallet.html         # Wallet management
└── static/
    ├── css/                # Stylesheets
    └── js/                 # JavaScript files
```

---

## Security Implementation (Pre-Deployment Code)

All application-level security is implemented in `security.py` and integrated into `app.py`.

### File: `security.py`

| Function | Purpose |
|---|---|
| `log_security_event()` | Logs structured JSON to stdout → captured by CloudWatch. Sends SNS alert if `notify=True` |
| `record_failed_login(ip, email)` | Tracks failed logins per IP. Triggers `BRUTE_FORCE_DETECTED` after 5 failures in 5 minutes |
| `is_ip_blocked(ip)` | Returns True if IP has exceeded login failure threshold |
| `clear_failed_logins(ip)` | Resets failed login count on successful login |
| `record_unauthorized_access(ip, route, method)` | Logs unauthorized route access. Triggers `SCANNING_DETECTED` after 10 hits in 1 minute |
| `check_suspicious_payment(user_id, amount, to_user)` | Blocks and logs payments exceeding $5,000 |

### Integration Points in `app.py`

| Route / Function | Security Check Added |
|---|---|
| `login_required` decorator | Calls `record_unauthorized_access()` on every unauthenticated request |
| `login()` route | Calls `is_ip_blocked()` before processing, `record_failed_login()` on failure, `clear_failed_logins()` on success |
| `send_payment()` route | Calls `check_suspicious_payment()` before processing the transaction |

### SNS Events Implemented

| Event | Trigger Condition | Status |
|-------|-------------------|--------|
| `BRUTE_FORCE_DETECTED` | 5+ failed logins from same IP within 5 minutes | Implemented in `security.py` |
| `SCANNING_DETECTED` | 10+ unauthorized route hits from same IP in 1 minute | Implemented in `security.py` |
| `SUSPICIOUS_TRANSACTION` | Payment amount exceeds $5,000 | Implemented in `security.py` |
| `Stealth:IAMUser/CloudTrailLoggingDisabled` | Log deletion attempt | After deployment via GuardDuty + EventBridge |

### Environment Variables Required

| Variable | Purpose |
|---|---|
| `SNS_TOPIC_ARN` | ARN of AWS SNS topic to send alerts to (set after deployment) |
| `AWS_DEFAULT_REGION` | AWS region where SNS topic is created (e.g. `us-east-1`) |

### Log Format (JSON)

Every security event is logged as structured JSON to stdout:
```json
{
  "timestamp": "2025-04-12T10:30:00Z",
  "service": "spiltwise",
  "event_type": "BRUTE_FORCE_DETECTED",
  "severity": "CRITICAL",
  "details": {
    "ip": "203.0.113.45",
    "email": "victim@example.com",
    "attempts_in_window": 6,
    "action": "IP blocked from further login attempts"
  }
}
```
