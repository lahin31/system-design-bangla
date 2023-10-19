## Consistency

ডেটাবেসে যখন কোনো ট্রান্সেকশন হবে মানে কোনো প্রকারের আপডেট অপারেশন ঘটলে, সব নোডে/সার্ভার এ একই বা constant ভ্যালু থাকবে।

<p align="center">
  <img src="./images/consistency.png" alt="consistency">
</p>

(CAP Theorem এবং ACID এর C(consistency) কিন্তু এক না। CAP Theorem এর consistency মানে হল Distributed System এর সব নোডে একই সময় একই(consistent) ভ্যালু থাকবে। ACID এর consistency মানে হল প্রতিটি Transection একটি Database state কে, একটি constant state থেকে আরেকটি constant state এ রূপান্তর করবে।)

(চলমান)
