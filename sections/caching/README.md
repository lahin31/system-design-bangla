## Distributed Cahing

এটি একটি সিস্টেম যেখানে একাধিক ক্যাশিং সার্ভার থাকবে এবং কোনো নেটওয়ার্কের একাধিক নোডে বার বার আসা সেই রেস্পন্সের রিকোয়েস্টকে দ্রুত রেসপন্সটি একাধিক নোডে ডিস্ট্রিবিউট করতে পারে।

<p align="center">
  <img src="./images/distributed_caching.png" alt="Distributed Caching" />
</p>

### কেন আমাদের Distributed Cahing এর প্রয়োজন?

আমাদের সিস্টেমকে স্কেল করতে। আমাদের ক্যাশিং সিস্টেমকে Resilient/Fault Tolerant করতে আমাদের Distributed Cahing প্রয়োজন।

## Caching Strategy

৫ প্রকারের Caching Strategy বিদ্যমান।

- Cache-aside: Application সার্ভার প্রথমে ক্যাশিং সার্ভারকে রিকোয়েস্ট করবে,
  - যদি ডেটা ক্যাশিং সার্ভারে বিদ্যমান থাকে তাহলে ক্যাশিং সার্ভার সেই ডেটা Application সার্ভারে রিটার্ন করবে।
  - যদি ডেটা ক্যাশিং সার্ভারে বিদ্যমান না থাকে তাহলে Application সার্ভার Database হিট করবে এবং ডেটা সংগ্রহ করে ক্যাশিং সার্ভারে রাখবে এবং তারপর সেই ডেটা ইউজার কে রিটার্ন করবে।

<p align="center">
  <img src="./images/cache_aside.png" alt="Cache Aside" />
</p>
