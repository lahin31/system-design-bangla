## কেন Rate Limiter?

- Denial of Service(DoS) attack প্রতিরোধ করতে পারি।
- অপ্রয়োজনীয় রিকোয়েস্ট block করার মাধ্যমে আমরা cost reduce করতে পারি।

## কোথায় Rate Limiter ইমপ্লিমেন্ট করবো?

আমরা চাইলে client-side কিংবা server-side এ ইমপ্লিমেন্ট করতে পারবো। তবে server-side এ ইমপ্লেমেট করা better। কারন client-side এত reliable না।

## HTTP Rate Limiter

<p align="center">
  <img src="./images/http-rate-limiter.png" alt="http rate limiter">
</p>

যখন ক্লায়েন্ট নির্দিষ্ট সময়ের ভিতর নির্দিষ্ট পরিমাণ এর থেকে বেশি http রিকোয়েস্ট সেন্ড করে তখন HTTP status code 429 যার মানে To many request পাঠিয়ে দেয়া হয়।

## Rate Limiter এর Throttle কিসের উপর নির্ভর করে তৈরী করবো?

আপনি চাইলে নির্দিষ্ট IP ধরে কিংবা নির্দিষ্ট user ID ধরে rate limiter এর throttle তৈরী করতে পারেন। IP ধরে করলে করলে নির্দিষ্ট সময়ের ভিতর নির্দিষ্ট IP থেকে নির্দিষ্ট পরিমাণের থেকে বেশি রিকোয়েস্ট আসলে তা block হয়ে যাবে।
