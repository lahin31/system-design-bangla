## Kafka

Kafka খুবই পপুলার Message Queue। High-performance data pipelines, streaming analytics, data integration, এবং mission-critical applications গুলোর জন্য Kafka ব্যবহার করা হয়। 

## Components of Kafka

- Producer: Producer কিংবা Publisher যার কাজ হচ্ছে message pass করা। এখানে message হতে পারে API রিকোয়েস্ট।

- Cluster: Producer message টি Cluster এ pass করবে। Cluster এর ভিতর Topics, Broker এবং ZooKeeper থাকবে।

<p align="center">
  <img src="./images/cluster.png" alt="Cluster">
</p>

- Broker: Cluster এর ভিতর Broker থাকে। এটি একটি সার্ভার যার কাজ হল message/data গ্রহণ এবং প্রসেস এর পর send করা। 

<p align="center">
  <img src="./images/broker.png" alt="Broker">
</p>

- Topic: Topic মূলত Broker এর একটি Core Section, যার কাজ হল মেসেজগুলোকে Organize করা। Broker এর ভিতর এক বা একাধিক Topic থাকবে।

<p align="center">
  <img src="./images/topic.png" alt="Topic">
</p>

Topic এর ভিতর Data Partitioning হয়ে থাকে। প্রতিটি Partition এ মূলত Set of Messages বিদ্যমান থাকে। প্রতিটি message identifie করা হয় partition এর offset দ্বারা। Messages গুলো partition এর শেষে যুক্ত হয় এবং অন্য দিক থেকে send করা হয়।

<p align="center">
  <img src="./images/partition.png" alt="Partition">
</p>

(চলমান)