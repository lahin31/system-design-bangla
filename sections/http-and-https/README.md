## HTTP 1

HTTP 1 ১৯৯৬ সালে এসেছে। এটি মূলত TCP এর উপর নির্ভর করে তৈরী করা হয়েছিল। এখানে প্রতিটি HTTP রিকোয়েস্ট এর জন্য ডেডিকেটেড TCP connection তৈরী হত। যেমন একটি পেজ এ যদি ১০ টি জাভাস্ক্রিপ্ট লোড করা লাগে তাহলে এই ১০ টি জাভাস্ক্রিপ্ট এর জন্য ১০ টি TCP Connection তৈরী হবে। 

## HTTP 1.1

HTTP 1.1 এ keep-alive mechanism তৈরী করা হয়েছিল যার মাধ্যমে একটি TCP কানেকশন কে একাধিক HTTP request এবং response এ reuse করা যেত, যাকে Pipelining বলে। Pipelining মূলত Sequentially হয়ে থাকে।

এই Pipelining থাকার পরেও দেখা গেল দিন দিন জাভাস্ক্রিপ্ট, images এর পরিমান দিন দিন বাড়তে লাগলো। একটি TCP connection reuse করে আমরা content টি পেয়ে যেতে পারি কিন্তু তবুও প্রতিটি content এর জন্য একটি করে TCP connection ব্যবহার করতে হবে।  

HTTP 1.1 এ CORS mechanism আনা হয়েছিল।

## HTTP 2

HTTP 2 তে Multiplexing আনা হয়েছিল যার মানে একাধিক STREAMS of REQUESTS একটি TCP connection এর ভিতর transfer হতে পারে। HTTP 1 এ প্রতিটা কনটেন্ট request এর জন্য একটি reuse করা TCP connection ব্যবহার করা হত কিন্তু HTTP 2 তে একাধিক রিকোয়েস্ট একটি single TCP connection এর মধ্যে যেত। । 

<p align="center">
  <img src="./images/http-2.png" alt="http 2">
</p>

HTTP 2 এর জন্য TLS setup বাধ্যতামূলক, মানে HTTPS enable থাকতে হবে।

HTTP 2 এর একটি feature হল HPACK। HPACK মূলত HTTP headers কে compress করে থাকে। HTTP 1 এ headers plain-text ফরম্যাটে send করা হত, HPACK efficiently headers transmit করে থাকে। 

## HTTP 3

(বিস্তারিত চলমান)
