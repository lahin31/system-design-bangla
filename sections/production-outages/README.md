# 🚨 Production Outages (Root Cause Engineering Series)

## 🧠 Philosophy

এই সিরিজটা লেখার আসল উদ্দেশ্য debugging shortcuts শেখানো না।

বরং আমি চাই আপনারা বুঝুন:

> Systems আসলে কীভাবে ভেঙে পড়ে, এবং কেন তারা ঠিক ওই নির্দিষ্ট উপায়েই ভাঙতে বাধ্য হয়।

আমার একটা বিশ্বাস আছে এই নিয়ে:

> কোনো system হুট করে, random ভাবে fail করে না।  
> প্রতিটা failure আসলে সেই system-এর design-এর মধ্যেই লেখা থাকে — শুধু আমরা সেটা আগে থেকে দেখি না।

তাই এই সিরিজে প্রতিটা incident-কে আমি দেখব এই দৃষ্টিকোণ থেকে:

- লুকিয়ে থাকা constraints, যেগুলো আমরা ততক্ষণ খেয়াল করি না যতক্ষণ না সেগুলো আমাদের কামড় দেয়
- Feedback loops — ছোট একটা সমস্যা কীভাবে নিজেকে বড় করে তোলে
- System কী দেখাচ্ছে আর আসলে ভিতরে কী ঘটছে, এই দুটার মধ্যে ফাঁক
- সময়কে একটা resource হিসেবে দেখা - latency আসলে সবচেয়ে underrated bottleneck
- এবং সেই design decisions, যেগুলো অজান্তেই failure-কে অনিবার্য করে তুলেছিল

---

## 📝 একটি ছোট্ট ডিসক্লেইমার

এই সিরিজের প্রতিটি সিনারিও কোনো কাল্পনিক থিওরি বা বই থেকে নেওয়া নয়। এগুলো আমার নিজের ক্যারিয়ারে ফেস করা বাস্তব প্রোডাকশন ক্র্যাশ, রাতের পর রাত জাগা স্ট্রাগল এবং অন-ফিল্ড root cause অ্যানালিসিসের অভিজ্ঞতা থেকে লেখা।

---

## 📌 সূচিপত্র

