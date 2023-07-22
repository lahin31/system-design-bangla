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
