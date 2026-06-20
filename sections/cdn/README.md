## CDN কীভাবে কাজ করে

- **Origin server vs Edge server**: User এর request প্রথমে nearest Edge server-এ যায়, origin সার্ভারে নয়।

- **Cache Hit vs Cache Miss**: Edge server-এ content থাকলে (cache hit) সরাসরি serve হয়; না থাকলে (cache miss) origin থেকে fetch করে cache করে রাখে।

- **TTL (Time To Live)**: কতক্ষণ content cache-এ থাকবে তা নির্ধারণ করে। এক্সপায়ার হলে origin থেকে আবার fetch করে।

- **Push CDN vs Pull CDN**: দুই ধরনের CDN strategy — কে কন্টেন্ট push করে রাখে vs CDN নিজে demand অনুযায়ী pull করে।

```
User → DNS → Nearest Edge Server (CDN)
              ├── Cache Hit → Response সরাসরি ফিরে যায়
              └── Cache Miss → Origin Server → Cache → Response
```

## CDN কী ধরনের Content Cache করে?

সাধারণত Image, CSS, JavaScript, Font, Video ইত্যাদি Static Content CDN-এ cache করা হয়। Dynamic API Response-ও cache করা সম্ভব, তবে সেটি Cache-Control, TTL এবং অন্যান্য caching rule-এর উপর নির্ভর করে।

## CDN এর উপকারিতা

- Reduce Latency: CDN এর ব্যবহারের মাধ্যমে Latency reduce করা যায়। যার ফলে পেইজের লোড টাইম কমে যায়।

- Reduce Bandwidth Cost: Edge server cached content serve করায় একই content বারবার origin server থেকে পাঠাতে হয় না। ফলে origin server-এর bandwidth usage এবং data transfer cost কমে যায়।

- Ensure Content Availability: একটি CDN সার্ভার যদি নষ্ট কিংবা offline হয়ে যায়, তখন অন্য একটি CDN সার্ভার সেই কনটেন্ট ডেলিভার করতে পারবে। এভাবে আমরা Availability ensure করতে পারি।

- Handle DDoS Attack: অনেক CDN provider Edge network-এ malicious traffic filter, rate limiting এবং traffic absorption-এর মাধ্যমে origin server-এর উপর চাপ কমাতে সাহায্য করে।

<p align="center">
  <img src="./images/benefits_of_cdn.png" alt="Benefits of CDN" />
</p>

## CDN এর ব্যবহার: Video & Live Streaming

Video & Live Streaming সিস্টেম যেখানে Videos কিংবা Audio সার্ভ করা হয় সেখানে সাধারণত CDN ব্যবহার করা হয়,
যাতে Bandwidth Cost কমানো যায়, সহজে Scale করা যায় এবং Delivery Time দ্রুত করা যায়। আরো জানতে চাইলে <a href="https://youtu.be/EJQkBd_-CMo" target="_blank">এটি</a> দেখতে পারেন।

## জনপ্রিয় CDN Provider

* Cloudflare
* Akamai
* AWS CloudFront
* Fastly
* Google Cloud CDN