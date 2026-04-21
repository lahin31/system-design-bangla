## Cache Invalidation

আমরা ক্যাশিং সার্ভারে সবসময়ের জন্য ডেটা স্টোর করে রাখতে পারবো না, একটি নির্দিষ্ট সময় পর ক্যাশিং সার্ভারের ডেটা খালি করে, নতুন ডেটা ক্যাশিং সার্ভারে স্টোর করে রাখতে হবে, এই পদ্ধতি হল Cache Invalidation।

আমরা যদি Cache Invalidation না করি তাহলে আমরা ক্যাশিং সার্ভারে Stale ডেটা পাব, কারণ ডাটাবেসে যদি নতুন ডেটা থাকে তখন ক্যাশিং সার্ভারে পুরোতন ডেটা থাকবে। যার ফলে ক্লায়েন্ট সবসময় পুরোনো ডেটা পাবে। এই জিনিসটি avoid করতে Cache Invalidation করা হয়।

Invalidate করার একটি পদ্ধতি হল, একটি নির্দিষ্ট সময়সীমা সেট করে দেয়া এই নির্দিষ্ট সময়সীমার পর ডেটা ক্যাশিং সার্ভার থেকে ডিলিট হয়ে যাবে, এই নির্দিষ্ট সময়সীমাকে TTL(Time to live) বলে। আরেকটি পদ্ধতি হল, যখনই নতুন রিকোয়েস্ট আসবে তখন ডাটাবেস আপডেট হওয়ার পাশাপাশি আমরা ক্যাশিং সার্ভারে Cache Invalidation করব।

## Cache Eviction

এটি একটি প্রসেস যার মাধ্যমে ক্যাশিং সার্ভার থেকে ডেটা বাদ দেয়া হয় যাতে করে নতুন ডেটা সংরক্ষণ করতে পারে। ক্যাশিং সার্ভারে সবসময় একটি Limited Capacity থাকে, যখন ক্যাশিং সার্ভার সম্পূর্ণভাবে ভরে যায় তখন এই Cache Eviction পদক্ষেপ নেয়া লাগে, যাতে করে নতুন ডেটা ক্যাশিং সার্ভারে সংরক্ষন করা যায়।

### Cache Eviction Policy

- Least Recently Used (LRU): এই স্ট্রাটেজিতে সম্প্রতি ব্যবহৃত না হওয়া ডেটাগুলোকে ক্যাশিং সার্ভারে থেকে বাদ দেয়া হয়।

```js
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
  }

  get(key) {
    if (!this.cache.has(key)) return null;
    const val = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, val);
    return val;
  }

  put(key, value) {
    if (this.cache.has(key)) this.cache.delete(key);
    this.cache.set(key, value);
    if (this.cache.size > this.capacity) {
      const oldestKey = this.cache.keys().next().value;
      this.cache.delete(oldestKey);
    }
  }
}

// আমরা মাত্র ১০০টি প্রোডাক্ট মেমোরিতে রাখবো
const productCache = new LRUCache(100);

async function getProductDetails(productId) {
  // ১. প্রথমে ক্যাশে চেক করি
  let product = productCache.get(productId);

  if (product) {
    console.log("🚀 ক্যাশ থেকে পাওয়া গেছে (Fast!)");
    return product;
  }

  // ২. ক্যাশে না থাকলে MySQL (Prisma) থেকে আনি
  console.log("🐢 ডাটাবেস থেকে আনা হচ্ছে (Slower...)");
  product = await prisma.product.findUnique({
    where: { id: productId }
  });

  // ৩. ভবিষ্যতে দ্রুত পাওয়ার জন্য ক্যাশে রেখে দিই
  if (product) {
    productCache.put(productId, product);
  }

  return product;
}

// ব্যবহার:
// প্রথমবার: ডাটাবেস হিট করবে
// দ্বিতীয়বার: সরাসরি ক্যাশ থেকে রিটার্ন করবে (ভীষণ দ্রুত!)
```

- First-in-first-out (FIFO): এই স্ট্রাটেজিতে যে ডেটা ক্যাশিং সার্ভারে প্রথমে প্রবেশ করবে সেই ডেটা প্রথমে বাদ যাবে।

