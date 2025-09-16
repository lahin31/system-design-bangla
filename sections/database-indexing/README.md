## ডাটাবেস B+ ট্রি কেনো ব্যবহার করে?

MySQL, PostgreSQL, MongoDB মূলত B+ ট্রি ব্যবহার করে তার কিছু কারণ রয়েছে,

- প্রতিটি নোড একাধিক key-pointer সংরক্ষণ করতে পারে। এমনকি কোটি কোটি row এর জন্যও, একটি lookup সাধারণত মাত্র ২-৪টি page এক্সেস করলে expected data পেতে পারি।

- Leaf Node গুলো sequentially সংযুক্ত থাকে, যা রেঞ্জ কোয়েরিকে (যেমন, WHERE age BETWEEN 20 AND 30) খুবই efficient করে তুলে।

### B+ ট্রি দেখতে কেমন হয়?

(চলমান)
