# 🚨 Production Outages (Root Cause Engineering Series)

## 🧠 Philosophy

এই সিরিজ কোনো debugging guide না।

এটা হচ্ছে:

> কীভাবে systems সত্যিই fail করে, এবং কেন তারা সেইভাবেই fail করতে বাধ্য ছিল — সেটা বোঝার চেষ্টা।

আমরা ধরে নেই:

> Systems randomভাবে fail করে না।  
> তারা ঠিক তাদের design boundary অনুযায়ী fail করে।

প্রতিটি incident analysis করা হবে এই lens দিয়ে:

- Hidden constraints (অদৃশ্য সীমাবদ্ধতা)
- Feedback loops (amplification behavior)
- Illusion vs reality (system কী দেখায় vs আসলে কী করছে)
- Design decisions যা failure allow করেছে

একটি ছোট্ট ডিসক্লেইমার:

এই সিরিজের প্রতিটি সিনারিও কোনো কাল্পনিক থিওরি বা বই থেকে নেওয়া নয়। এগুলো আমার নিজের ক্যারিয়ারে ফেস করা বাস্তব প্রোডাকশন ক্র্যাশ, রাতের পর রাত জাগা স্ট্রাগল এবং অন-ফিল্ড root cause অ্যানালিসিসের অভিজ্ঞতা থেকে লেখা।

---

## 📌 সূচিপত্র

- Database Scenarios
  - The Silent MySQL Crash
  - Flash Sale Crash: The Pool wasn’t the problem - Query latency was
- Distributed Systems Scenario

---

# 🗄️ Database Scenarios

## ⚠️ The Silent MySQL Crash

| Field | Detail |
|------|--------|
| Time | রাত ১১:৩২ |
| Symptom | Dashboard লোড হচ্ছে না, API হ্যাং করছে |
| Root Cause | Disk ১০০% ফুল — MySQL write path block |
| Resolution Time | ~১৫ মিনিট |
| Severity | 🔴 Critical |

---

## 📟 Alert

“Lahin bhai, system অনেক slow… userরা dashboard load করতে পারছে না।”

---

## 🔍 Investigation Timeline

### ১১:৩৫ PM — Application Layer

pm2 logs

✔ API চলছে  
✔ Workers চলছে  

curl /api/dashboard → hang

👉 Process alive, but no request progress

---

### ১১:৩৮ PM — Database Layer

systemctl status mysql → running (misleading)

mysql login → freeze

👉 DB alive, but no forward progress

---

### ১১:৪০ PM — Infrastructure Layer

df -h → 100% disk full

---

## 🧠 System Behavior Model

Request → MySQL → write blocked → txn stuck → connection stuck → pool full → API hang

---

## 🔗 Root Cause Chain

Disk Full
→ Write blocked
→ Transactions stuck
→ Connections not released
→ Queue builds
→ System stall

---

## 🛠️ Fix

du -sh /* | sort -rh | head -20

/var/lib/mysql  
/var/log  
/tmp  

mysql purge logs  
restart mysql  

---

## 🧠 Takeaway

Running ≠ Healthy  
Disk full = silent killer  
No forward progress = outage  

---

# ⚡ Flash Sale Crash

## Symptom
Too many connections + timeout

## Root Cause
Query latency (~500ms)

---

## Timeline

Traffic: 100 → 15000

pm2 logs → ER_CON_COUNT_ERROR

Threads_running = 10

---

## System Model

Slow query → connection held → pool full → queue → amplification loop

---

## Root Cause Chain

Missing index → full scan → slow query → pool saturation → collapse

---

## Fix

CREATE INDEX user_id

500ms → 10ms

10 / 0.01 = 1000 QPS

---

## Takeaway

Latency > Load  
Amplification kills systems  
Pool is time-based resource