```js
class FIFOQueue {
  constructor() {
    this.queue = [];
  }

  // লাইনে নতুন আইটেম যোগ করা (Enqueue)
  enqueue(item) {
    this.queue.push(item);
    console.log(`${item} লাইনে যোগ করা হয়েছে।`);
  }

  // লাইন থেকে প্রথম আইটেমটি বের করা (Dequeue)
  dequeue() {
    if (this.isEmpty()) {
      return "লাইনটি খালি!";
    }
    const item = this.queue.shift(); // প্রথম আইটেমটি সরিয়ে নেয়
    console.log(`${item} প্রসেস করা হয়েছে।`);
    return item;
  }

  isEmpty() {
    return this.queue.length === 0;
  }
}

// ব্যবহার:
const myLine = new FIFOQueue();
myLine.enqueue("Task 1");
myLine.enqueue("Task 2");
myLine.enqueue("Task 3");

myLine.dequeue(); // আউটপুট: Task 1 প্রসেস করা হয়েছে।
console.log(myLine.queue); // আউটপুট: ['Task 2', 'Task 3']
```

- Least Frequently Used (LFU): এই স্ট্রাটেজিতে যে ডেটা যত কমবার ব্যবহৃত হয়েছে সেই ডেটাগুলোকে ক্যাশিং সার্ভারে থেকে বাদ দেয়া হয়।

```js
class LFUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.values = new Map();     // কী এবং ভ্যালু রাখার জন্য
    this.counts = new Map();     // প্রতিটি কী কতবার ব্যবহৃত হয়েছে
    this.freqMap = new Map();    // ফ্রিকোয়েন্সি অনুযায়ী কী-গুলোর সেট
    this.minFreq = 0;            // বর্তমানে সবচেয়ে কম ফ্রিকোয়েন্সি কত
  }

  get(key) {
    if (!this.values.has(key)) return null;

    // ফ্রিকোয়েন্সি আপডেট করা
    const count = this.counts.get(key);
    this.counts.set(key, count + 1);

    // পুরোনো ফ্রিকোয়েন্সি সেট থেকে কী-টি সরানো
    this.freqMap.get(count).delete(key);
    
    // যদি এই কাউন্টটিই মিনিমাম ছিল এবং এখন খালি হয়ে যায়
    if (count === this.minFreq && this.freqMap.get(count).size === 0) {
      this.minFreq++;
    }

    // নতুন ফ্রিকোয়েন্সিতে যোগ করা
    if (!this.freqMap.has(count + 1)) {
      this.freqMap.set(count + 1, new Set());
    }
    this.freqMap.get(count + 1).add(key);

    return this.values.get(key);
  }

  put(key, value) {
    if (this.capacity <= 0) return;

    if (this.values.has(key)) {
      this.values.set(key, value);
      this.get(key); // ফ্রিকোয়েন্সি আপডেট করতে get() কল করা হলো
      return;
    }

    if (this.values.size >= this.capacity) {
      // সবচেয়ে কম ফ্রিকোয়েন্সির সেট থেকে প্রথম আইটেমটি ডিলিট করা
      const list = this.freqMap.get(this.minFreq);
      const keyToRemove = list.values().next().value;
      list.delete(keyToRemove);
      this.values.delete(keyToRemove);
      this.counts.delete(keyToRemove);
    }

    // নতুন আইটেম সেট করা
    this.values.set(key, value);
    this.counts.set(key, 1);
    this.minFreq = 1;
    if (!this.freqMap.has(1)) this.freqMap.set(1, new Set());
    this.freqMap.get(1).add(key);
  }
}

// ব্যবহার:
const lfu = new LFUCache(2);
lfu.put("a", 1);
lfu.put("b", 2);
lfu.get("a");    // 'a' এর ফ্রিকোয়েন্সি ২
lfu.put("c", 3); // 'b' মুছে যাবে কারণ তার ফ্রিকোয়েন্সি ১ (সবচেয়ে কম)
```

## Distributed Cahing

