# গুরুত্বপূর্ণ পয়েন্টস

## ১. OAuth2 Authentication Protocol নয়

আমরা সবাই কোনো না কোনো সময় ভুল করে ভেবেছি — "Google Login করালাম মানেই OAuth2 দিয়ে User-কে Authenticate করলাম।" কিন্তু আসল ঘটনা একটু আলাদা।

OAuth2 বানানোই হয়েছিল একদম ভিন্ন একটা সমস্যা সমাধানের জন্য — Login করানোর জন্য না, বরং Authorization-এর জন্য। মানে প্রশ্নটা হলো:

> কোনো Application-কে কি User-এর Resource Access করার Permission দেওয়া হবে?


উদাহরণ:

```text
Google Drive
    │
    ▼
আমার Application
    │
    ▼
User অনুমতি দিল
    │
    ▼
Application এখন Drive Access করতে পারবে
```

এখানে Application পেয়ে গেল Drive-এ ঢোকার Permission। কিন্তু একটা জিনিস খেয়াল করুন — এই পুরো প্রসেসে কোথাও কিন্তু এটা কনফার্ম হলো না যে, আসলে কে এই Permission দিল। User-টা কে, সেটা OAuth2 কখনোই বলে না — এটা ওর কাজের পরিধির মধ্যেই পড়ে না।

এই "User কে?" প্রশ্নের উত্তর দেয় OpenID Connect (OIDC) — যেটা OAuth2-এর উপরেই বসানো একটা আলাদা লেয়ার।

