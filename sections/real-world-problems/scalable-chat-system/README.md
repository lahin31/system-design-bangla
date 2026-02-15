# Design a Scalable One to One and One to Many chat system

## Key requirements

**Concurrency Handling**

- একই সময়ে হাজার হাজার ইউজার message পাঠাবে (1-to-1 এবং 1-to-many)

- একই conversation-এ একসাথে multiple message আসলে ordering নষ্ট হওয়া যাবে না।

- message duplication বা lost message হওয়া যাবে না।

**Atomic Message Processing**

- একটি message send হলে: DB-তে save হবে এবং recipient-এ deliver হবে।

**1-to-1 Chat Requirements**

- message send/receive

- message history

- offline message support

- delivery confirmation

- low latency

**1-to-many Chat Requirements**

- একজন message দিলে group members receive করবে।

- large group হলেও system overload হওয়া যাবে না।

- DB write explosion avoid করতে হবে।

**Scalability requirements**

- System scale: Thousands of users, High Message Rate এবং Worker increase করে processing scale করতে পারে।

(চলমান)