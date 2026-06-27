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
- সময়কে একটা resource হিসেবে দেখা — latency আসলে সবচেয়ে underrated bottleneck
- এবং সেই design decisions, যেগুলো অজান্তেই failure-কে অনিবার্য করে তুলেছিল

---

## 📝 একটি ছোট্ট ডিসক্লেইমার

এই সিরিজের প্রতিটি সিনারিও কোনো কাল্পনিক থিওরি বা বই থেকে নেওয়া নয়। এগুলো আমার নিজের ক্যারিয়ারে ফেস করা বাস্তব প্রোডাকশন ক্র্যাশ, রাতের পর রাত জাগা স্ট্রাগল এবং অন-ফিল্ড root cause অ্যানালিসিসের অভিজ্ঞতা থেকে লেখা।

---

## 📌 সূচিপত্র

- 🗄️ Database Scenarios
  - The Silent MySQL Crash
  - Flash Sale Crash: Pool নয়, আসল ভিলেন ছিল Query Latency
- ⚙️ Application & Process Management Scenarios
  - The 255-Character Crash Loop
- ⚡ Distributed Systems Scenarios (আসছে...)

---

# 🗄️ Database Scenarios

---

# ⚠️ The Silent MySQL Crash

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

তারপর চেক করলাম:

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

## 🛠️ যেভাবে ঠিক করলাম

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

---

# ⚡ Flash Sale Crash: Pool নয়, আসল ভিলেন ছিল Query Latency

---

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

## 🛠️ যেভাবে ঠিক করলাম

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

---

# ⚙️ Application & Process Management Scenarios

---

# The 255-Character Crash Loop

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

---