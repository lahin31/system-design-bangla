## How insertion works in bloom filter

- শুরুতে bucket এর সব index এর ভ্যালু 0। 

<p align="center">
  <img src="./images/bf-1.png" alt="bloom filter">
</p>

- `generateHash(10, 'david')` এর রিটার্ন ভ্যালু 5, index 5 এর ভ্যালু 0 থেকে 1 বানানো হল, david এর একটি রেফারেন্স index 5 এ রাখবে। । 

<p align="center">
  <img src="./images/bf-2.png" alt="bloom filter">
</p>

- `generateHash(10, 'lahin')` এরও রিটার্ন ভ্যালু 5, index 5 এ এখন ২ টি রেফারেন্স থাকবে। 

<p align="center">
  <img src="./images/bf-3.png" alt="bloom filter">
</p>