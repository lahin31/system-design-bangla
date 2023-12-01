## Replication Lag

সর্বমোট যত সময় Master Database এর update কিংবা insert গুলোকে Replica ডাটাবেসগুলোতে replicate করতে লাগে, সেই সময়টি হচ্ছে Replication Lag।

Replication Lag এর জন্য একটি সমস্যা তৈরী হয়। যেমন, Master Database এ update কিংবা insert হলে তা replica ডাটাবেসগুলোতে replicate করতে গেলে যদি Replication Lag এর সময় বেশি হয়, তাহলে replicate করার সময় যদি read (replica database এ) read request আসে তখন different value অর্থাৎ inconsistent data রিড করার সুযোগ থাকে। 

<p align="center">
  <img src="./images/db-replication-1.png" alt="replication">
</p>

## Benefits of Database Replication

Database Replication এর সুবিধা,

- Performance: যেহেতু সবধরনের Write, Update and Delete Master Database এ হবে এবং সবধরনের Read Slave Database এ হবে সেহেতু একাধিক Query Parallely সংঘটিত হবে যার ফলে Performance Better হবে। 

- Reliability: যদি কোনো Database Server নষ্ট হয়ে যায় তবুও আমরা সেই ডেটা অন্য কোনো Database Server এ পাব। ডেটা নষ্ট হওয়ার কোনো সুযোগ নেই।

- Availability: কোনো Database Server নষ্ট হয়ে যায় তখন আমরা অন্য Database Server থেকে ডেটা নিয়ে আমাদের System/Website কে সচল রাখতে পারি।

- Reduce Latency: একাধিক Database Server(Slave Database) থাকার ফলে Latency Time reduce হয়। 

## Some facts

- কোনো কারণে যেকোনো Slave Database নষ্ট হয়ে গেলে অন্য Slave Database থেকে ডেটা নেয়া যায়।

- Master Database নষ্ট হয়ে গেলে, কোনো Slave Database সেই নষ্ট Master Database কে Replace করবে। পরে নতুন Slave Database তৈরি হয়ে পুরনো Slave Database কে Replace করবে।
