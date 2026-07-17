# ব্যাকএন্ড ইন্টারভিউ প্রশ্ন ও উত্তর

## "Logout from All Devices" ফিচারটা কিভাবে Implement করা হয়?

সাধারণত User একাধিক Device (মোবাইল, ব্রাউজার, ল্যাপটপ) থেকে লগইন করে থাকে, প্রতিটা Login এর জন্য আলাদা Token ইস্যু হয়। "Logout from all devices" মানে হচ্ছে একসাথে সবগুলো Device এর সব Token কে একবারে Invalidate করে দেয়া।

এটা Implement করার কয়েকটা common approach আছে,

- Session-based Auth: যদি Server side এ Session/Token গুলো Database বা Redis এ store করা থাকে (userId অনুযায়ী), তাহলে logout all এর সময় শুধু সেই userId এর সব session/token রেকর্ড delete করে দিলেই হয়ে যায়। পরের রিকোয়েস্টে Server যখন Token verify করতে যাবে, Database এ না পেয়ে reject করে দিবে।

- Stateless JWT এর ক্ষেত্রে: প্রতিটা User এর জন্য একটা tokenVersion বা sessionVersion নাম্বার রাখা Database এ, এবং সেই ভ্যালুটা JWT payload এর মধ্যেও রাখা হয়। Logout all করলে শুধু Database এ ওই ইউজারের tokenVersion টা +১ increment করে দিলেই হয়। Token verify করার সময় প্রতিবার JWT এর tokenVersion আর Database এর current tokenVersion মিলিয়ে দেখা হয়, না মিললে Token কে invalid ধরা হয় (পুরনো সব JWT একসাথে অকেজো হয়ে যায়, নতুন login এ নতুন version দিয়ে নতুন Token ইস্যু হয়)।

## ধরুন কেউ JWT এর Payload decode করে role: user পরিবর্তন করে role: admin করে দিল। তাহলে কি সে Admin Access পেয়ে যাবে, নাকি Server সেটা ধরে ফেলবে?

না, শুধুমাত্র Payload পরিবর্তন করলেই কেউ Admin হয়ে যেতে পারবে না।

Server যখন কোনো JWT পায়, তখন Header এবং Payload নিয়ে আবার নিজের Secret/Public Key দিয়ে একটা নতুন Signature বানায়, এবং সেটা token এর সাথে আসা Signature এর সাথে মিলিয়ে দেখে। দুটো মিললেই Server নিশ্চিত হয় token টা valid এবং কেউ Tamper করেনি।

কেউ যদি Payload এর ভিতরের ডাটা (যেমন role: user কে role: admin বানিয়ে) পরিবর্তন করে, তাহলে Signature আর মিলবে না (কারণ Signature টা আগের Payload দিয়ে বানানো হয়েছিল), Server সাথে সাথে সেই Token কে Invalid/Unauthorized ধরে reject করে দিবে।

## Password কেন Hash করা হয়, Encrypt করা হয় না কেন?

Encryption একটা Two-way প্রসেস, মানে Encrypt করা ডাটাকে সঠিক Key দিয়ে আবার Decrypt করে আসল ডাটায় ফেরত আনা যায়। অন্যদিকে, Hashing একটা One-way প্রসেস, মানে Hash থেকে আসল ডাটা (Plain Text) ফেরত পাওয়ার কোনো সরাসরি উপায় নেই।

Password-এর ক্ষেত্রে আমাদের আসল Password পরে পড়ার বা ফেরত পাওয়ার প্রয়োজন হয় না; শুধু Login-এর সময় ব্যবহারকারীর দেওয়া Password সঠিক কিনা সেটি যাচাই করাই যথেষ্ট। তাই Password Database-এ সংরক্ষণের জন্য Encryption-এর পরিবর্তে Hashing ব্যবহার করা হয়।

যদি Password Encryption ব্যবহার করে সংরক্ষণ করা হতো, তাহলে Server-কে Decryption Key সংরক্ষণ করতে হতো। সেই Key কোনোভাবে ফাঁস হয়ে গেলে বা চুরি হয়ে গেলে, Database-এর সব Password সহজেই উদ্ধার করা সম্ভব হতো।

কিন্তু Hashing ব্যবহার করলে Database Leak হলেও Attackers সরাসরি User-এর আসল Password জানতে পারে না। Login-এর সময় User যে Password দেয় সেটিকে আবার Hash করে Database-এ সংরক্ষিত Hash-এর সাথে মিলিয়ে দেখা হয়। দুইটি Hash মিলে গেলে User-কে Authenticate করা হয়।

## JWT ব্যবহারের সাথে জড়িত Security Risk গুলো কি কি?

JWT Stateless Authentication-এর জন্য খুবই জনপ্রিয়, কিন্তু ভুলভাবে ব্যবহার করলে এটি বড় ধরনের Security Risk তৈরি করতে পারে।

- **Token চুরি হওয়া (XSS/Token Theft)**: Token যদি Browser এর LocalStorage এ রাখা হয়, XSS Attack এর মাধ্যমে সেটা চুরি হতে পারে। এজন্য Token কে HttpOnly এবং Secure Cookie তে রাখা বেশি নিরাপদ, যাতে JavaScript দিয়ে সরাসরি Access করা না যায়।

- **alg: none Attack**: পুরনো কিছু Library তে JWT এর Header এ Algorithm none সেট করে দিলে Signature Verification skip হয়ে যেতো। এখন প্রায় সব Modern Library তে এটা Fix করা আছে, তবুও Verify করার সময় Server কে অবশ্যই নির্দিষ্ট করে বলে দিতে হবে সে কোন Algorithm Expect করছে।

- **Sensitive Data Payload এ রাখা**: Payload শুধু Base64 Encoded, Encrypted না, তাই এখানে Password বা Sensitive তথ্য রাখা উচিত না, যে কেউ Decode করে পড়ে ফেলতে পারবে।

- **Long Expiry রাখা**: Access Token এর Expiry অনেক বড় রাখলে, চুরি হলে সেটা অনেক লম্বা সময় ধরে ব্যবহারযোগ্য থাকে। তাই ছোট Expiry + Refresh Token pattern ব্যবহার করাই ভালো।