এটি একটি সিস্টেম যেখানে একাধিক ক্যাশিং সার্ভার থাকবে এবং কোনো নেটওয়ার্কের একাধিক নোডে বার বার আসা সেই রেস্পন্সের রিকোয়েস্টকে দ্রুত রেসপন্সটি একাধিক নোডে ডিস্ট্রিবিউট করতে পারে।

<p align="center">
  <img src="./images/distributed_caching.png" alt="Distributed Caching" />
</p>

### কেন আমাদের Distributed Cahing এর প্রয়োজন?

আমাদের সিস্টেমকে স্কেল করতে। আমাদের ক্যাশিং সিস্টেমকে Resilient/Fault Tolerant করতে আমাদের Distributed Cahing প্রয়োজন।

## Caching Strategy

৫ প্রকারের Caching Strategy বিদ্যমান।

### Cache-aside:

- প্রথমে Client Application সার্ভারে request করবে, তারপর Application সার্ভার প্রথমে ক্যাশিং সার্ভারকে হিট করবে,,
- যদি ডেটা ক্যাশিং সার্ভারে বিদ্যমান থাকে তাহলে ক্যাশিং সার্ভার সেই ডেটা Application সার্ভারকে দিবে এবং Application সার্ভার সেই ডেটা Client কে দিবে।
- যদি ডেটা ক্যাশিং সার্ভারে বিদ্যমান না থাকে তাহলে Application সার্ভার Database হিট করবে এবং ডেটা সংগ্রহ করে ক্যাশিং সার্ভারে রাখবে এবং তারপর সেই ডেটা ইউজার কে রিটার্ন করবে।

<p align="center">
  <img src="./images/cache_aside.png" alt="Cache Aside" />
</p>

#### Pros of cache-aside

- read-heavy application এর জন্য উত্তম।
- Cache server ডাউন থাকলেও request ব্যর্থ হওয়ার সুযোগ নাই, যেহেতু তখন ডাটাবেসে থেকে ডেটা নিবে।

#### Cons of cache-aside

- নতুন ডেটা Read এর ক্ষেত্রে সবসময় cache miss হবে।

Memcached এবং Redis, cache-aside caching strategy follow করে।

### Read-through:

- প্রথমে Client Application সার্ভারে request করবে, তারপর Application সার্ভার প্রথমে ক্যাশিং সার্ভারকে হিট করবে,
- যদি ডেটা ক্যাশিং সার্ভারে বিদ্যমান থাকে তাহলে ক্যাশিং সার্ভার সেই ডেটা Application সার্ভারকে দিবে এবং Application সার্ভার সেই ডেটা Client কে দিবে।
- যদি ডেটা ক্যাশিং সার্ভারে বিদ্যমান না থাকে তাহলে ক্যাশিং সার্ভার Database হিট করবে এবং ডেটা সংগ্রহ করে ক্যাশিং সার্ভারে রাখবে এবং তারপর সেই ডেটা Application সার্ভারকে রিটার্ন করবে।
- তারপর Application সার্ভার সেই ডেটা'টি ক্লায়েন্টকে দিবে।

<p align="center">
  <img src="./images/read_through.png" alt="Read-through" />
</p>

#### Pros of Read-through

- read-heavy application এর জন্য উত্তম।
- Application Server এ কোনো প্রকারের Data Fetching হবে না।

#### Cons of Read-through

- নতুন ডেটা Read এর ক্ষেত্রে সবসময় cache miss হবে।

### Write Around

- প্রথমে Client Application সার্ভারে write রিকোয়েস্ট করবে।
- Application Server সরাসরি ডাটাবেসে value update করবে।
- Application Server ক্যাশিং সার্ভারে ডাটা Dirty হিসেবে যোগ করবে। যাতে করে পরবর্তী সময় Read রিকোয়েস্ট আসলে বুজা যায় ডাটা Stale।

<p align="center">
  <img src="./images/write_around.png" alt="Write Around" />
</p>

#### Pros of Write Around

- Inconsistency issue resolve করে between Cache and Database।

(বিস্তারিত চলমান)
