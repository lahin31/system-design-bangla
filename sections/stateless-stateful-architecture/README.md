## Stateful Architecture এর Limitation

আমরা জানি Stateful Archtecture ডেটা সার্ভার স্টোর করে রাখে। এখন যদি request এর পরিমাণ বেশি হয় তখন কোনো এক কারণে সার্ভারে কোনো ত্রুটি হওয়ার ফলে ডেটা নষ্ট হয়ে যেতে পারে, তখন ডেটা recover করার সুযোগ থাকে না।

<p align="center">
  <img src="./images/stateful-1.png" alt="Stateful pic">
</p>

ডেটা নষ্ট হওয়ার আগে।

<p align="center">
  <img src="./images/stateful-2.png" alt="Stateful pic">
</p>

অতিরিক্ত Client Request এর ফলে Client 1 এর Data নষ্ট হওয়ার পর, সার্ভারে Client 1 এর ডেটা আর নাই।

## Scale Stateful and Stateless Architecture

উপরের যে issue নিয়ে বলা হলো তা Stateful Architecture এ হয়ে থাকে। একই issue Stateless Architecture যেরকম কাজ করে,

<p align="center">
  <img src="./images/stateless-1.png" alt="Stateless pic">
</p>

একাধিক user সার্ভারে call দেয়া শুরু করলো, আমাদের সার্ভারের রিসোর্স কম সেজন্য আমাদের সার্ভার একটি সময় ক্র্যাশ করবে। এই সমস্যা সমাধানের জন্য আমরা Load Balancer দিয়ে Horizontal Scaling ব্যবহার করতে পারি।

<p align="center">
  <img src="./images/stateless-2.png" alt="Stateless pic">
</p>

যেহেতু Stateless Architecture এ সার্ভারের মধ্যে কোনো প্রকারের ডেটা কিংবা ইনফরমেশন সংরক্ষন হচ্ছে না, সেহেতু ক্লায়েন্ট যেকোন সার্ভার থেকে ডেটা পেলেই হবে। এরকম আমরা Stateless Architecture এ খুব সহজে Scale করতে পারি।

(চলমান)