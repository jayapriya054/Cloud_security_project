# 💸 Spiltwise — Full-Stack Expense Splitting App

A complete Splitwise-like application built with **Python (Flask)** + **MongoDB** — 100% free to run!

---

## 🚀 Quick Start (5 minutes)

### Step 1: Install Python dependencies
```bash
cd spiltwise
pip install -r requirements.txt
```

### Step 2: Set up MongoDB (FREE — choose ONE option)

**Option A: Local MongoDB (no internet needed)**
1. Download MongoDB Community from: https://www.mongodb.com/try/download/community
2. Install and start it — it runs on `localhost:27017` by default
3. No config needed! The app connects automatically.

**Option B: MongoDB Atlas FREE Cloud (no install needed)**
1. Go to https://www.mongodb.com/atlas
2. Sign up for free → Create a FREE cluster (M0 tier — always free)
3. Get your connection string → Add it to `.env`

### Step 3: Configure environment
```bash
cp .env.example .env
# Edit .env if using Atlas, otherwise leave as-is for local MongoDB
```

### Step 4: Run the app!
```bash
python app.py
```

Open **http://localhost:5000** in your browser 🎉

---

## 🏗️ Architecture (Microservices Design)

| Service | Responsibility |
|---------|---------------|
| **User Service** | Signup, Login (bcrypt), Sessions, Profiles |
| **Expense Service** | Create expenses, equal/custom splits |
| **Balance Service** | Net balance computation per user pair |
| **Payment Service** | Simulate payments, mark debts settled |
| **Wallet Service** | Digital wallet, top-up, transfer history |
| **Analytics** | Monthly trends, category breakdown, partner stats |

---

## ✨ Features

- 🔐 **JWT-style auth** with bcrypt password hashing
- 💸 **Expense splitting** — equal or custom amounts
- 🧠 **Smart debt simplification** — graph algorithm collapses A→B→C to A→C
- 📊 **Financial dashboard** — monthly charts, category breakdown
- 👛 **Digital wallet** — top up, send, receive
- 📈 **Spending analytics** — who you spend most with
- 🎨 **Beautiful UI** — light theme, Syne + DM Sans fonts

---

## 📁 Project Structure

```
spiltwise/
├── app.py              # Main Flask app (all services)
├── requirements.txt    # Python dependencies
├── .env.example        # Environment config template
└── templates/
    ├── base.html       # Sidebar layout
    ├── landing.html    # Marketing homepage
    ├── login.html      # Login page
    ├── signup.html     # Registration page
    ├── dashboard.html  # Main dashboard with charts
    ├── expenses.html   # Expense list + settle
    ├── add_expense.html # Add expense with user search
    ├── balances.html   # Balance view + payment modal
    ├── payments.html   # Payment history
    └── wallet.html     # Digital wallet
```

---

## 💡 Interview Highlights

1. **Graph debt simplification** — reduces N IOUs to minimum payments
2. **Real-time user search** — AJAX search across users database
3. **Category analytics** — fintech-layer on top of basic splitting
4. **Wallet system** — mini digital banking with internal transfers
5. **Modular design** — each route group maps to a microservice concept

---

## 🆓 Running Cost: **₹0**

- Flask: Free, open source
- MongoDB Atlas M0: Free forever (512MB)
- No paid APIs, no subscriptions
