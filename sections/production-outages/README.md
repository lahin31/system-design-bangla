# Production Outages

## সূচিপত্র

- [Database Scenarios](#database-scenarios)
  - [The Silent MySQL Crash](#the-silent-mysql-crash)
  - [Flash Sale Crash: The Pool wasn’t the problem - Query latency was](#flash-sale-crash-the-pool-wasnt-the-problem---query-latency-was)


## Database Scenarios

### The Silent MySQL Crash

| Field               | Detail                                  |
| ------------------- | ----------------------------------------|
| **Time**            | রাত ১১:৩২                                |
| **Symptom**         | Dashboard লোড হচ্ছে না, API হ্যাং করছে        |
| **Root Cause**      | Disk ১০০% ফুল — MySQL লিখতে পারছে না        |
| **Resolution Time** | ~১৫ মিনিট                                 |
| **Severity**        | 🔴 Critical                              |

---

#### 📟 Alert

```
"Lahin bhai, system অনেক slow… userরা dashboard load করতে পারছে না।"
```

---

#### 🔍 Investigation Timeline

**`১১:৩৫ PM` — Application Layer**

```bash
pm2 logs
```

```
✅ API চলছে  
✅ Workers চলছে  
```

সবকিছু দেখে ঠিক মনে হচ্ছিল। কিন্তু endpoint manually hit করতেই বোঝা গেল আসল সমস্যা:

```bash
curl /api/dashboard
# হ্যাং করে থাকে...
```

Application বেঁচে আছে, কিন্তু response দিচ্ছে না — মানে সমস্যা নিচের লেয়ারে।

---

**`১১:৩৮ PM` — Database Layer**

```bash
systemctl status mysql
# Active (running) ← শুধু নামেই running
```

Direct login করার চেষ্টা:

```bash
mysql -u root -p
# ফ্রিজ করে… তারপর fail
```

মানে MySQL technically চালু, কিন্তু practically dead — কোনো query serve করতে পারছে না।

---

**`১১:৪০ PM` — Infrastructure Layer**

```bash
df -h
```

```
Filesystem      Size  Used  Avail  Use%  Mounted on
/dev/sda1        50G   50G     0G  100%  /          ← 🔴
```

**এখানেই সমস্যা।** Disk পুরোপুরি ফুল।

MySQL তখন “alive but paralyzed” — log লিখতে পারছে না, temp file তৈরি করতে পারছে না, transaction complete করতে পারছে না।

ফলাফল: সব query হ্যাং।

---

#### 🛠️ Fix

**Step 1 — কোনটা disk খাচ্ছে খুঁজে বের করুন**

```bash
du -sh /* | sort -rh | head -20
```

---

**Step 2 — সাধারণত যেগুলো culprit হয়**

```
/var/lib/mysql/        ← binary logs  
/var/log/              ← log rotate হয়নি  
/tmp/                  ← temp file, dump  
```

---

**Step 3 — কিছু space খালি করুন, তারপর restart**

```bash
# পুরানো MySQL binary logs delete
mysql -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);"

systemctl restart mysql
```

---

#### 🧠 Takeaway

* “Service running” মানেই service healthy না
* Disk full = silent killer (especially for DB)
* Monitoring ছাড়া এই সমস্যা আগে থেকে ধরা কঠিন
* Log rotation, alerting, disk threshold — production এ must-have

---

এই ধরনের আউটেজ আপনাকে শেখাবে — problem সবসময় code-এ না, অনেক সময় infrastructure-এই আসল bottleneck থাকে।

### Flash Sale Crash: The Pool wasn’t the problem - Query latency was

| Field               | Detail                                      |
| ------------------- | --------------------------------------------|
| **Time**            | রাত ৮:০০ (Black Friday Sale শুরু)             |
| **Symptom**         | `Too many connections`, API timeout          |
| **Root Cause**      | High query latency (~500ms)                  |
| **Resolution Time** | ~৩০ মিনিট                                    |
| **Severity**        | 🔴 Critical                                  |

---

#### 📟 Alert

```
"Lahin bhai, সেল শুরু হতেই অ্যাপ ডাউন! ডাটাবেজ কানেকশন এরর দিচ্ছে, ইউজাররা কিছুই কিনতে পারছে না।"
```

---

#### 🔍 Investigation Timeline

**`৮:০০ PM` — Traffic Spike**

ফ্ল্যাশ সেল শুরু হওয়ার সাথে সাথে traffic jumps from **~100 → 15,000** concurrent users। API suddenly শুরু করে fail করতে:

```
ER_CON_COUNT_ERROR
```

প্রথম সন্দেহ — "Connection pool ছোট নাকি?"

---

**`৮:০৫ PM` — Application Layer**

```bash
pm2 logs
```

```
0|app  | Error: connect ETIMEDOUT 127.0.0.1:3306
0|app  | at Connection._handleConnectTimeout
0|app  | SequelizeConnectionError: connect ETIMEDOUT
0|app  | at /app/node_modules/sequelize/lib/dialects/mysql/connection-manager.js

0|app  | Error: ER_CON_COUNT_ERROR: Too many connections
0|app  | at Handshake.Sequence._packetToError
```

PM2 ঠিক আছে, কিন্তু API fail করছে — মানে app layer alive, but struggling।

---

**`৮:১০ PM` — Database Layer**

ডাটাবেজ: MySQL

```sql
SHOW STATUS LIKE 'Threads_running';
```

```
Threads_running = 10 (always maxed out)
```

এটা clear signal — সব connection busy, নতুন request ঢুকতেই পারছে না।

---

**`৮:১২ PM` — Deeper Analysis**

| Parameter            | Value        |
| -------------------- | ------------ |
| Connection pool size | `10`         |
| Avg query latency    | `~500ms`     |

Capacity calculation:

```
10 / 0.5 = 20 QPS
```

Incoming traffic ~750 req/sec — কিন্তু system handle করতে পারছে মাত্র 20।

```
✅ Processed  →   20/sec
❌ Queued     →  ~730/sec
```

10টা connection আছে, প্রতিটা একটা query শেষ করতে 0.5s নেয়। তাহলে 1 সেকেন্ডে সর্বোচ্চ 20টা query serve করা সম্ভব।

---

**`৮:১৫ PM` — Root Cause Identified**

সমস্যা pool size **না**।

আসল সমস্যা — **Slow queries (high latency)**:

```
Missing index → Full table scan → ভারী JOIN
```

**প্রতিটা query বেশি সময় নিচ্ছে → connection ধরে রাখছে → pool blocked।**

---

#### 🛠️ Fix

**Step 1 — Slow query identify করুন**

```sql
SHOW FULL PROCESSLIST;
```

---

**Step 2 — Query optimize করুন + index দিন**

```sql
CREATE INDEX idx_user_id ON orders(user_id);
```

---

**Step 3 — Latency drop করুন**

```
Before: ~500ms
After:  ~10ms
```

---

**Step 4 — Capacity recalculate**

```
10 / 0.01 = 1000 QPS
```

Same infra, same pool — কিন্তু এখন 50x বেশি throughput।

---

#### 🧠 Takeaway

* Pool size বাড়ানো quick fix, কিন্তু real solution না
* Slow query = hidden bottleneck
* `Threads_running == pool size` → DB saturated-এর signal
* Latency কমালে একই infra দিয়ে huge scale handle করা যায়

---

এই ঘটনা একটা জিনিস clear করে — system down হয় না sudden load-এ, down হয় inefficient query-তে।
