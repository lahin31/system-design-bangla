# Concurrency and Parallelism handling

একটি প্রোডাকশন environment এ User Experience ভালো রাখার জন্য সঠিকভাবে Concurrency/Parallelism ব্যবহার করতে হয়। 

## Worker Thread

Worker Thread হল Node.js-এর এমন একটি ফিচার যা আপনাকে Heavy CPU-bound কাজগুলো Main Thread থেকে আলাদা thread-এ চালাতে দেয়।

Heavy CPU-bound কাজগুলো হতে পারে,

- Image প্রসেসিং
- Video প্রসেসিং
- PDF প্রসেসিং

এই কনসেপ্ট প্রায় সব Programming Language-এ আছে।

Worker Thread ব্যবহার করার সুবিধা - Main Thread free থাকে, যার ফলে API response ফাস্ট হয়।

একটি ব্যপার আমাদেরকে বুজতে হবে, Queue (যেমন RabbitMQ) আর Worker Thread এক জিনিস না।

Worker Thread ব্যবহার করা হয় CPU-bound কাজ parallel ভাবে চালানোর জন্য, যাতে Main Thread block না হয়।

Queue ব্যবহার করা হয় job decouple(scalability), distribute(scalability) এবং reliable করার জন্য — যাতে system crash হলেও কাজ হারিয়ে না যায় এবং multiple worker parallel কাজ করতে পারে।

### একাধিক প্রসেস কিংবা থ্রেড একই পোর্ট বিন্ড করার চেষ্টা করে থাকলে কি হয়?

আমরা অনেক সময় দেখতে পাই, Error: listen EADDRINUSE: address already in use 0.0.0.0:3000

**কারণ #১: অন্য কোনো প্রসেস ইতিমধ্যেই সেই পোর্টে চলমান**

যদি একই পোর্টে অন্য কোনো প্রোসেস কাজ করছে, তাহলে নতুন সার্ভার সেই পোর্টে bind করতে পারবে না। এই প্রসেসটি খুঁজে বের করে terminate করতে হবে।

**কারণ #২: আগের প্রসেস সম্প্রতি বন্ধ হয়েছে, কিন্তু পুরানো socket এখনও TIME_WAIT স্টেটে আছে**

TIME_WAIT হলো TCP connection এর একটি স্টেট, যা তখন ঘটে যখন connection সক্রিয়ভাবে (actively) বন্ধ করে। যখন connection বন্ধ হয়, operating system তৎক্ষণাৎ পোর্টটি ফ্রি করে না। বরং কিছু সময়ের জন্য (প্রায় এক মিনিট বা কম) সেই পোর্টকে reserved রাখে।

এর মূল উদ্দেশ্য হলো: বন্ধ হওয়া connection থেকে যদি কোনো delayed বা late packet আসে, তা নতুন connection এর সঙ্গে মিশে না যায়। এইভাবে network reliability বজায় থাকে এবং পুরনো ডেটা নতুন connection কে কনফিউজ করতে পারে না।

সাধারণত, যদি একই পোর্টে আবার সার্ভার চালাতে চাও, এবং পুরনো connection এখনও TIME_WAIT-এ থাকে, তখন “Address already in use” এরর দেখা দিতে পারে।
