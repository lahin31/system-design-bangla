## Database Replication

Database Replication এর সুবিধা,

- Performance: যেহেতু সবধরনের Write, Update and Delete Master Database এ হবে এবং সবধরনের Read Slave Database এ হবে সেহেতু একাধিক Query Parallely সংঘটিত হবে যার ফলে Performance Better হবে। 

- Reliability: যদি কোনো Database Server নষ্ট হয়ে যায় তবুও আমরা সেই ডেটা অন্য কোনো Database Server এ পাব। ডেটা নষ্ট হওয়ার কোনো সুযোগ নেই।

- Availability: কোনো Database Server নষ্ট হয়ে যায় তখন আমরা অন্য Database Server থেকে ডেটা নিয়ে আমাদের System/Website কে সচল রাখতে পারি।
