## Polling এর টাইপ

### Short Polling

এটি এক প্রকারের Polling, যেখানে ক্লায়েন্ট একটি সময় পর পর সার্ভারকে রিকোয়েস্ট করবে সার্ভার সেই রিকোয়েস্ট এর রেসপন্স করবে, সার্ভারের কাছে যদি সেই ডেটার রেসপন্স না থাকে তাহলে empty response পাঠাবে।

<p align="center">
  <img src="./images/short-polling.png" alt="short-polling">
</p>

### Long Polling

এখানে ক্লায়েন্ট একটি সময় পর পর সার্ভারকে রিকোয়েস্ট করবে সার্ভার সেই রিকোয়েস্ট এর রেসপন্স করবে, এখন যদি সার্ভার এর কাছে রেসপন্স না থাকে তাহলে সার্ভার একটি নির্দিষ্ট সময় অপেক্ষা করবে নির্দিষ্ট সময়ের ভিতর সার্ভারে ডেটা আসে তাহলে ডেটা সার্ভার রেসপন্স আকারে ক্লায়েন্ট কে দিয়ে দিবে না হয় অপারেশন Timeout হয়ে যাবে,তখন ক্লায়েন্ট আবার নির্দিষ্ট সময় পর সার্ভারকে রিকোয়েস্ট করবে।

<p align="center">
  <img src="./images/long-polling.png" alt="long-polling">
</p>

## Web Socket কিভাবে কাজ করে?

Web Socket একটি single TCP কানেকশন এর মাধ্যমে তৈরী হওয়া Bidirectional Communication Protocol। প্রসেস শুরু হয়,

- TCP Connection: client প্রথমে সার্ভার এর সাথে TCP Connection তৈরী করবে।

- Connection সবসময় open থাকা: TCP Connection তৈরী হওয়ার পরে, কানেকশন সবসময় ওপেন থাকবে। Open কানেকশন এর মধ্যে Data Transmit হবে।

- Connection বন্ধ: যখন কোনো এন্ড(client কিংবা server) নির্দিষ্টভাবে কানেকশন বন্ধ করলে, তখন কানেকশন সিস্টেম বন্ধ হবে। অন্যথায় কানেকশন সবসময় Open থাকবে।

## Server-Sent Events

(চলমান)
