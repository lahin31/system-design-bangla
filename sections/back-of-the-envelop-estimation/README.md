## Find Back of the Envelop Estimation

Back of the Envelop Estimation বের করতে আমাদের একটি Cheat Sheet মনে রাখতে হবে,

- 3 zeros = thousand traffic (KB)
- 6 zeros = million traffic (MB)
- 9 zeros = billion traffic (GB)
- 12 zeros = trillion traffic (TB)

x Million users \* y KB = xy GB

উপরের ফর্মুলা ব্যবহার করে আমরা কিছু Estimations বের করতে পারি,

**আমরা এখন ফেইসবুক এর Storage Capacity Estimation বের করতে পারি।**

ধরি ফেসবুক এর DAU(Daily Active User) ১০ Billion।

Daily Active User যদি গড় ১ টি করে post করে তাহলে, ১ টি post ধরে নি 1kb

(10 Billion \* 1 kb) = 10 trillion

উপরের চার্ট অনুযায়ী billion এর ৯ টি zeros আর kb এর ৩ টি zeros মোট ৯ + ৩ = ১২, ১২ টি zeros মানে trillion আর ১০ \* ১ মানে ১০। storage এর পরিমান দাঁড়ায় ১০ trillion।

**আমরা এখন ফেইসবুক এর Traffic Estimation বের করতে পারি।**

প্রতিটি user গড় ৪ টি করে read অপারেশন এবং ২ টি করে write অপারেশন করছে।

(10 Billion \* 6 queries) / 86400 = ~700000

প্রতি সেকেন্ডে ফেইসবুককে গড় ৭০০০০০ রিকোয়েস্ট প্রসেস করতে হচ্ছে।