- 🗄️ Database Scenarios
  - [The Silent MySQL Crash](#the-silent-mysql-crash)
  - [Flash Sale Crash: Pool নয়, আসল ভিলেন ছিল Query Latency](#flash-sale-crash-pool-নয়-আসল-ভিলেন-ছিল-query-latency)
  - [Migration Disaster: The Table that Froze Production](#mgration-disaster-the-table-that-froze-production)
- ⚙️ Application & Process Management Scenarios
  - [The 255-Character Crash Loop](#the-255-character-crash-loop)
  - [Cron ভুল টাইমজোনে ট্রিগার হওয়া](#cron-ভুল-টাইমজোনে-ট্রিগার-হওয়া)
- 🌐 Web Server & Routing Scenarios
  - [The SPA Refresh 404 (Nginx Routing Mismatch)](#the-spa-refresh-404-nginx-routing-mismatch)
- ⚡ Distributed Systems Scenarios (আসছে...)

# 🗄️ Database Scenarios

## The Silent MySQL Crash

| Field | Detail |
|------|--------|
| Time | রাত ১১:৩২ |
| Symptom | Dashboard load হচ্ছে না, API hang করছে |
| Root Cause | Disk 100% full → MySQL-এর write path পুরোপুরি ব্লক |
| Resolution Time | প্রায় ১৫ মিনিট |
| Severity | 🔴 Critical |

---

## 📟 Alert

"Lahin bhai, system অনেক slow… userরা dashboard load করতে পারছে না।"

এই ধরনের message পেলে প্রথম instinct হয় ভাবা যে app বা code-এ কোথাও bug আছে। কিন্তু বেশিরভাগ সময় সমস্যাটা অনেক নিচের layer-এ বসে থাকে।

---

## 🔍 Investigation Timeline

### ১১:৩৫ PM — Application Layer-এ প্রথম নজর

```bash
pm2 logs
```

API process তো বেঁচে আছে। Workers-ও চলছে। সব দেখে মনে হচ্ছে ঠিকই আছে।

তারপর চেক করা হলো:

```bash
curl /api/dashboard
```

→ কিছুই ফেরত আসছে না, শুধু hang করে আছে।

এটাই আসলে প্রথম clue — process জীবিত, কিন্তু কোনো request-ই এগোচ্ছে না। মানে কোথাও একটা জিনিস আটকে আছে।

---

### ১১:৩৮ PM — এবার Database-এর দিকে তাকালাম

```bash
systemctl status mysql
```

✔ MySQL "running" দেখাচ্ছে — কিন্তু এই signal-টা আসলে বিভ্রান্তিকর।

```bash
mysql -u root -p
```

→ এটাও আটকে গেল।

মানে DB টেকনিক্যালি "up" আছে, কিন্তু কাজ করছে না। দুটো আলাদা জিনিস।

---

### ১১:৪০ PM — Infrastructure Layer, এবং এখানেই উত্তর পাওয়া গেল

```bash
df -h
```

→ **Disk 100% full।**

ব্যাস, এখানেই পুরো গল্পের রহস্য খুলে গেল।

---

## 🧠 System কী হয়েছিল, আসলে

এটা কোনো crash না।

এটা হলো **backpressure যার কোনো escape path নেই**:

```
Request আসে
 → MySQL-এ write করতে যায়
 → Disk full, তাই ব্লক হয়ে যায়
 → Transaction আটকে থাকে
 → Connection কখনো release হয় না
 → Pool ভরে যায়
 → পুরো API layer থেমে যায়
```

System আসলে fail করছে না। সে শুধু সীমানার সামনে দাঁড়িয়ে অনন্তকাল অপেক্ষা করছে।

---

## 🔗 Root Cause Chain

```
Disk Full
→ Write path ব্লক
→ Transaction শেষ হতে পারছে না
→ Connection খোলা থেকে যাচ্ছে
→ Connection pool saturate
→ নতুন কোনো request এগোতে পারছে না
→ সিস্টেম-ওয়াইড stall
```

---

## 🛠️ যেভাবে ঠিক করা হলো

```bash
du -sh /* | sort -rh | head -20
```

বিশেষভাবে চেক করার জায়গা:

- /var/lib/mysql
- /var/log
- /tmp

তারপর:

- পুরনো log purge করলাম
- কিছু disk space খালি করলাম
- MySQL restart দিলাম, কারণ stuck state নিজে থেকে recover হয় না

---

## 🧠 আসল শিক্ষা

- Process running থাকা মানেই system healthy, এটা ধরে নেওয়া ভুল
- Disk full হওয়া শুধু storage সমস্যা না — এটা পুরো সিস্টেমকে freeze করে দেওয়ার মতো একটা condition
- "কোনো forward progress নেই" — এটাই আসল outage-এর সংজ্ঞা, error message না
- Backpressure যদি escape path না পায়, সেটাই ধীরে ধীরে collapse-এ পরিণত হয়

# Flash Sale Crash: Pool নয়, আসল ভিলেন ছিল Query Latency

## 📟 Alert

"সেল শুরু হতেই পুরো অ্যাপ ডাউন!"

ফ্ল্যাশ সেলের প্রথম মিনিটেই এই message — আর এই ধরনের incident-এ সবাই প্রথমে server বা ইনফ্রাকে দোষ দেয়। আসল কারণ কিন্তু অন্য জায়গায় ছিল।

---

## 🔍 Investigation Timeline

### ৮:০০ PM — Traffic Spike

```
100 users → 15,000 users
```

হঠাৎ করেই এই জাম্প। কোনো warm-up নেই, সরাসরি ১৫০x ট্রাফিক।

---

### ৮:০৫ PM — Application Layer-এ যা দেখা গেল

```bash
pm2 logs
```

Error গুলো:

- ER_CON_COUNT_ERROR
- ETIMEDOUT

প্রথম দেখায় মনে হয় "connection শর্টেজ" — কিন্তু এই ধারণাটা ভুল, এবং এটাই সবচেয়ে কমন trap।

---

### ৮:১০ PM — Database Layer খুঁটিয়ে দেখলাম

```sql
SHOW STATUS LIKE 'Threads_running';
```

```
Threads_running = 10 (maxed out)
```

এটা MySQL-এর কোনো hard limit না। এটা আসলে application-এর নিজের connection pool saturate হয়ে গিয়েছিল।

---

## 🧠 আসলে কী ঘটছিল

এই সিস্টেম connection-bound না।

এটা **time-bound**।

```
একটা slow query (~500ms)
→ Connection দীর্ঘ সময় ধরে রাখা হয়
→ Pool যথেষ্ট তাড়াতাড়ি recycle করতে পারছে না
→ Queue জমতে শুরু করে
→ Latency আরও বাড়ে
→ এবং এটা একটা positive feedback loop-এ পরিণত হয়
```

মানে সমস্যাটা নিজেই নিজেকে আরও খারাপ করছিল।

---

## 🔗 Root Cause Chain

```
Missing Index
→ Full table scan
→ Query latency ~500ms
→ DB connection দীর্ঘসময় ধরে থাকছে
→ Connection pool saturate (10 threads)
→ Request জমা হচ্ছে
→ ETIMEDOUT + ER_CON_COUNT_ERROR
→ পুরো সিস্টেম কলাপ্স
```

---

## 🧮 আসল হিসাব (Little's Law বাস্তবে কীভাবে কাজ করে)

### Fix-এর আগে

```
10 threads / 0.5s = 20 QPS
```

মানে সর্বোচ্চ ২০টা request প্রতি সেকেন্ডে হ্যান্ডেল করার ক্ষমতা ছিল — আর ট্রাফিক ছিল হাজারে হাজার।

### Fix-এর পরে

```
10 threads / 0.01s = 1000 QPS
```

একই ১০টা thread, কিন্তু এখন ৫০ গুণ বেশি throughput। Thread বাড়াইনি, শুধু latency কমিয়েছি।

---

## 🛠️ যেভাবে ঠিক করা হলো

```sql
SHOW FULL PROCESSLIST;
CREATE INDEX idx_user_id;
```

ফলাফল:

- Query time 500ms থেকে নেমে ১০ms
- Queue collapse-এর সমস্যা পুরোপুরি দূর হয়ে গেল

---

## 🧠 আসল শিক্ষা

- Throughput নির্ভর করে latency-র উপর, thread সংখ্যার উপর না
- Connection pool আসলে একটা time-based resource — যত বেশি thread, ততটা সমাধান না
- বেশিরভাগ "scaling problem" আসলে ভিতরে ভিতরে একটা query inefficiency problem
- Latency নিজেই feedback loop তৈরি করে, এবং সেটা দেখতে capacity শর্টেজের মতো লাগে — কিন্তু আসলে তা না

## Migration Disaster: The Table that Froze Production

### 🧱 Environment

- MySQL 5.7
- InnoDB
- `users` table size: ~43 million rows

> **নোট:** এই ঘটনাটি **MySQL 5.x**-এ ঘটেছিল, যেখানে `ALTER TABLE ... ADD COLUMN`-এর মতো অনেক DDL অপারেশন টেবিল রিবিল্ড ঘটাতে পারত এবং উল্লেখযোগ্য সময় ধরে মেটাডেটা লক ধরে রাখতে পারত। আধুনিক MySQL 8 ভার্সনগুলো এই ধরনের অনেক অপারেশনের জন্য `ALGORITHM=INSTANT` সাপোর্ট করে, যার ফলে এই নির্দিষ্ট ধরনের ব্যর্থতা এখন অনেক কম দেখা যায়।

## 📌 Incident Summary

| Field | Detail |
|---|---|
| Time | দুপুর ২:১৫ |
| Symptom | API suddenly hanging, database CPU low, connections increasing |
| Root Cause | A seemingly harmless migration acquired metadata locks and blocked the write path |
| Resolution | Kill migration session |
| Severity | 🔴 Critical |

## 📟 Alert

> "ভাই API খুব slow."

কয়েক মিনিট পরে:

> "Create user কাজ করছে না।"

তারপর:

> "Checkout কাজ করছে না।"

সবচেয়ে অদ্ভুত বিষয় ছিল:

- CPU usage low
- Memory healthy
- Database running
- Network healthy

কিন্তু পুরো system কার্যত unusable।

## 🔍 Investigation Timeline

### 2:17 PM

```sql
SHOW PROCESSLIST;
```

দেখা গেলো:

```text
Waiting for table metadata lock
Waiting for table metadata lock
Waiting for table metadata lock
Waiting for table metadata lock
```

### 2:19 PM

Deploy pipeline-এর অংশ হিসেবে একটি migration run হয়েছিল:

```sql
ALTER TABLE users
ADD COLUMN last_login_ip VARCHAR(45);
```

দেখতে harmless।

কিন্তু `users` table-এ ছিল:

```text
43 million rows
```

## 🧠 System কী হয়েছিল, আসলে

MySQL 5.x schema change করার আগে table-এর উপর metadata lock নেয় এবং অনেক DDL operation-এর ক্ষেত্রে table rebuild করে।

```text
মাইগ্রেশন শুরু হলো
→ মেটাডেটা লক নেওয়া হলো
→ নতুন write অপেক্ষায় থাকল
→ ট্রানজ্যাকশন জমা হতে লাগল
→ কানেকশন পুল ভরে গেল
→ API আটকে যেতে শুরু করল
→ রিট্রাই বাড়তে লাগল
→ পুরো সিস্টেম থমকে গেল
```

## 🔗 Root Cause Chain

```text
ALTER TABLE শুরু হলো
→ মেটাডেটা লক নেওয়া হলো
→ INSERT/UPDATE ব্লক হয়ে গেল
→ ট্রানজ্যাকশনগুলো খোলা থেকে গেল
→ কানেকশন রিলিজ হলো না
→ পুল saturation দেখা দিল
→ রিকোয়েস্ট queue বাড়তে থাকল
→ সর্বত্র টাইমআউট
```

## 🛠️ Resolution

```sql
SHOW PROCESSLIST;
```

Output

```markdown
Id    User   Command   Time    State
101   deploy Query     420     altering table
102   app    Query     300     Waiting for table metadata lock
103   app    Query     280     Waiting for table metadata lock
104   app    Query     275     Waiting for table metadata lock
```

এখানে:

- 101 হচ্ছে lock holder
- 102-104 হচ্ছে lock waiter

তখন:

```KILL 101;```

দিলে:

- ALTER TABLE বাতিল হলো
    - → কানেকশন টার্মিনেট হলো
    - → মেটাডেটা লক রিলিজ হলো
    - → অপেক্ষমান queries চলতে থাকল

## 🛡️ Prevention

- `gh-ost` বা `pt-online-schema-change` ব্যবহার করুন
- ভারী মাইগ্রেশনগুলো কম ট্রাফিকের সময়ে চালান
- প্রোডাকশনে রোলআউটের আগে মাইগ্রেশনের সময়কাল টেস্ট করুন
- মেটাডেটা লক এবং লক ওয়েট টাইম মনিটর করুন

## 🧠 আসল শিক্ষা

```markdown
> বেশিরভাগ প্রোডাকশন আউটেজ সিস্টেম ভেঙে ফেলার কারণে হয় না।

> এগুলো হয় তখন, যখন সিস্টেমকে ট্রাফিক সার্ভ করা অবস্থাতেই পরিবর্তন করতে বলা হয়।
```

# ⚙️ Application & Process Management Scenarios

## The 255-Character Crash Loop

## 🧠 Philosophy

এটা কোনো bug fixing গল্প না।

এটা হলো এমন একটা ঘটনা যেখানে ধরা হয়েছিল:

> "Database error মানেই application handle করে ফেলবে"

কিন্তু production সেই ধারণা follow করে না।

Production শুধু সেটাই চালায় যেটা বাস্তবে implement করা হয়েছে।

---

## 📌 Incident Summary

| Field | Detail |
|------|--------|
| Time | রাত (approx) |
| Symptom | API flapping — restart loop, dashboard/API intermittently unreachable |
| Root Cause | Service layer-এ uncaught DB error → process crash → PM2 restart loop |
| Resolution | `pm2 restart` দিয়ে stabilize, পরে code-level fix |
| Severity | 🔴 Critical |

---

## 🧱 System Context

Stack:

- Node.js (Express API)
- MySQL
- PM2 (Process Manager)
- REST API (Web + Mobile clients)

### Layered Architecture

```
Controller Layer  →  try/catch ছিল
Service Layer     →  try/catch ছিল না
```

### Endpoint

```
POST /users
```

### Database Schema

```sql
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50)
);
```

### Design Assumption

- Controller layer error handle করে দিচ্ছে — এই ধরে নেওয়া হয়েছিল যথেষ্ট
- Service layer-এর ভেতরের errors কে controller-এর try/catch automatically cover করবে
- Database acts as final safety net

---

## 💥 Incident Trigger

একজন user পাঠায়:

```
name = "A very long string exceeding 50 characters........"
```

MySQL ঠিকভাবে reject করে:

```
ER_DATA_TOO_LONG: Data too long for column 'name'
```

(এই error code-এর মানে হলো — column-এ যতটুকু capacity আছে, তার চেয়ে বেশি data পাঠানো হয়েছে। MySQL নিজেই কিছু ভাঙেনি, সে শুধু constraint enforce করেছে।)

এটা expected behavior ছিল।

কিন্তু সমস্যা এখানে শুরু হয়।

---

## 🔍 Investigation Timeline

### Step 1 — Application Layer

```
pm2 list
```

```
┌────┬──────────┬─────────┬────────┬─────────┬──────────┐
│ id │ name     │ status  │ ↺      │ uptime   │ cpu      │
├────┼──────────┼─────────┼────────┼─────────┼──────────┤
│ 0  │ api      │ online  │ 47     │ 3s       │ 0%       │
└────┴──────────┴─────────┴────────┴─────────┴──────────┘
```

👉 `↺` (restart count) অনেক বেশি, `uptime` বারবার রিসেট হচ্ছে — process স্থিতিশীল হচ্ছে না, repeatedly crash করছে।

### Step 2 — Crash Evidence

```
pm2 logs api --lines 100
```

```
ER_DATA_TOO_LONG: Data too long for column 'name'
[PM2] App [api] exited with code 1
[PM2] App [api] starting in -fork mode-
[PM2] App [api] online
ER_DATA_TOO_LONG: Data too long for column 'name'
[PM2] App [api] exited with code 1
[PM2] App [api] starting in -fork mode-
[PM2] App [api] online
ER_DATA_TOO_LONG: Data too long for column 'name'
[PM2] App [api] exited with code 1
```

👉 একই error বারবার ফিরে আসছে — pattern repeat করছে, random crash না।

### Step 3 — কোথায় Error Catch হচ্ছিল না

Controller layer-এ try/catch ছিল:

```js
// controller.js
async function createUser(req, res) {
  try {
    const user = await userService.create(req.body.name);
    return res.status(201).json(user);
  } catch (err) {
    return res.status(500).json({ error: "Something went wrong" });
  }
}
```

কিন্তু service layer-এ কোনো try/catch ছিল না:

```js
// userService.js
async function create(name) {
  // No try/catch here — query error throws straight up
  const result = await db.query(
    `INSERT INTO users (name) VALUES (?)`,
    [name]
  );
  return result;
}
```

দেখতে মনে হয় controller-এর try/catch পুরো request lifecycle কে cover করবে — service layer যেখানেই error throw করুক, সেটা await chain ধরে controller-এর catch পর্যন্ত bubble up করা উচিত।

কিন্তু practically, এখানে error escape করেছিল — কারণ error properly `await` করা হয়নি কোনো একটা জায়গায় (একটা fire-and-forget call, বা একটা non-awaited promise এর ভেতরে error থ্রো হয়েছিল), যার ফলে সেটা controller-এর synchronous catch boundary-র বাইরে গিয়ে unhandled rejection হিসেবে process-কে crash করিয়েছে।

👉 **Lesson:** "একটা try/catch থাকলেই যথেষ্ট" এই ধারণাটাই ভুল ছিল। Error যেখানে actually throw হয়, catch সেখানেই থাকতে হয় — উপরের layer-এর catch নিচের layer-এর প্রতিটা throw path কে guarantee করে না, বিশেষ করে async/promise chain ঠিকভাবে না থাকলে।

---

## ❌ What Actually Happened

ফলাফল:

- Service layer-এ unhandled rejection
- Controller-এর try/catch সেটা ধরতে পারেনি
- Node.js নিজেই default behavior অনুযায়ী crash করে process terminate করে দেয় (exit code 1, কোনো explicit `process.exit()` call ছাড়াই)

👉 এবং process terminate হয়ে যায়, PM2 restart করে, কিন্তু পরের request আসলে আবার একই জায়গায় crash করে।

---

## 🔁 PM2 Restart Loop

```
Service layer-এ error throw
      ↓
Controller-এর catch escape করে
      ↓
Node process crash (unhandled rejection)
      ↓
PM2 detects crash → restart
      ↓
পরের identical request এ আবার same crash
      ↓
Restart count বাড়তে থাকে (47+)
```

System unstable, from outside দেখতে মনে হচ্ছিল:

> "API randomly down / flapping"

---

## ✅ Resolution

```
pm2 restart api
```

👉 এই কমান্ড চালানোর পর system **stable হয়ে গিয়েছিল** — loop থেমে গিয়েছিল।

এটা একটা গুরুত্বপূর্ণ পয়েন্ট: এই incident একটা **self-sustaining infinite loop ছিল না**। এটা ছিল একই bad input বারবার আসার ফলে তৈরি হওয়া একটা crash burst — manual restart দেওয়ার সময় থেকে নতুন কোনো invalid request না আসায় system সুস্থ অবস্থায় থেকে গিয়েছিল। Restart নিজে গিয়ে underlying bug কে fix করেনি — শুধু সাময়িকভাবে crash chain টা ভেঙে দিয়েছিল। সেই bad input আবার আসলে crash আবার হতো, যতক্ষণ না code-level fix দেওয়া হয়।

---

## 🧨 Root Cause

MySQL এখানে কোনো সমস্যা করেনি — সে শুধু বলেছে data column capacity exceed করছে।

Real issue ছিল architectural:

> Controller layer-এর error handling-কে "পুরো request lifecycle-এর safety net" ভাবা হয়েছিল, কিন্তু service layer-এর ভেতরের একটা uncaught throw path সেই boundary-কে bypass করে গিয়েছিল।

---

## 🔥 Why This Became an Outage

তিনটা component individually তাদের নিজের জায়গায় ঠিক কাজ করেছে:

### 1. MySQL
✔ Correct constraint enforcement

### 2. Controller Layer
✔ try/catch ছিল, কিন্তু সেটার coverage ছিল incomplete — সব throw path cover করেনি

### 3. Service Layer
❌ কোনো try/catch ছিল না — error সরাসরি uncaught থেকে process crash করিয়েছে

### 4. PM2
✔ Crash detect করে restart করেছে (expected behavior) — কিন্তু `pm2 restart` manual intervention-ই আসলে loop থামিয়েছে, PM2-র auto-restart নিজে থেকে কিছু "ঠিক" করেনি

---

## ⚠️ Hidden Failure Mode

> একটা upper-layer try/catch থাকা মানেই "error handled" — এই assumption টাই false security দিয়েছিল।

Single bad request একটা crash trigger করলো → PM2 restart করলো → আবার same/similar bad request এলে আবার crash → এই pattern রিপিট হয়ে restart count বাড়তে থাকলো — যতক্ষণ manual restart না দেওয়া হলো এবং traffic pattern বদলালো।

---

## 🧠 System Design Lesson

ভুল ধারণা ছিল:

> "Controller-এ try/catch আছে, তাই পুরো request safe।"

Reality:

> প্রতিটা layer-এর own error boundary প্রয়োজন। Upper layer-এর catch নিচের layer-এর প্রতিটা throw/reject path কে automatically cover করে না — বিশেষ করে async code-এ, যেখানে একটা missed `await` বা un-awaited promise সহজেই catch boundary বাইপাস করতে পারে।

---

## 🛠️ Fix Applied

### 1. Service Layer-এ Proper Error Handling

```js
// userService.js
async function create(name) {
  try {
    return await db.query(
      `INSERT INTO users (name) VALUES (?)`,
      [name]
    );
  } catch (err) {
    if (err.code === "ER_DATA_TOO_LONG") {
      const validationError = new Error("Input too long");
      validationError.statusCode = 400;
      throw validationError;
    }
    throw err;
  }
}
```

👉 প্রতিটা layer নিজের errors নিজে handle করে এবং proper, predictable error shape upward pass করে — controller-এর catch তখন একটা known shape-এর উপর কাজ করতে পারে।

### 2. Input Validation Before DB

```js
if (name.length > 50) {
  return res.status(400).json({
    error: "Name exceeds maximum allowed length"
  });
}
```

### 3. PM2 Configuration Hardening

```js
// ecosystem.config.js
module.exports = {
  apps: [{
    name: "api",
    script: "./server.js",
    max_restarts: 15,              // hard ceiling instead of unlimited
    min_uptime: "10s",             // crash within 10s doesn't count as a "successful" run
    exp_backoff_restart_delay: 100 // exponential backoff between restarts
  }]
};
```

👉 এই configuration থাকলে repeated crash-এর ক্ষেত্রে PM2 নিজেই restart বন্ধ করে process-কে `errored` state-এ রাখতো — manual intervention এর জন্য অপেক্ষা না করিয়ে একটা clean, alertable failure তৈরি হতো।

### 4. `process.exit()` ব্যবহারের নীতি

`process.exit()` শুধু এই ক্ষেত্রে use করা উচিত:

- Boot failure
- Missing critical config
- Corrupted runtime state

❌ কখনোই user input errors-এর জন্য নয় — এবং এই incident-এ কোনো explicit `process.exit()` call ছিলও না; crash হয়েছিল Node-এর default unhandled-rejection behavior থেকে।

---

## 🧠 আসল শিক্ষা

> Systems don't fail randomly — they fail exactly according to their design boundaries.

এই case এ boundary টা ছিল layer-ভিত্তিক, error-path ভিত্তিক না:

- Controller layer ভেবেছিল এটা safe
- Service layer-এর একটা path সেই assumption ভেঙে দিয়েছিল
- PM2 faithfully restart করেছে — কিন্তু restart নিজে bug fix করেনি, শুধু symptom সাময়িকভাবে থামিয়েছে

## Cron ভুল টাইমজোনে ট্রিগার হওয়া

## 📟 Alert

প্রতিদিন রাত ১২টায় (Asia/Dhaka, UTC+6) একটা ডেইলি রিপোর্ট/বিলিং জব(CRON) রান হওয়ার কথা ছিল। কিন্তু ইউজাররা রিপোর্ট করলো যে রিপোর্ট কখনো ৬ ঘণ্টা আগে, কখনো ৬ ঘণ্টা পরে আসছে — কখনো কখনো পুরোপুরি মিসিং, আবার কখনো একই দিনে দুইবার রান হয়ে যাচ্ছে।

## 🔍 Investigation Timeline

- CRON এন্ট্রি দেখতে সঠিকই মনে হচ্ছিলো: 0 0 * * *

- লগ চেক করে দেখা গেলো, জব ঠিকই ফায়ার হচ্ছে — কিন্তু আগের দিন সন্ধ্যা ৬টায়, নাহলে সকাল ৬টায়।

- সার্ভারে date কমান্ড দিয়ে দেখা গেলো সার্ভার UTC টাইমে চলছে, লোকাল টাইমে না — কিন্তু যে ডেভেলপার CRON এক্সপ্রেশনটা লিখেছিলো, সে মাথায় ঢাকার টাইম রেখে 0 0 * * * লিখেছিলো।

- আরেকটু গভীরে গিয়ে দেখা গেলো: সিস্টেম CRON OS-এর টাইমজোন অনুযায়ী চলে (ক্লাউড ইনস্ট্যান্সে সাধারণত ডিফল্ট UTC থাকে), অথচ অ্যাপ্লিকেশন নিজে TZ এনভায়রনমেন্ট ভ্যারিয়েবল বা কোনো ডেট লাইব্রেরির মাধ্যমে Asia/Dhaka ব্যবহার করছিলো — মানে একই লজিক্যাল জবের জন্য দুইটা আলাদা ঘড়ি কাজ করছিলো।

- বোনাস সমস্যা: পরে যদি অ্যাপ্লিকেশনটা মাল্টি-রিজিয়ন/কন্টেইনারে ডিপ্লয় করা হয়, প্রতিটা কন্টেইনারের TZ সেটিং আলাদা হতে পারে — তখন "একই" CRON আলাদা আলাদা মেশিনে আলাদা আলাদা রিয়েল-ওয়ার্ল্ড সময়ে ফায়ার হবে।

## 🧠 System কী হয়েছিল, আসলে

প্রথমে মনে হয়েছিল সমস্যাটা billing calculation logic-এই আছে — হয়তো usage aggregation query ভুল date range ধরছে। টিম প্রথম কিছু সময় query-ই রিভিউ করলো।

কিন্তু আসল সমস্যাটা query-তে ছিলোই না। ডেভেলপার নিজের মাথার টাইমজোন ধরে cron এক্সপ্রেশন লিখলো → সিস্টেম cron সেটাকে OS/কন্টেইনারের টাইমজোনে ইন্টারপ্রেট করলো (সাধারণত UTC) → মিসম্যাচ → জব ভুল সময়ে ফায়ার হলো → এর ডাউনস্ট্রিম প্রভাব (ডাবল-চার্জিং, মিসিং ডেইলি রিপোর্ট, অন্য জবের সাথে অর্ডার নিয়ে রেস কন্ডিশন)।

```
# ডেভেলপার যা ভেবেছিলেন:
0 0 * * *  →  "রাত ১২টা, মানে Dhaka midnight"

# cron daemon যা আসলে করলো:
0 0 * * *  →  "00:00 UTC"  =  "06:00 AM Dhaka time" (৬ ঘণ্টা পার্থক্য)
```

## 🛠️ যেভাবে ঠিক করা হলো

- সিস্টেম cron-এর বদলে `node-cron` ব্যবহার করে explicitly `timezone` অপশন পাস করে দেওয়া হলো — এতে সার্ভারের OS timezone যাই হোক না কেন, job সবসময় সঠিক লোকাল সময়ে ফায়ার হবে:

```javascript
const cron = require("node-cron");

cron.schedule(
  "0 0 * * *",
  () => runDailyBilling(),
  { timezone: "Asia/Dhaka" }
);
```

- সার্ভারের সিস্টেম টাইমজোন স্পষ্টভাবে UTC-তে সেট করে দেওয়া হলো (provisioning script-এ enforce করা হলো), যাতে ভবিষ্যতে নতুন কোনো ইনস্ট্যান্স/কন্টেইনার ভিন্ন টাইমজোনে স্পিন-আপ হয়ে একই বাগ রিপিট না করে:

```
timedatectl set-timezone UTC
```

## 🧠 আসল শিক্ষা

- Cron কোনো টাইমজোন বোঝে না — শুধু মেশিনের system clock অনুসরণ করে।

- ক্লাউড ইনস্ট্যান্স/কন্টেইনার প্রায় সবসময় ডিফল্ট UTC — সার্ভারের লোকাল টাইম আপনার লোকাল টাইমের সাথে মিলবে এটা ধরে নেওয়া যাবে না।

- কোনো জবের অর্থ যদি নির্দিষ্ট টাইমজোনের সাথে বাঁধা থাকে, সেই অর্থটাকে UTC-তে কনভার্ট করে cron expression-এ লিখুন, আর কমেন্টে ব্যাখ্যা রাখুন।

- DST মেনে চলা টাইমজোনেও সতর্ক থাকতে হবে (বাংলাদেশ প্রযোজ্য না, তবে আন্তর্জাতিক সিস্টেমে গুরুত্বপূর্ণ)।

# 🌐 Web Server & Routing Scenarios

## The SPA Refresh 404 (Nginx Routing Mismatch)

## 📌 Incident Summary

| Field | Detail |
|---|---|
| Time | Deploy-এর পরের দিন, সকাল |
| Symptom | `/dashboard`-এ refresh দিলেই 404, কিন্তু সরাসরি homepage থেকে navigate করলে ঠিকঠাক কাজ করে |
| Root Cause | Nginx file-system-ভিত্তিক routing আশা করছিল, কিন্তু SPA route গুলো client-side JavaScript-এ resolve হচ্ছিল — সার্ভারে সেই path-এর কোনো ফাইল নেই |
| Resolution | `try_files` directive দিয়ে সব unknown route-কে `index.html`-এ fallback করানো |
| Severity | 🟠 High (user-facing, কিন্তু data loss নেই) |

---

## 📟 Alert

"ভাই, dashboard page refresh দিলেই 404 দেখাচ্ছে। কিন্তু লিংকে ক্লিক করলে ঠিকই কাজ করে!"

এই ধরনের bug report শুনলে প্রথমে মনে হয় frontend routing-এ কোনো bug আছে। কিন্তু আসল সমস্যা client এবং server — দুই জায়গায় route বোঝার দুইটা সম্পূর্ণ আলাদা মডেলের মধ্যে।

---

## 🔍 Investigation Timeline

### Step 1 — Browser থেকে প্রথম পর্যবেক্ষণ

```
example.com → click "Dashboard" link → example.com/dashboard লোড হয়
```

কোনো problem নেই। URL বদলেছে, page-ও বদলেছে।

```
example.com/dashboard → press F5 (refresh)
```

→ **404 Not Found**

👉 একই URL, কিন্তু একবার কাজ করছে, একবার করছে না। মানে সমস্যাটা route নিজে নয় — সমস্যা হলো *কীভাবে* এই route-এ পৌঁছানো হচ্ছে।

### Step 2 — Network Tab খুঁটিয়ে দেখা

ক্লিক করার সময়:

```
No network request sent for /dashboard
```

Refresh করার সময়:

```
GET /dashboard → 404
```

👉 এখানেই clue পাওয়া গেল — link click-এর সময় কোনো HTTP request-ই Nginx পর্যন্ত যাচ্ছে না। পুরোটাই browser-এর ভেতরে handle হচ্ছে।

### Step 3 — Nginx Config চেক করা

```
cat /etc/nginx/sites-available/default
```

```nginx
location / {
  try_files $uri $uri/ =404;
}
```

👉 পাওয়া গেল আসল কারণ — `=404` মানে Nginx-কে বলা আছে, file বা directory না পাওয়া গেলে সরাসরি 404 ফেরত দিতে। Nginx-এর কাছে `/dashboard` মানে disk-এ থাকা একটা ফাইল বা ফোল্ডার — যা আদতে কখনো বানানোই হয়নি।

---

## 🧠 System কী হয়েছিল, আসলে

এটা কোনো crash বা bug না — এটা দুই layer-এর routing model-এর মধ্যে একটা **mismatch**।

```
Browser একটা link-এ click করে
→ React Router URL বদলে দেয় (pushState)
→ কোনো server request যায়ই না
→ Component client-side render হয়
→ সব ঠিকঠাক দেখায়

কিন্তু refresh করলে:
→ Browser একদম fresh GET request পাঠায়
→ Nginx সেই path-কে ফাইল-সিস্টেম path হিসেবে treat করে
→ /dashboard নামে কোনো ফাইল বা folder নেই
→ Nginx 404 ফেরত দেয় — JavaScript লোড হওয়ার সুযোগই পায় না
```

Nginx ভুল কিছু করেনি — সে নিজের নিয়ম অনুযায়ী ঠিকই কাজ করছে। সমস্যা হলো, SPA routing model-এর সাথে এই নিয়মটা কখনো align করানোই হয়নি।

---

## 🔗 Root Cause Chain

```
SPA deploy করা হয় (client-side routing সহ)
→ Nginx config-এ শুধু literal file/folder lookup আছে
→ Client navigation request পাঠায় না (JS intercept করে)
→ Refresh/direct-URL hit সরাসরি Nginx-কে আসল path জিজ্ঞেস করে
→ ডিস্কে সেই path-এর কোনো file নেই
→ try_files $uri $uri/ =404 fallback ট্রিগার হয়
→ Nginx 404 পাঠায়, index.html কখনো লোড হয় না
→ React Router-এর সুযোগই থাকে না route resolve করার
```

---

## 🛠️ যেভাবে ঠিক করা হলো

```nginx
server {
  listen 80;
  server_name example.com;
  root /var/www/my-spa-app;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

```
sudo nginx -t
sudo systemctl reload nginx
```

### `try_files` ধাপে ধাপে যা করে

1. **`$uri`** — exact file আছে কিনা চেক করে (যেমন `logo.png`)। থাকলে সরাসরি serve করে।
2. **`$uri/`** — file না হলে directory কিনা চেক করে।
3. **`/index.html`** — কোনোটাই না থাকলে (যেমন `/dashboard`), `index.html` ফেরত দেয়। Browser SPA লোড করে, client-side router URL দেখে ঠিক component বানায়।

ফলাফল: refresh করলেও আর 404 আসে না, কারণ Nginx এখন routing decision client-side JavaScript-এর হাতে ছেড়ে দেয়, নিজে judge করার চেষ্টা করে না।

---

## ⚠️ Hidden Failure Mode

> "যদি ক্লিক করে কাজ করে, তাহলে route-টা ঠিক আছে" — এই assumption-টাই বিভ্রান্তিকর।

Client-side navigation আর server-side request — এই দুটো সম্পূর্ণ আলাদা path। QA-তে যদি কেউ শুধু click করে test করে, refresh কখনো না করে, এই bug প্রোডাকশনে যাওয়া পর্যন্ত ধরাই পড়বে না। আর deep-link শেয়ার করা (কাউকে সরাসরি `/dashboard` লিংক পাঠানো) একই কারণে ভেঙে যাবে — কারণ সেটাও একটা fresh server request।

---

```

## 🧨 Common Gotchas (Fix করার পরেও যা মাথায় রাখতে হবে)

### 1. Real 404 হারিয়ে যায়

```
example.com/this-route-does-not-exist-anywhere
```

এখন এটাও `index.html`-এ fallback হয়ে যাবে — Nginx-level 404 আর দেখা যাবে না। এর মানে **router নিজেই** একটা catch-all "Not Found" route define করতে হবে, নাহলে user একটা broken page-কে valid page ভেবে বসে থাকবে।

### 2. `index.html` Caching

`index.html` যদি aggressively cache হয়, নতুন deploy করার পরেও user পুরনো JS bundle reference করা version দেখতে থাকবে। `index.html`-এর জন্য `Cache-Control: no-cache` রাখা ভালো, আর static asset (JS/CSS bundle)-এর জন্য hashed filename + long-term caching।

---

## 🧠 আসল শিক্ষা

> Server কখনো জানে না client-side router-এর ভেতরে কী কী route define করা আছে — তাই তাকে বলে দিতে হয়, "তোমার যা চেনার কথা না, সেটার fallback দিয়ে দিও।"

- File-system-ভিত্তিক routing আর client-side routing — এই দুই মডেল কখনো নিজে থেকে একে অপরের সাথে align হয় না, explicitly বলে দিতে হয়
- "ক্লিক করে কাজ করছে" মানেই "route ঠিক আছে" না — refresh আর direct-URL hit আলাদাভাবে test করতে হয়
- একটা fix (try_files fallback) নিজের সাথে নতুন trade-off নিয়ে আসে (real 404 হারানো) — fix দেওয়ার মানে এই না যে সব side-effect ভাবা শেষ
- Infrastructure layer (Nginx) আর application layer (router) — দুই জায়গাতেই "not found" handling থাকা লাগে, একটা আরেকটার বিকল্প না
