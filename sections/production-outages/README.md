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
