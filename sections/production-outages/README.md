# Production Outages

## সূচিপত্র

- [Database Scenarios](#database-scenarios)
  - [The Silent MySQL Crash](#the-silent-mysql-crash)
  - [Flash Sale Crash: The Pool wasn’t the problem - Query Latency was]


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

(চলমান)