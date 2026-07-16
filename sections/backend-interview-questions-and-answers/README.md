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