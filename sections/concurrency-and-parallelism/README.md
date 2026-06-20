# Concurrency and Parallelism handling

একটি প্রোডাকশন environment এ User Experience ভালো রাখার জন্য সঠিকভাবে Concurrency/Parallelism ব্যবহার করতে হয়। 

## Event Loop

Node.js single-threaded। মানে by default এটার একটাই Main Thread আছে যেখানে আপনার সব JavaScript code execute হয়।

Event Loop হলো সেই মেকানিজম, যা এই Main Thread-কে block না করে একসাথে অনেক I/O operation (যেমন database query, file read, API call) handle করতে দেয়। যখন একটা I/O operation শুরু হয়, Node.js সেটা background-এ পাঠিয়ে দেয় এবং Main Thread immediately পরের কাজে চলে যায়। I/O শেষ হলে তার callback Event Loop-এর মাধ্যমে আবার Main Thread-এ এসে execute হয়।

কিন্তু এই সিস্টেমের একটা বড় সমস্যা আছে - যদি কোনো কাজ CPU-bound হয় (যেমন বড় একটা loop চালানো, image process করা), তাহলে সেটা Main Thread-কেই আটকে রাখে, কারণ এটা কোনো I/O operation না যে background-এ পাঠানো যাবে। এই সময়ে Event Loop পুরো freeze হয়ে থাকে, এবং নতুন কোনো request-ই process হতে পারে না।

**ঠিক এই সমস্যার সমাধানের জন্যই Worker Thread লাগে।**

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

## Cluster Module

Worker Thread দিয়ে একটা নির্দিষ্ট heavy task আলাদা thread-এ চালানো যায়, কিন্তু পুরো অ্যাপ্লিকেশনটাই তো এখনও একটা মাত্র CPU core-এ চলছে। সার্ভারে ৮টা core থাকলেও, by default Node.js মাত্র ১টা core ব্যবহার করে, বাকি core গুলো idle থেকে যায়।

এই সমস্যার সমাধান Cluster Module। এটা দিয়ে একই অ্যাপ্লিকেশনের একাধিক copy (worker process) চালানো যায়, যেখানে প্রতিটা copy আলাদা CPU core-এ চলে, কিন্তু সবাই একই port-এ incoming request handle করতে পারে।

## Worker Thread আর Cluster এর পার্থক্যটা স্পষ্ট থাকা জরুরি

- Worker Thread: একটা নির্দিষ্ট heavy কাজ (যেমন একটা image process করা) Main Thread থেকে আলাদা করার জন্য।

- Cluster: পুরো অ্যাপ্লিকেশনকে multiple process-এ চালিয়ে সার্ভারের সবগুলো CPU core ব্যবহার করার জন্য, যাতে বেশি concurrent request handle করা যায়।

মানে, একটা সার্ভারে Cluster module দিয়ে ৪টা process চালালে, প্রতিটা process আবার আলাদাভাবে Worker Thread ব্যবহার করতে পারে নিজের heavy কাজের জন্য — এই দুটো একসাথেই কাজ করতে পারে, একটা আরেকটার বদলি না।

### একাধিক প্রসেস কিংবা থ্রেড একই পোর্ট বিন্ড করার চেষ্টা করে থাকলে কি হয়?

আমরা অনেক সময় দেখতে পাই, Error: listen EADDRINUSE: address already in use 0.0.0.0:3000

**কারণ #১: অন্য কোনো প্রসেস ইতিমধ্যেই সেই পোর্টে চলমান**

যদি একই পোর্টে অন্য কোনো প্রোসেস কাজ করছে, তাহলে নতুন সার্ভার সেই পোর্টে bind করতে পারবে না। এই প্রসেসটি খুঁজে বের করে terminate করতে হবে।

**কারণ #২: আগের প্রসেস সম্প্রতি বন্ধ হয়েছে, কিন্তু পুরানো socket এখনও TIME_WAIT স্টেটে আছে**

TIME_WAIT হলো TCP connection এর একটি স্টেট, যা তখন ঘটে যখন connection সক্রিয়ভাবে (actively) বন্ধ করে। যখন connection বন্ধ হয়, operating system তৎক্ষণাৎ পোর্টটি ফ্রি করে না। বরং কিছু সময়ের জন্য (প্রায় এক মিনিট বা কম) সেই পোর্টকে reserved রাখে।

এর মূল উদ্দেশ্য হলো: বন্ধ হওয়া connection থেকে যদি কোনো delayed বা late packet আসে, তা নতুন connection এর সঙ্গে মিশে না যায়। এইভাবে network reliability বজায় থাকে এবং পুরনো ডেটা নতুন connection কে কনফিউজ করতে পারে না।

সাধারণত, যদি একই পোর্টে আবার সার্ভার চালাতে চাও, এবং পুরনো connection এখনও TIME_WAIT-এ থাকে, তখন “Address already in use” এরর দেখা দিতে পারে।
