## কখন SQL কিংবা NoSQL ব্যবহার করব?

SQL এবং NoSQL কিছু বৈশিষ্ট্য আছে যার উপর নির্ভর করে আমরা সিদ্ধান্ত নিতে পারব কোন সময় আমরা কোনটি ব্যবহার করব,

- Predetermined Schema: SQL দিয়ে আমরা কোনো operation (read, insert, update, delete) চালানোর আগে আমাদের টেবিলের schema Predetermined থাকতে হবে। টেবিলের schema Predetermined বলতে বুজানো হচ্ছে একটি টেবিল যেমন user table এখানে schema হবে,

  - id
  - name
  - email
  - password
  - (ইত্যাদি)

এইগুলো Predetermined মানে পূর্বনির্ধারিত থাকলে, আমরা SQL ভিত্তিক RDBMS Database যেমন MySQL, PostgreSQL ইত্যাদি ব্যবহার করতে পারব।

অপরপক্ষে schema যদি Predetermined না থাকে তাহলে NoSQL ব্যবহার করা যায়। NoSQL এর ডেটা সাধারণত key-value, Document, Graph আকারে ডিস্কে স্টোর হয়ে থাকে, সেজন্য schema ফিক্সড হওয়া লাগে না।

- Scalability: ডেটা অনেক বেশি হয়ে গেলে SQL ভিত্তিক Database গুলোতে ভার্টিকাল স্কেলিং করা হয় মানে storage এর capacity বৃদ্ধি করা। (তাছাড়া Database Sharding-ও করা হয়ে থাকে)

আর NoSQL ভিত্তিক হরাইজন্টাল স্কেলিং করা হয় মানে সার্ভারের Capacity বৃদ্ধি করার পরিবর্তে নতুন সার্ভার যোগ করাই হল হরাইজন্টাল স্কেলিং।

এখন আমাদের সিস্টেম এ কোন রকমের scaling করলে আমাদের system চালাতে পারব তার উপর ভিত্তি করে সিদ্ধান্ত নিব।

- ACID: SQL ভিত্তিক Database সাধারণত ACID property follow করে থাকে। যেখানে A মানে Atomicity, C মানে Consistency, I মানে Isolation এবং D মানে Durability। এগুলোর উদ্দেশ্য হল Data Integrity এবং Consistency বজায় রাখা। যেমন Banking Software/ATM Vendor Machine এগুলোর জন্য ACID খুব গুরুত্বপূর্ণ।

NoSQL ভিত্তিক Database ACID properties সম্পূর্ণ সাপোর্ট করে না।

এখন আমাদের সিস্টেমে Data Integrity এবং Consistency বজায় রাখতে চাইলে আমরা SQL ভিত্তিক Database ব্যবহার করব, না হয় NoSQL ভিত্তিক Database।

## Database Performance

Database Performance সিস্টেম ডিজাইনে খুবই গুরুত্বপূর্ণ বিষয়। এটি একটি খুবই বড় বিষয়, সেজন্য এখানে শুধু বিষয় Index নিয়ে সংক্ষিপ্ত আলোচনা করা হল,

Database Indexing একটি সাধারণ টেকনিক যা আমাদের Database Query কে দ্রুত সম্পন্ন করে থাকে।

সাধারণত ডেটা Disk-এ সংরক্ষন হয়ে থাকে। যখন ডেটা বেড়ে যায় তখন সেই ডেটাগুলো থেকে Query করতে অনেক সময় লাগে, এই সময় কমানোর জন্য আরেকটি টেবিল Disk-এ তৈরী হয় যাকে Index Table বলে। এই Index Table-এ মূলত আমাদের মূল টেবিল এর row(s) এর সাথে একটি লিংক করা থাকে, সেটি key-value আকারেও থাকতে পারে। যখন নতুন row কিংবা entry ডেটাবেস টেবিলে insert হয়, Index Table-এ সেই নতুন ডেটার সাথে একটি লিংক তৈরী হয় (সেজন্য আমাদের write operation slow হয়ে যেতে পারে)।

পরবর্তী সময়ে যখন কেউ নির্দিষ্ট ডেটা query করবে, তখন Index Table বলে দিবে কোন এড্ড্রেসে বা কোন ব্লকে ডেটা আছে।

বিস্তারিত জানতে চাইলে আমার ব্লগ দেখতে পারেন,

- https://codemacaw.com/2023/04/04/database-indexing-makes-db-query-faster/
- https://codemacaw.com/2023/06/24/what-is-b-tree-b-tree-in-dbms/
