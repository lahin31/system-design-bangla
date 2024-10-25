## Session based Authentication এবং Token based Authentication

এই দুই Authentication টেকনিক আমাদের নিজেদের জানা অত্যন্ত গুরুত্বপূর্ণ।

### Session based Authentication

এক্ষেত্রে Authentication করার সময় session ইনফরমেশন/তথ্য ডাটাবেসে কিংবা Session Store এ রাখা হয়। কিভাবে কাজ করে?

- ফ্রন্টএন্ড থেকে ব্যাকএন্ড এ user লগইন রিকোয়েস্ট করবে।
- username কিংবা email এবং password যখন সঠিক হবে, সার্ভার তখন Session তৈরী করে থাকে একটি নির্দিষ্ট Secret Key এর মাধ্যমে। তারপর সেই Session কোনো ডাটাবেস টেবিলে কিংবা Session Store এ স্টোর করে রেখে দেয়।
- তারপর সার্ভার একটি unique Session ID সম্বলিত cookie ক্লায়েন্ট এর কাছে পাঠাবে।
- পরবর্তী সময় ক্লায়েন্ট যখন কোনো প্রোটেক্টেড রিসোর্স এর জন্য সার্ভারের কাছে রিকোয়েস্ট পাঠাবে, তখন সার্ভার সেই রিকোয়েস্ট এর মধ্যে থাকা Session ID কে যাচাই করবে(তা হতে পারে কোনো ডাটাবেস টেবিল থেকে কিংবা কোনো Session Store থেকে)।

যেহেতু Session কোনো স্থানে স্টোর করে রাখা হয় সেজন্য Session based Authentication কে Stateful বলা হয়।

### Token based Authentication

এক্ষেত্রে Authentication করার সময় session ইনফরমেশন/তথ্য ডাটাবেসে কিংবা Session Store এ রাখা হয় না। কিভাবে কাজ করে?

- ফ্রন্টএন্ড থেকে ব্যাকএন্ড এ user লগইন রিকোয়েস্ট করবে।
- username কিংবা email এবং password যখন সঠিক হবে, সার্ভার তখন Private Key এর মাধ্যমে একটি Token তৈরী করবে। Token টি কে সাধারণত JSON Web Token বলে। এটি মূলত একটি Signed Token মানে এতে অনেক ইনফরমেশন থাকে এক প্রকারের এনক্রিপ্ট অবস্থায়।
- তারপর সার্ভার JWT সম্বলিত cookie ক্লায়েন্ট এর কাছে পাঠাবে।
- পরবর্তী সময় ক্লায়েন্ট যখন কোনো প্রোটেক্টেড রিসোর্স এর জন্য সার্ভারের কাছে রিকোয়েস্ট পাঠাবে, তখন সার্ভার সেই রিকোয়েস্ট এর মধ্যে থাকা JWT কে যাচাই করবে সেই Private Key এর মাধ্যমে।

যেহেতু Session কোনো স্থানে স্টোর করে রাখা হয় না সেজন্য Token based Authentication কে Stateless বলা হয়।

## কখন Session based Authentication/Token based Authentication ব্যবহার করব?

- যে সিস্টেমে user ট্র্যাক করার প্রয়োজন হয় সেখানে Session based Authentication ব্যবহার করবো।
- সিষ্টেম যদি sensitive data হ্যান্ডেল করে থাকে তাহলে Session based authentication ব্যবহার করা যায়।
- অন্যান্য ক্ষেত্রে Token based authentication ব্যবহার করা যায়।
