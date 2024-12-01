## Database Sharding এর সুবিধাগুলো

- **improve response time**: ডেটা retrive করা অনেক ফাস্ট হয়ে থাকে। একসাথে লক্ষ্য লক্ষ্য row থেকে ডেটা retrive করতে যত সময় লাগবে, ছোট ছোট সার্ড টেবিল থেকে ডেটা retrive করতে আরো কম সময় লাগবে। এভাবে response time ইম্প্রোভ করা যায়।

- **effective scaling**: মাল্টিপল সার্ড টেবিলে ডিস্ট্রিবিউট করার ফলে scalability ensure হয়ে থাকে।

- **database resiliency**: মাল্টিপল সার্ড টেবিলে ডিস্ট্রিবিউট করার ফলে প্রতিবার যখন একটি সার্ভার(সার্ড) কোন কারনে নষ্ট হয়ে যায়, তখন অন্য সার্ভার(সার্ড) up and running থাকে।

## Sharding Techniques

### Range Based Sharding

Range Based Sharding এ মূলত একটি নির্দিষ্ট রেঞ্জ এর ভিত্তিতে ডেটা বিভিন্ন সার্ড-এ ডিস্ট্রিবিউট হয়ে থাকে।

<p align="center">
  <img src="./images/range-based-sharding.png" alt="range based sharding">
</p>

উপরের ছবিতে দেখা যাচ্ছে, রেঞ্জ ১০ থেকে ২০ id একটি সার্ড-এর মধ্যে থাকবে অপর সার্ড-এ ২১ থেকে ৩৫ id পর্যন্ত ডেটা থাকবে।

### Hash Based Sharding

Hash Based Sharding এ মূলত একটি **হ্যাশ ফাংশন** থাকবে যা বলে দিবে কোন row এর ডেটা কোন সার্ড এ যাবে।

<p align="center">
  <img src="./images/hash-based-sharding.png" alt="hash based sharding">
</p>

উপরের ছবিতে দেখা যাচ্ছে, হ্যাশ ফাংশন প্রসেস করার পর ১০ এবং ১৫ id সার্ড ১ এ থাকবে। বাকিগুলো সার্ড ২ এ।

### Directory Based Sharding

Directory Based Sharding এ মূলত একটি Lookup table থাকবে যা বলে দিবে কোন row এর ডেটা কোন সার্ড এ যাবে।

<p align="center">
  <img src="./images/directory-based-sharding.png" alt="directory based sharding">
</p>

## Sharding in SQL, NoSQL and Cloud

AWS Database Sharding <a href="https://aws.amazon.com/blogs/database/sharding-with-amazon-relational-database-service/" target="_blank">যেভাবে করে</a>

PostgreSQL এ natively Database Sharding সাপোর্ট করে না তবে PostgreSQL 11 Foreign Data Wrappers এর মাধ্যমে আমরা ডাটা বিভিন্ন সার্ভারে ডিস্ট্রিবিউট এবং read করতে পারি।

আমরা Connection Pool এবং Proxy হিসেবে Pgcat ব্যবহার করে আমরা ডেটা বিভিন্ন shard এর মধ্যে ডিস্ট্রিবিউট করতে পারি।

<p align="center">
  <img src="./images/sharding-1.png" alt="sharding">
</p>

## কখন সার্ড করবো না?

(চলমান)

## Sharding এবং Partitioning এর পার্থক্য

Sharding মূলত ডাটাবেসের ডেটাগুলোকে একাধিক সার্ভারের ভিতর একাধিক টেবিল এর মধ্যে ডিস্ট্রিবিউট করে থাকে অন্যদিকে Partition একটি সার্ভারের ভিতর একাধিক টেবিল এর মধ্যে ডিস্ট্রিবিউট করে।
