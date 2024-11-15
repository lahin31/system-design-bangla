## HTTP 1

HTTP 1 ১৯৯৬ সালে এসেছে। এটি মূলত TCP এর উপর নির্ভর করে তৈরী করা হয়েছিল। এখানে প্রতিটি HTTP রিকোয়েস্ট এর জন্য ডেডিকেটেড TCP connection তৈরী হত। যেমন একটি পেজ এ যদি ১০ টি জাভাস্ক্রিপ্ট লোড করা লাগে তাহলে এই ১০ টি জাভাস্ক্রিপ্ট এর জন্য ১০ টি TCP Connection তৈরী হবে। 

## HTTP 1.1

HTTP 1.1 এ keep-alive mechanism তৈরী করা হয়েছিল যার মাধ্যমে একটি TCP কানেকশন কে একাধিক HTTP request এবং response এ reuse করা যেত, যাকে Pipelining বলে। Pipelining মূলত Sequentially হয়ে থাকে।

একটি TCP connection reuse করে করে আমরা সব content/request করতে পারি।

আর আমাদের content request এর জন্য TCP connection প্রয়োজন, TCP connection নতুন করে তৈরী হচ্ছে না তবে concurrent requests বেশি হওয়ার কারণে keep-alive scalable না।

HTTP 1.1 এ CORS mechanism আনা হয়েছিল।

## HTTP 2

HTTP 2 তে Multiplexing আনা হয়েছিল যার মানে একাধিক STREAMS of REQUESTS একটি TCP connection এর ভিতর transfer হতে পারে। HTTP 1 এ প্রতিটা কনটেন্ট request এর জন্য একটি reuse করা TCP connection ব্যবহার করা হত কিন্তু HTTP 2 তে একাধিক রিকোয়েস্ট একটি single TCP connection এর মধ্যে যেত।

<p align="center">
  <img src="./images/http-2.png" alt="http 2">
</p>

HTTP 2 এর জন্য TLS setup বাধ্যতামূলক, মানে HTTPS enable থাকতে হবে।

HTTP 2 এর একটি feature হল HPACK। HPACK মূলত HTTP headers কে compress করে থাকে। HTTP 1 এ headers plain-text ফরম্যাটে send করা হত, HPACK efficiently headers transmit করে থাকে।

HTTP 2 এর সবচেয়ে বড় Limitation হল, যখন Stream of Requests একটি single TCP connection এর মধ্য দিয়ে যায় তখন কোনো কারণে যদি একটি রিকোয়েস্টের Packet Loss হয় তাহলে সব রিকোয়েস্ট এর উপর নেগেটিভ ইমপ্যাক্ট পড়বে। তার মূল কারণ TCP এর কাজ হল একই order মেইনটেইন করে সব packet/request এক end থেকে অন্য end এ ডেলিভার করা, এখন যদি একটিও packet loss হয় তাহলে TCP আর order maintain করতে পারবে না।

এই সমস্যাকে Head of Line Blocking বলে। 

## HTTP 3

এই Head of Line Blocking সমস্যা সমাধানের জন্য Google একটি নতুন প্রোটোকল তৈরী করেছিল যার নাম QUIC(Quick UDP Internet Connections)। এখানে আমরা TCP ব্যবহার না করে QUIC ব্যবহার করলে যা সুবিধাটি পাবো তা হল যখন packet loss হবে তখন অন্য packet/request এ কোনো প্রভাব ফেলবে না।

<p align="center">
  <img src="./images/quic.png" alt="QUIC">
</p>

QUIC মূলত UDP প্রটোকলের উপর ব্যবহার হয়ে থাকে। সেজন্য এখানে TCP মত Reliability নিশ্চিত করা যায় না। 

এখানে TCP এর মত 3-way handshake তৈরী হয় না, QUIC নিজের initial handshake এ সব encryption সহ TLS এর কাজ হয়ে থাকে। 

<p align="center">
  <img src="./images/quic-2.png" alt="QUIC">
</p>

