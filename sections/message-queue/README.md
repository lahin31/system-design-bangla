## Kafka

Kafka খুবই পপুলার Message Queue। High-performance data pipelines, streaming analytics, data integration, এবং mission-critical applications গুলোর জন্য Kafka ব্যবহার করা হয়। 

## Components of Kafka

- Producer: Producer কিংবা Publisher যার কাজ হচ্ছে message pass করা। এখানে message হতে পারে API রিকোয়েস্ট।

- Broker: Producer message টি Broker এ pass করবে। Broker কে Kafka Server বলা হয়।

<p align="center">
  <img src="./images/broker.png" alt="Publisher and Broker">
</p>

- Topic: Topic মূলত Broker এর একটি Core Section, যার কাজ হল মেসেজগুলোকে Organize করা। Broker এর ভিতর এক বা একাধিক Topic থাকবে।

<p align="center">
  <img src="./images/topic.png" alt="Topic">
</p>

(চলমান)