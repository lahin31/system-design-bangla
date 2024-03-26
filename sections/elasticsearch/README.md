## Why Elasticserach?

আমরা একটি scenario চিন্তা করি, আমাদের একটি e-commerce application আছে। যেখানে ৮০ লক্ষ প্রোডাক্ট আছে। এখন end user কোনো একটি প্রোডাক্ট এর keyword দিয়ে যখন সার্চ করবে তখন response time 400ms বা তার থেকেও বেশি হতে পারে।

আরেকটি scenario চিন্তা করি, আপনার একটি ব্লগ আছে, যেখানে end user এসে Full Text Search করছে মানে একটি sentence দিয়ে blog সার্চ করবে।

এরকম আরো অনেক scenario আছে। এসবের জন্য Elasticserach খুবই প্রয়োজনীয়।

## Structure of Elasticsearch

Elasticsearch এ কালামগুলোকে সাধারণত Field এবং Row গুলোকে Document বলে। এখানে মানে Elasticsearch এ টেবিলগুলোকে index বলা হয়।

## Elasticsearch কিভাবে GET রিকোয়েস্ট প্রসেস করে থাকে?

প্রথমে ক্লায়েন্ট GET রিকোয়েস্ট করবে সার্ভারকে, ব্যাকএন্ড সার্ভার নিজ থেকে Elasticsearch Instance এর মধ্যে GET রিকোয়েস্ট করবে। Elasticsearch server এর কাজ হল সেই query এর ইনডেক্স ভ্যালু রিটার্ন করে দেয়া, তা array আকারে হতে পারে। তারপর সেই ইনডেক্সগুলো দিয়ে ব্যাকএন্ড সার্ভার তখন মূল ডাটাবেস MySQL কিংবা Postgresql এ হিট করবে। অতঃপর MySQL সেই ইনডেক্স ভ্যালু দিয়ে সার্চ করে ডাটাগুলোকে ব্যাকএন্ড সার্ভারে পাঠিয়ে দিবে।

<p align="center">
  <img src="./images/elasticsearch-get.png" alt="Elasticsearch">
</p>

## Mapping

Elasticsearch এর Mapping হল index এর মধ্যে কিভাবে Field কিভাবে স্টোর হবে তা নির্ধারণ করে থাকে। Field এর ডাটা টাইপ কেমন হবে তা বলে দেয়।

```
{
  "mappings": {
    "properties": {
      "name": { type: "text" },
      "email": { type: "text" },
      "username": { type: "text" },
      "age": { type: "text", index: false }
    }
  }
}
```

Elasticsearch এ সাধারণত সব field এ ইনডেক্স করা থাকে, আমরা চাইলে { index: false } দিতে পারব যদি নির্দিষ্ট field এর জন্য index প্রয়োজন না হয়।

## DSL Query

Domain Specific Language সংক্ষেপে DSL যা একটি JSON based query language। তা দ্বারা আমরা Elasticsearch এ query operation চালাতে পারি। উদাহরণ,

```
{
  "query": {
    "match": {
      "field_name": "search_term"
    }
  }
}
```

এখানে,

- query: query configuration শুরু হবে query অবজেক্টের ভিতর দিয়ে।
- match: কোন রকমের query থাকবে Full Text Search এর জন্য।
- field_name: কোন ফিল্ড search হবে এখানে বলে দিতে হয়। যদি আমরা description নামের field search করি তাহলে description দিতে হবে।
- search_term: যে টার্ম আমরা search করব তা এখানে বলে দিব।
