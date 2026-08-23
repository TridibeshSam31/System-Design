# Rate Limiting — Leaky Bucket, Comparison & Distributed Design — Poori Samajh (Enhanced Notes)

> Ye notes tere original Rate Limiting notes (Leaky Bucket se lekar poore distributed Redis-based design tak) ka enhanced version hain — kuch bhi remove nahi kiya (casual conversation bhi wahi rakhi hai), bas har technical point ke saath "iska matlab kya hai", "kyu aisa hota hai" wale explanations add kiye hain.

---

## 0. Sabse pehle basic terminology clear kar

**Rate Limiting kya hoti hai, ek line mein:**

Rate limiting ek mechanism hai jo control karta hai ki ek user/client kitni requests ek fixed time-period mein bhej sakta hai — taaki koi ek user (ya bot/attacker) system ko overwhelm na kar sake.

**Burst kya hota hai?**

"Burst" ka matlab hai — thodi der ke liye bahut zyada requests ek saath aa jaana (jaise normal traffic 10 req/sec ho, lekin achanak 100 requests ek hi second mein aa jaayein). Ye poore Rate Limiting topic ka ek core consideration hai — different algorithms bursts ko differently handle karte hain, jaisa tu aage dekhega.

Ye poore is note ki foundation hai — chalo ab actual content dekhte hain.

---

# Leaky Bucket

Token Bucket mein idea tha:

> Tokens refill hote hain, aur request token consume karti hai.

Leaky Bucket ka idea:

> Requests ek queue/bucket mein collect hoti hain, aur fixed rate se bahar process hoti hain.

**Ye fundamental difference pehle hi lock kar:** Token Bucket ka focus hai — "kya request ko allow karne ke liye ek token available hai?" (matlab agar tokens hain, to request turant process ho sakti hai, chahe kitni bhi ek saath aayein). Leaky Bucket ka focus hai — "requests ko ek queue mein daal do, aur unhe ek **fixed, controlled rate** se bahar nikaalo" — chahe kitni bhi requests ek saath aayein, output rate hamesha same rahega. Ye ek different philosophy hai — Token Bucket "flexible burst allow karta hai," Leaky Bucket "output ko hamesha smooth rakhta hai."

Imagine ek bucket jisme neeche ek chhota hole hai:

```
        Requests
       ↓  ↓  ↓  ↓
    ┌─────────────┐
    │ R R R R R   │
    │ R R R       │
    └──────┬──────┘
           ↓
       fixed rate
           ↓
        Backend
```

**Ye analogy deeply samajh:** Socho ek bucket hai jisme paani (requests) upar se daala ja raha hai — chahe kitna bhi tez daalo, bucket ke neeche ek chhota, fixed-size hole hai jisse paani hamesha ek **constant, predictable rate** se bahar nikalta hai. Chahe upar se dhaar (torrent) aaye ya boond-boond, neeche se nikalne wala rate hamesha same rehta hai (jab tak bucket overflow na ho jaaye).

Chahe ek saath 100 requests aa jayein, backend ko woh controlled rate par milengi.

## Example

Suppose:

```
Bucket capacity = 10 requests
Processing rate = 2 requests/sec
```

Suddenly:

```
10 requests
   ↓
┌──────────┐
│ R R R R R│
│ R R R R R│
└────┬─────┘
     ↓
  2 req/sec
```

Backend ko:

```
Second 1 → 2 requests
Second 2 → 2 requests
Second 3 → 2 requests
Second 4 → 2 requests
Second 5 → 2 requests
```

milengi.

**Is timeline ko carefully samajh:** Chahe saari 10 requests ek hi instant mein aayi thi (bursty input), lekin backend ko wo requests spread out milengi — 2-2 karke, 5 seconds ke duraan. Ye backend ko ek predictable, manageable load deta hai, chahe client-side traffic kitna bhi spiky ho.

So incoming traffic bursty ho sakta hai, but outgoing traffic smooth ho jata hai.

**Ye poore Leaky Bucket ka core value proposition hai:** Ye ek "traffic shaping" mechanism hai — input pattern chahe kaisa bhi ho (bursty, irregular), output pattern hamesha smooth aur predictable rahega. Ye particularly useful hai jab downstream system (jaise backend, database, external API) ek consistent, predictable load prefer karta ho, sudden spikes handle nahi kar sakta ho.

## Agar bucket full ho gaya?

Suppose capacity:

```
10 requests
```

Already:

```
R R R R R R R R R R
```

Bucket full hai.

Ab new request:

```
R11
 ↓
Bucket full
 ↓
❌ Reject
```

Depending on implementation, request ko reject/drop kiya ja sakta hai.

**Ye overflow scenario kyu hota hai:** Bucket ki capacity **finite** hai — matlab ek limited number of requests hi queue mein wait kar sakti hain. Agar incoming rate, outgoing (processing) rate se bahut zyada ho, aur ye lambi der tak chale, to bucket eventually full ho jaayega — us point ke baad, naya koi bhi request ke liye jagah nahi bachegi, isliye use reject karna padta hai. Ye ek necessary trade-off hai — infinite queue rakhna practically possible nahi hai (memory constraints), isliye ek limit set karni padti hai.

---

## Token Bucket vs Leaky Bucket

Ye sabse important difference hai.

### Token Bucket

```
Requests
   ↓
Need token?
   ↓
YES → immediately process
NO  → reject
```

Isliye burst allow kar sakta hai.

Example:

```
10 tokens available
↓
10 requests almost immediately
```

**Token Bucket ka behavior deeply samajh:** Agar bucket mein tokens available hain (jaise 10), to saari 10 requests **turant, ek hi saath** process ho sakti hain — koi artificial delay nahi. Ye "burst-friendly" hai — agar user ne recently kam requests bheji thi (isliye tokens accumulate ho gaye the), to use ek chhota burst allowed hai jab wo achanak zyada requests bheje.

### Leaky Bucket

```
Requests
   ↓
Queue
   ↓
Fixed processing rate
   ↓
Backend
```

Chahe 10 requests ek saath aayi:

```
10 requests
   ↓
Queue
   ↓
2/sec
```

So Leaky Bucket traffic ko smooth karta hai.

**Ye comparison ko ek line mein crystallize kar:** Token Bucket poochta hai "kya turant allow kar sakta hoon?" — aur agar haan, to koi delay nahi. Leaky Bucket poochta hai "kaise ise hamesha ek fixed rate pe process karoon?" — chahe requests turant aayi hon, unhe queue mein wait karna padega taaki output rate consistent rahe. Yahi fundamental behavioral difference hai jo dono algorithms ko alag banata hai — Token Bucket **allow-immediately-if-possible**, Leaky Bucket **always-smooth-output**.

---

## Simple real-life example

### Token Bucket

Imagine tere paas 10 coupons hain.

```
10 coupons
↓
10 people can enter immediately
```

Then coupons refill hote hain.

**Ye analogy kya represent karti hai:** Jaise ek event mein 10 "fast-pass" coupons hain — agar tere paas coupon hai, to tu turant andar ja sakta hai, koi wait nahi. Agar 10 log ek saath aayein aur sabke paas coupon ho, sab turant andar ja sakte hain (burst allowed). Coupons dobara "refill" hote rehte hain over time (jaise "har minute 2 naye coupons milenge").

### Leaky Bucket

Imagine cinema mein ek single entry gate hai:

```
100 people waiting
      ↓
Gate
      ↓
2 people/sec
```

People queue mein hain, but entry controlled rate par hai.

**Ye analogy bhi ek different scenario dikhati hai:** Chahe 100 log gate ke bahar wait kar rahe hon (bursty demand), gate khud ek fixed rate (2 log/sec) se hi andar bhej sakta hai — chahe demand kitni bhi ho, entry rate hamesha same rahega. Ye Leaky Bucket ke "hamesha smooth output" wale nature ko perfectly capture karta hai.

---

## Leaky Bucket ki limitation

Agar requests bahut zyada aa gayi:

```
1000 requests
```

aur:

```
Bucket capacity = 100
Processing = 2/sec
```

toh:

```
100 → queue
900 → reject
```

Aur queue mein jo request hai usko wait karna padega.

So latency increase ho sakti hai.

**Ye trade-off deeply samajh:** Leaky Bucket ke saath do risks hain — (1) agar bucket capacity se zyada requests aayein, extra requests reject ho jaayengi (jaisa upar dekha), aur (2) jo requests queue mein successfully enter ho gayi hain, unhe bhi apni turn ka wait karna padega — jaise agar 100 requests queue mein hain aur processing rate 2/sec hai, to last wali request ko 50 seconds tak wait karna pad sakta hai! Ye ek genuine latency cost hai jo Token Bucket mein nahi hota (jahan agar token available hai, turant process ho jaata hai, koi queueing delay nahi).

## Kab useful hai?

Jab tu chahta hai ki downstream service ko smooth, predictable traffic mile.

Example:

```
1000 requests suddenly
        ↓
   Leaky Bucket
        ↓
   controlled flow
        ↓
     Backend
```

Particularly useful when downstream system burst handle nahi kar sakta.

**Ye use-case ko concretely samajh:** Agar tera backend/database/external-API ek fixed, limited capacity handle kar sakta hai (jaise "database sirf 2 writes/sec efficiently handle kar sakta hai"), to Leaky Bucket ensure karta hai ki chahe frontend/clients kitni bhi bursty traffic bhejein, downstream system ko hamesha ek manageable, consistent rate hi milega. Ye particularly critical hota hai legacy systems, third-party APIs (jinki apni rate limits hoti hain), ya resource-constrained services ke saath integrate karte waqt.

---

## Ab 4 algorithms ek line mein

```
Fixed Window
→ fixed time window mein count

Sliding Window
→ last N seconds ki requests count

Token Bucket
→ tokens available hain toh request immediately allow

Leaky Bucket
→ requests queue mein jaati hain aur fixed rate se process hoti hain
```

## Most important distinction:

```
Token Bucket
→ allows controlled BURSTS

Leaky Bucket
→ SMOOTHS the traffic
```

**Is one-liner ko interview mein hamesha ready rakh:** Ye sabse concise, powerful way hai in dono ko differentiate karne ka — Token Bucket "flexibility with bursts" priority karta hai, Leaky Bucket "consistency/smoothness" priority karta hai. Interview mein jab bhi ye do algorithms compare karne ko bola jaaye, ye ek line seedha bol dena strong impression deta hai.

Bas ab Rate Limiting algorithms complete.

Next hum in chaaron ka proper comparison + interview mein "which one would you choose?" karenge. Uske baad distributed Redis-based Rate Limiter design karenge.

---

# Rate Limiting — Part 5: Which algorithm when?

Ab tere paas 4 algorithms hain:

```
1. Fixed Window
2. Sliding Window
3. Token Bucket
4. Leaky Bucket
```

Ab actual system-design mein interviewer algorithm ka definition nahi poochega. Woh bolega:

> "Design a rate limiter. Which approach will you use and why?"

Toh ye samajh.

**Ye framing shift bahut important hai, isko lock kar:** Beginner-level preparation mein log sirf "define each algorithm" pe focus karte hain. Lekin real interviews mein ye rarely hota hai — interviewer ek open-ended problem dega ("design a rate limiter"), aur tujhe khud decide karna hoga kaunsa algorithm fit baithta hai, aur **kyu**. Isliye ab jo section aa raha hai (har algorithm ka "use when"), wahi asli interview-ready knowledge hai.

---

## 1. Fixed Window

Rule:

```
100 requests / minute
```

System:

```
12:00 ───────── 12:01
       100 max
```

**Ye kaise kaam karta hai, basic mechanism:** Time ko fixed-size windows mein baant diya jaata hai (jaise har minute ek naya window) — us window ke andar, ek counter track karta hai kitni requests aa chuki hain. Jab tak counter limit (100) se kam hai, requests allow hoti hain. Jaise hi naya window shuru hota hai (12:01), counter reset ho jaata hai.

### Good

```
Very simple
Low memory
Easy to implement
```

**Ye fayde kyu hain:** Implementation sirf ek counter aur ek timestamp maintain karta hai per user/window — bahut kam memory chahiye, aur logic bhi straightforward hai (increment counter, check against limit, reset on new window).

### Bad

Boundary burst:

```
12:00:59 → 100 requests
12:01:00 → 100 requests
```

Potentially 200 requests almost instantly.

**Ye "boundary burst" problem deeply samajh, ye Fixed Window ki sabse badi weakness hai:** Suppose user 12:00:59 (window ke bilkul end mein) 100 requests bhej deta hai — sab allow ho jaayengi (kyunki us window ke andar limit exactly 100 hai). Fir agla second aate hi (12:01:00), naya window shuru ho jaata hai, counter reset ho jaata hai — ab user turant phir se 100 requests bhej sakta hai! Effectively, sirf 2 seconds ke andar (12:00:59 se 12:01:00), user ne 200 requests bhej di, jabki intended limit "100 per minute" tha. Ye ek genuine design flaw hai jo attacker exploit kar sakta hai.

### Use when

Accuracy extremely strict nahi hai aur simplicity important hai.

**Ye recommendation ka reasoning:** Agar tera use-case rate limiting ko "approximately" enforce karna hai (jaise general API abuse prevention, strict precision critical nahi), aur tujhe simple, low-overhead solution chahiye, to Fixed Window kaafi hai. Lekin agar precision critical hai (jaise financial/security-sensitive endpoint), to boundary-burst problem ek real risk hai.

---

## 2. Sliding Window Log

Question:

> "Pichhle 60 seconds mein kitni requests aayi?"

Example:

timestamps:

```
12:00:10
12:00:20
12:00:31
12:00:50
```

Every request par old timestamps remove karo.

**Ye Fixed Window se fundamentally kaise different hai:** Fixed Window ek "static" time-boundary use karta hai (jaise 12:00 se 12:01 tak). Sliding Window Log ek "moving" window use karta hai — har request ke time, system dekhta hai "pichhle 60 seconds" (jo har moment badalta rehta hai) mein kitni requests thi. Isliye ye boundary-burst problem ko naturally avoid karta hai — kyunki koi hard, fixed boundary hi nahi hai jise exploit kiya ja sake.

**"Har request par old timestamps remove karo" ka matlab:** Har naye request ke saath, system apne stored timestamps list mein se un timestamps ko hata deta hai jo current-time se 60 seconds se zyada purane ho chuke hain — isliye list hamesha sirf "last 60 seconds" ki requests ka accurate record rakhta hai.

### Good

```
Accurate
Fixed-window boundary problem avoid karta hai
```

**Ye accuracy kyu genuinely better hai:** Kyunki ye "true" rolling window use karta hai, ye kabhi bhi ek precise, correct answer dega ki "abhi is exact moment, pichle 60 seconds mein kitni requests thi" — koi artificial boundary-related exploit possible nahi.

### Bad

Har request ka timestamp store karna padta hai.

Millions of users + high traffic:

```
Memory ↑
Processing ↑
```

**Ye scalability problem deeply samajh:** Fixed Window mein sirf ek counter (ek number) store karna padta tha per user. Sliding Window Log mein, har individual request ka timestamp store karna padta hai — agar user ka limit "1000 requests/minute" ho, to worst case mein 1000 timestamps store ho sakti hain per user, per window. Millions of users ke saath, ye storage requirement bahut zyada ho sakta hai — aur har request pe "purane timestamps ko filter karna" bhi processing overhead add karta hai.

### Use when

Traffic relatively manageable hai aur accuracy important hai.

**Ye trade-off reasoning:** Sliding Window Log "correctness-first" approach hai — agar tera system size manageable hai (bahut extreme scale nahi), aur precision critical hai (jaise koi security-sensitive limit), to iske memory/processing overhead ko justify kiya ja sakta hai. Lekin bahut bade scale pe (millions of users, high-frequency requests), ye impractical ho sakta hai.

---

## 3. Token Bucket

Bucket:

```
Capacity = 100 tokens
Refill = 10 tokens/sec
```

Request:

```
1 token consume
```

No token:

```
reject
```

### Important property

Burst allow karta hai.

Agar bucket mein 100 tokens hain:

```
100 requests
```

almost immediately process ho sakti hain.

Then refill gradually hota hai.

**Ye "gradual refill" mechanism deeply samajh:** Bucket mein tokens time ke saath continuously add hote rehte hain (jaise 10 tokens per second), lekin ek maximum capacity (100) tak hi accumulate ho sakte hain. Agar user ne kuch der se requests nahi bheji thi, tokens accumulate ho jaate hain (up to capacity) — isliye jab wo achanak bahut saari requests bheje, use ek "saved up" burst allowance milta hai. Lekin agar wo continuously requests bhej raha ho, to sirf utni hi requests allow hongi jitne tokens refill ho rahe hain (average rate limited rahega long-term).

### Use when

Tu chahta hai:

> Normal traffic controlled ho, but legitimate short bursts allowed ho.

API rate limiting ke liye ye extremely useful model hai.

**Ye kyu "extremely useful" hai, real-world context mein:** Real applications mein, users ka traffic naturally bursty hota hai — jaise ek user browser mein multiple tabs khol de, ya ek app initial load pe ek saath multiple API calls kare (jaise dashboard load karte waqt 5-6 different data fetch ho rahe hon). Agar rate limiter is natural burst ko bhi strictly block kar de, to legitimate users ka experience kharab ho jaayega. Token Bucket is realistic pattern ko accommodate karta hai — short, occasional bursts allow karta hai, lekin sustained high-rate abuse ko still limit karta hai (kyunki refill rate cap hai).

---

## 4. Leaky Bucket

Requests:

```
        Queue
          ↓
      2 req/sec
          ↓
       Backend
```

Chahe incoming:

```
100 requests instantly
```

outgoing:

```
2/sec
```

### Good

Traffic smooth karta hai.

### Bad

Queue fill hone par:

```
queue full
   ↓
reject
```

Aur queued requests ko wait karna padega.

### Use when

Downstream service ko smooth/predictable traffic chahiye.

**Ye recap section-A (upar) ka summary hai — key point ye hai ki ab ye ek "recommendation framework" ka part hai, sirf definition nahi.**

---

## 5. Final comparison

| Algorithm | Main idea | Burst | Memory | Main problem |
|---|---|---|---|---|
| Fixed Window | Fixed time mein count | Possible | Low | Boundary burst |
| Sliding Window Log | Last N seconds count | Controlled | High | Timestamp storage |
| Token Bucket | Tokens consume/refill | Yes | Low | Distributed atomicity |
| Leaky Bucket | Queue → fixed output rate | No/controlled | Queue required | Queue latency/overflow |

**Is table ka interview mein use kaise karna hai:** Ye table dikhata hai ki har algorithm ka apna trade-off hai — koi bhi "universally best" nahi hai. Agar interviewer poochhe "which one would you use," tera answer specific requirements pe depend karna chahiye (jaisa agla section explain karega). Notice karo Token Bucket ka "main problem" hai "distributed atomicity" — ye ek preview hai us cheez ka jo poore notes ke doosre half mein deeply explore hoga (race conditions, Redis, atomic operations).

## Ekdum simple:

```
Fixed Window
→ "Is minute mein kitni requests?"

Sliding Window
→ "Last 60 sec mein kitni requests?"

Token Bucket
→ "Token hai? Request allow."

Leaky Bucket
→ "Queue mein daalo, fixed rate se process karo."
```

**Ye four one-liners interview mein rapid-recall ke liye perfect hain** — agar interviewer achanak "sabhi 4 algorithms ek line mein bata" bole, ye directly use kar sakta hai.

---

## 6. Ab actual System Design

Suppose interviewer bolta hai:

> Design a rate limiter for my API.

Architecture:

```
                 Clients
                    ↓
               Load Balancer
                    ↓
             Rate Limiter
                    ↓
                Backend
                    ↓
                Database
```

**Ye high-level architecture kya establish kar raha hai:** Rate Limiter ek dedicated "checkpoint" hai jo backend tak pahunchne se pehle har request ko evaluate karta hai — "kya ye request allowed hai, ya reject karni chahiye?" Sirf allowed requests hi actual backend processing tak pahunchti hain. Ye ek important design principle hai — rate limiting jitni jaldi (request lifecycle mein) ho, utna better, taaki reject hone wali requests unnecessary backend resources consume na karein.

But ek problem hai.

Suppose 3 backend servers:

```
             Load Balancer
            /      |      \
           ↓       ↓       ↓
         S1       S2       S3
```

User ki rate limit:

```
100 requests/minute
```

Agar har server apna counter rakhe:

```
S1 → 100
S2 → 100
S3 → 100
```

User theoretically:

```
300 requests/minute
```

kar sakta hai.

**Ye exact wahi problem hai jo tune GameManager notes mein bhi dekhi thi, connect kar:** Agar har server ka apna, independent, in-memory rate-limit counter hai (jaise pehle GameManager ka apna independent `games[]` array tha), to koi shared, consistent state nahi hai. Agar user ki requests load-balancer ke through 3 alag servers mein distribute ho rahi hain, aur har server independently "apne paas" count kar raha hai, to actual total request-count (jo user ne bheji) us "shared truth" se bahut zyada ho sakta hai jo intended limit tha. Ye distributed systems ka ek classic problem hai — **local state, distributed context mein sufficient nahi hai.**

So local memory mein rate-limit state rakhna wrong hai for a distributed rate limiter.

---

## 7. Shared Rate-Limit State

Use Redis:

```
             Load Balancer
            /      |      \
           ↓       ↓       ↓
         S1       S2       S3
           \       |       /
            \      |      /
                 Redis
```

Redis contains rate-limit state.

For example:

```
user:123
   ↓
tokens = 37
last_refill = ...
```

**Ye exact wahi Singleton-jaisi philosophy hai, bas distributed scale par:** Jaise GameManager Singleton se "poore application mein ek shared state" achieve hua tha (single process ke andar), Redis yahan "poore distributed system mein ek shared state" achieve karta hai (multiple processes/servers ke across). Concept same hai — "state ek jagah honi chahiye jo sab share karein, na ki har jagah apni alag copy ho" — bas scope alag hai (single-process vs multi-server).

Ab chahe request S1 pe aaye:

```
S1 → Redis
```

ya S3 pe:

```
S3 → Redis
```

same user's bucket check hoga.

**Ye kaise problem ko solve karta hai:** Ab chahe user ki request kisi bhi server (S1, S2, ya S3) pe route ho, wo server Redis mein jaake **same, shared** rate-limit data check/update karega. Isliye chahe requests kitne bhi alag servers pe spread hon, total count accurately track hota hai — no more "300 requests theoretically possible" wali problem.

---

## 8. But Redis alone isn't enough

Yahan ek important distributed-systems problem hai.

Suppose:

```
tokens = 1
```

Two requests simultaneously:

```
Request A → Server 1
Request B → Server 2
```

Both Redis se read karte hain:

```
A → tokens = 1
B → tokens = 1
```

Dono bol sakte hain:

```
ALLOW
```

But actual mein sirf 1 token tha.

So:

```
2 requests allowed
1 token available
```

Wrong.

This is a race condition.

**Ye exact wahi race condition hai jo tune Idempotency notes mein bhi dekhi thi — pattern recognize kar:** Ye phir se wahi "check-then-act" problem hai — dono servers pehle "check" karte hain (tokens = 1?), dono ko same answer milta hai (kyunki koi bhi abhi tak update nahi kar chuka), aur dono independently "allow" decide kar lete hain — jabki actual mein sirf 1 request allow honi chahiye thi. Sirf Redis mein data share karna kaafi nahi hai agar us data ko safely, atomically update na kiya jaaye.

---

## 9. We need atomicity

Hume logically ye poora operation ek atomic operation banana hai:

```
1. Current tokens calculate karo
2. Refill calculate karo
3. Check karo token available hai?
4. Token consume karo
5. New state save karo
```

Ye sab ek saath hona chahiye.

**"Ek saath" (atomic) ka matlab precisely samajh:** In 5 steps ke beech, koi bhi doosri request "beech mein aake" apna operation nahi kar sakti — jab tak ek request ye poori 5-step sequence complete na kar le, koi doosri request usi user ke data ko touch nahi kar sakti. Ye guarantee ensure karta hai ki agar 2 requests "simultaneously" aayein, wo actually **sequentially, ek ke baad ek** process hongi (chahe bahut tez ho) — isliye second request ko always accurate, updated state milega (jo pehli request ne already consume kar diya hai).

Redis mein is tarah ke atomic logic ke liye Lua script jaise mechanism use kiye ja sakte hain.

**Lua script kyu use hota hai, iska core reasoning samajh:** Redis apne aap mein single-threaded hai (matlab ek waqt mein sirf ek command process karta hai) — isliye agar tu poori "check + update" logic ko ek single Lua script ke andar bhej de (jo Redis directly execute karta hai server-side), to Redis guarantee deta hai ki wo poora script **atomically, bina interruption ke** run hoga — koi doosri request beech mein interfere nahi kar sakti, chahe wo simultaneously aayi ho. Ye Redis ki built-in atomicity guarantee ko use karta hai complex, multi-step logic ke liye bhi.

Conceptually:

```
Request
   ↓
Redis atomic operation
   ↓
┌──────────────────────┐
│ refill tokens        │
│ check token          │
│ consume token        │
│ update timestamp     │
└──────────────────────┘
   ↓
ALLOW / REJECT
```

Do simultaneous requests aaye toh Redis ke andar operation safely serialize/atomically execute ho sakta hai.

**"Serialize" word yahan important hai:** Chahe 2 requests "wall-clock time" mein ek hi millisecond mein aayi hon, Redis unhe internally **ek ke baad ek (serially)** process karta hai — pehli request poora apna atomic block complete karti hai, phir doosri shuru hoti hai. Isliye jab doosri request apna check karti hai, use pehli request ka updated state milta hai (jaise tokens already consumed ho chuke), aur wo correctly reject ho sakti hai agar tokens khatam ho gaye ho.

---

## 10. Token Bucket ka distributed flow

Suppose:

```
Capacity = 10
Refill = 2 tokens/sec
```

Redis:

```
user:123

tokens = 7
last_refill = T
```

Request:

```
POST /api
```

### Step 1

Redis state retrieve.

### Step 2

Time ke according naye tokens calculate:

```
elapsed time × refill rate
```

**Ye calculation kaise kaam karti hai:** Suppose last refill `T` time pe hua tha, aur ab current time `T + 3 seconds` hai — to elapsed time 3 seconds hai. Agar refill rate 2 tokens/sec hai, to naye tokens = `3 × 2 = 6` tokens add honge (theoretically, capacity ki limit tak).

### Step 3

Capacity se upar nahi jaane dena.

```
tokens = min(capacity, tokens + new_tokens)
```

**Ye `min()` function ka role samajh:** Chahe kitne bhi tokens theoretically accumulate ho jaayein (jaise agar user bahut der se koi request nahi bheji), bucket ki maximum capacity (10) se zyada tokens kabhi store nahi honge — `min()` ensure karta hai ki total kabhi capacity cross na kare.

### Step 4

Agar:

```
tokens >= 1
```

then:

```
tokens -= 1
ALLOW
```

Otherwise:

```
REJECT
```

**Poore flow ko end-to-end ek baar phir se dekh:** Har request ke liye, system pehle current tokens (existing + naya refill, capped at capacity) calculate karta hai, phir check karta hai kya kam se kam 1 token available hai — agar haan, ek token consume karke request allow kar deta hai; agar nahi, reject kar deta hai. Ye poora sequence (Section 9 mein discuss hua) ek atomic operation ke roop mein execute hona chahiye, taaki race conditions avoid ho sakein.

---

## 11. Response kya denge?

Rate limit exceed:

```
HTTP 429 Too Many Requests
```

Often useful headers bhi diye ja sakte hain, such as remaining quota or retry timing, depending on the API design.

**Ye extra headers kyu useful hote hain (jaise `X-RateLimit-Remaining`, `Retry-After`):** In headers se client (ya frontend developer) ko explicit information milti hai — "tere paas abhi kitne requests bache hain," aur "kitni der baad tu dobara try kar sakta hai." Ye client-side experience ko improve karta hai, kyunki client apna behavior adjust kar sakta hai (jaise automatic backoff/retry logic implement karna) bina bas trial-and-error karte rehne ke.

Conceptually:

```
HTTP/1.1 429 Too Many Requests
```

Client samajh jaata hai:

> "Meri request rate limit exceed ho gayi."

---

## 12. Rate limiter kis basis par limit kare?

Ye bhi interview mein important hai.

Not necessarily only:

```
IP
```

You can rate-limit based on:

```
User ID
API key
IP address
Endpoint
Tenant
Combination
```

**Ye ek common misconception ko clarify karta hai — "rate limiting = sirf IP-based" nahi hai:** Beginners aksar sochte hain rate limiting sirf IP address ke basis par hoti hai, lekin real systems mein bahut zyada granular/flexible criteria use hote hain. IP-based limiting mein ek problem bhi hai — multiple users same IP (jaise office network, ya NAT ke peeche) share kar sakte hain, jisse legitimate users galti se block ho sakte hain. Isliye authenticated systems mein aksar User ID ya API key zyada accurate/fair basis hoti hai.

Example:

```
user123 + /search
```

vs

```
user123 + /payment
```

Different limits ho sakte hain.

For example:

```
/search
→ 100 req/min

/payment
→ 10 req/min
```

Payment endpoint ko stricter limit chahiye ho sakti hai.

**Ye per-endpoint differentiation ka reasoning samajh:** Har endpoint ka "risk profile" aur "resource cost" alag hota hai — search jaisa endpoint relatively low-stakes hai (thodi zyada requests bhi manageable), lekin payment jaisa endpoint high-stakes hai (fraud risk, financial impact) — isliye usko zyada strict limit chahiye. Ye "combination" wala approach (user_id + endpoint) bahut common pattern hai production systems mein, kyunki ye fine-grained control deta hai.

---

## 13. Tere Movie Search API ka mapping

Suppose tera endpoint:

```
GET /search?query=batman
```

Without rate limiting:

```
User
 ↓
1000 requests/sec
 ↓
Backend
 ↓
External movie API / DB
```

Cost/load badh sakta hai.

**Ye tere specific project ke context mein real risk kya hai:** Agar tera Movie Search API kisi external movie database API (jaise TMDb, OMDb) ko call kar raha hai, to har uncontrolled request potentially ek external API call (jiska apna cost/rate-limit ho sakta hai) trigger karti hai. Agar koi user (ya bot) bina rate-limit ke 1000 req/sec bhejta hai, to ye teri external API quota jaldi khatam kar sakta hai, ya teri billing cost badha sakta hai, ya tera database overwhelm kar sakta hai.

Add:

```
User
 ↓
Rate Limiter
 ↓
Redis
 ↓
Backend
```

Rule:

```
100 requests/minute/user
```

Then:

```
Request 1-100 → allowed
Request 101+ → 429
```

And because Redis shared hai:

```
Server 1 ─┐
Server 2 ─┼→ Redis
Server 3 ─┘
```

limit globally enforce ho sakti hai.

**Ye poore chain ko tere project ke context mein tie kar:** Ye exactly wahi architecture hai jo poore is note mein discuss hui — chahe tera Movie Search API kitne bhi backend instances par scale ho (horizontal scaling, jaisa Sharding notes mein bhi dekha), Redis-based shared rate limiting ensure karega ki total limit (100 req/min/user) accurately enforce ho, chahe requests kisi bhi instance pe route hon.

---

## 14. Tere Chat App mein

Suppose:

```
POST /messages
```

Limit:

```
60 messages/minute/user
```

Why?

Spam protection.

**Ye limit ka specific reasoning samajh:** Chat applications mein, agar koi user (ya automated script) unlimited rate se messages bhej sake, to ye spam, abuse, ya even Denial-of-Service jaisa impact create kar sakta hai (jaise chat rooms ko flood karna, ya server resources exhaust karna). "60 messages/minute" jaisi limit ek reasonable threshold hai jo genuine human typing speed se zyada hai (isliye legitimate users affect nahi honge), lekin automated abuse ko prevent karta hai.

Architecture:

```
User
 ↓
Load Balancer
 ↓
Backend
 ↓
Redis Rate Limiter
 ↓
Message processing
```

But chat mein message delivery itself rate limiting se different concern hai.

Rate limiter bas decide karega:

> "Kya ye user abhi ek aur message send kar sakta hai?"

**Ye distinction important hai — rate limiter ka scope clearly define kar:** Rate limiter sirf ek "gatekeeper" hai jo decide karta hai "is action ko allow karna chahiye ya nahi" — uske baad actual message ko store karna, deliver karna, WebSocket ke through bhejna, ye sab **separate concerns** hain jo rate limiter ke scope se bahar hain (jo tere doosre notes — Singleton/PubSub — mein already cover ho chuki hain). Ye separation-of-concerns ek achhi design principle hai — rate limiter sirf apna ek kaam karta hai, achhe se.

---

## 15. Interview: "Why Redis?"

Strong answer:

> I need shared, low-latency state because requests from the same client can hit different backend instances. Redis is suitable because it's fast and provides atomic operations that can be used to safely update rate-limit state.

That's much better than:

> "Redis is fast."

**Ye do answers mein farak kyu itna significant hai:** "Redis is fast" ek generic, surface-level statement hai jo actual problem (distributed state sharing + race conditions) ko address nahi karta. Strong answer specifically batata hai **kis problem ko** Redis solve kar raha hai (shared state across instances) aur **kaise** (atomic operations, jo race conditions se bachate hain) — ye dikhata hai ki tu sirf "buzzword" nahi bol raha, balki underlying reasoning samajhta hai.

---

## 16. Interview: "Why not store the counter in PostgreSQL?"

Possible answer:

> A high-throughput rate limiter performs a state update on almost every request. Using the primary database for every rate-limit check can add significant load and latency. Redis is generally more appropriate for this short-lived, high-frequency state.

**Is answer ke reasoning ko deeply samajh:** Rate limiting ek aisi operation hai jo **har single incoming request** ke saath trigger hoti hai (potentially bahut high frequency pe, jaise thousands of requests per second). Agar ye har baar PostgreSQL (ek disk-based, ACID-compliant database) ko hit kare, to ye database par bahut zyada extra load daalega — jo already important, "real" application data (jaise orders, users) handle kar raha hai. Redis (in-memory, extremely fast) is tarah ke high-frequency, short-lived (rate-limit state jo baar baar change hoti hai) data ke liye zyada suited hai — ye database ko is extra burden se free rakhta hai.

---

## 17. Interview: "What happens if Redis goes down?"

This is where you should think instead of giving one universal answer.

There isn't one correct answer.

Options depend on how critical rate limiting is.

**Ye ek important interview-signal hai — "there isn't one correct answer" wali baat khud ek lesson hai:** Ye dikhata hai ki mature system design thinking mein, har decision ek trade-off hai — koi "universally correct" answer nahi hota, balki context-dependent judgment calls hote hain. Interviewer aksar aisa hi sawaal poochta hai ye test karne ke liye ki candidate blindly ek "template answer" rata hai, ya genuinely reasoning kar sakta hai.

### Fail-open

Redis unavailable:

```
Rate limiter unavailable
       ↓
Allow request
```

Advantage:

Application stays available.

Risk:

Abuse can bypass the rate limiter.

**"Fail-open" ka philosophy samajh:** Jab rate-limiting mechanism khud fail ho jaaye (Redis down), to system "default to allow" karta hai — matlab requests process hoti rahengi, chahe rate-limit check nahi ho paa raha. Ye approach "availability-first" hai — application ka core functionality (users ko serve karna) chalu rehta hai, chahe protection temporarily compromise ho.

### Fail-closed

Redis unavailable:

```
Rate limiter unavailable
       ↓
Reject request
```

Advantage:

Protection remains strict.

Risk:

Legitimate users may be blocked.

**"Fail-closed" ka philosophy samajh:** Jab rate-limiting mechanism fail ho jaaye, to system "default to reject" karta hai — matlab jab tak rate-limiter reliably kaam nahi kar raha, koi bhi request process nahi hogi. Ye approach "security/protection-first" hai — chahe legitimate users temporarily affected hon, system apni protection guarantees ko compromise nahi karta.

For something like a highly sensitive endpoint, you may prefer stronger protection; for less critical APIs, availability may matter more.

This is exactly the kind of trade-off interviewers want to hear.

**Ye final line ko heart mein utaar:** Interviewers specific "correct" answer nahi chahte — wo chahte hain candidate is trade-off ko articulate kar sake, aur context ke basis pe reasoning kar sake (jaise "payment endpoint ke liye fail-closed behtar, general search endpoint ke liye fail-open behtar"). Ye exact wahi kind of nuanced thinking hai jo CAP theorem, Replication, aur poore is series mein baar baar emphasize hua hai — trade-offs ko samajhna, blindly ek "rule" follow karne ki jagah.

---

## 18. Now complete architecture

```
                         CLIENT
                           │
                           ↓
                    Load Balancer
                           │
               ┌───────────┼───────────┐
               ↓           ↓           ↓
             Server 1    Server 2    Server 3
               │           │           │
               └───────────┼───────────┘
                           ↓
                         Redis
                    Rate Limit State
                           │
                           ↓
                     ALLOW / REJECT
                       /       \
                     YES        NO
                      ↓          ↓
                   Backend      429
                      ↓
                   Database
```

One correction to this diagram conceptually: the rate-limit check is normally done before expensive backend processing, and the Redis interaction is part of that check; it doesn't have to be a separate physical server sitting after all application servers.

**Is correction ko deeply samajh, ye ek subtle but important clarification hai:** Ye diagram thoda misleading dikh sakta hai agar tu sochta hai "requests pehle sab backend servers pe jaati hain, phir alag se Redis check hota hai." Actually, real implementation mein, rate-limit check **har backend server ke andar hi** (application code ke shuru mein) hoti hai — matlab Server 1 khud apna Redis call karta hai request process karne se pehle, Server 2 khud apna, aur Server 3 khud apna. Redis ek shared, central "data store" hai jise sab servers query karte hain, lekin ye koi alag "physical routing layer" nahi hai jiske through requests guzarti hain. Ye distinction implementation clarity ke liye important hai.

---

## 19. What you should now be able to answer

If I ask:

> "Design a rate limiter for 1M users."

You should start thinking:

```
What is the limit?
↓
Per IP / user / API key?
↓
Which algorithm?
↓
Do I allow bursts?
↓
Multiple backend servers?
↓
Need shared state → Redis
↓
Race condition?
↓
Need atomic update
↓
What happens if Redis fails?
↓
429 response
```

That's actual System Design thinking.

**Is poore mental checklist ko ek "framework" ki tarah treat kar interview mein:** Ye ek structured, sequential decision-tree hai jo tujhe kisi bhi rate-limiter design question ke liye guide kar sakta hai — shuru karo requirements clarify karke (limit kya hai, kis basis par), phir algorithm choose karo (bursts allowed chahiye ya nahi), phir distributed concerns address karo (multiple servers → shared state → race conditions → atomicity), aur end mein failure-handling discuss karo. Ye ek repeatable pattern hai jo tu kisi bhi similar open-ended system-design question pe apply kar sakta hai.

---

## Rate Limiting is now complete

You know:

```
Fixed Window
Sliding Window Log
Token Bucket
Leaky Bucket

        ↓

Distributed Rate Limiter
        ↓
Redis
        ↓
Atomicity
        ↓
Race Conditions
        ↓
429
        ↓
Failure handling
```

---

## 20. Interview Questions — Poori List with Deeper Answers

### Q1. Char rate-limiting algorithms mein se konsa use karega, aur kyu?

> Answer: It depends on requirements. Fixed Window is simplest but has a boundary-burst issue — good for non-strict use cases. Sliding Window Log is accurate but memory-heavy — good when precision matters and scale is manageable. Token Bucket allows controlled bursts, which fits most API rate limiting use cases well since real traffic is naturally bursty. Leaky Bucket smooths output completely, which is ideal when a downstream system needs a strictly predictable, consistent rate.

### Q2. Distributed system mein rate limiting kaise implement karega, single-server approach kyu kaam nahi karta?

> Answer: If each backend server keeps its own local rate-limit counter, a user hitting different servers (via a load balancer) could exceed the intended global limit — e.g., 100/min per server across 3 servers effectively allows 300/min. The solution is shared state via Redis, so all servers check and update the same rate-limit data regardless of which server handles the request.

### Q3. Redis shared state use karne ke baad bhi kya problem reh jaati hai?

> Answer: A race condition — if two requests from the same user hit different servers at nearly the same time, both might read the same "tokens available" state before either updates it, leading to both being incorrectly allowed. This requires making the read-check-update sequence atomic, typically using something like a Redis Lua script that executes the whole operation without interruption.

### Q4. Rate limiter fail ho jaaye (Redis down ho jaaye) to kya karega?

> Answer: There's no single correct answer — it's a trade-off between fail-open (allow requests when the limiter is unavailable, prioritizing availability but risking abuse) and fail-closed (reject requests, prioritizing protection but risking blocking legitimate users). The right choice depends on how sensitive the specific endpoint is — e.g., fail-closed for payments, fail-open for less critical read endpoints.

### Q5. Rate limiting sirf IP address ke basis par kyu nahi karni chahiye?

> Answer: IP-based limiting can unfairly affect legitimate users who share an IP (e.g., behind a corporate NAT or public WiFi), and it doesn't distinguish between different users on the same network. Rate limiting based on user ID, API key, or a combination (like user + endpoint) is generally more accurate and fair, and also allows different limits for different endpoints based on their sensitivity or cost.

---

### Bonus: Quick self-check before interview

Khud se ye sawaal poochh aur bina notes dekhe answer de:

1. Token Bucket aur Leaky Bucket mein core philosophical difference kya hai (burst allow karna vs smooth karna)?
2. Fixed Window ka "boundary burst" problem kya hai, aur Sliding Window Log isse kaise avoid karta hai?
3. Distributed system mein local, per-server rate-limit counters kyu problematic hain?
4. Redis shared state use karne ke baad bhi race condition kyu ho sakti hai, aur atomicity (jaise Lua script) isse kaise solve karti hai?
5. Fail-open aur fail-closed mein kya trade-off hai, aur kis type ke endpoint ke liye kaunsa better hoga?
6. Rate limiting kis kis basis par ho sakti hai (sirf IP nahi)?
7. Agar interviewer bole "design a rate limiter for 1M users," tera mental checklist kya hoga (step by step)?

Agar in sabka answer bina ruke de paya, to Rate Limiting topic solid hai.

---

### Extra: Ye concept tere projects se kaise connect hota hai

**Movie Search API:** Jaisa Section 13 mein detail se discuss hua, agar tu external movie API ko call kar raha hai, rate limiting directly relevant hai — teri khud ki API par rate limit lagana teri external API costs/quota ko protect karta hai.

**Messaging Platform (Chat App):** Jaisa Section 14 mein discuss hua, message-sending endpoint ke liye rate limiting spam/abuse protection ke liye zaroori hai — especially jab tu apna Socket.IO backend multiple servers pe scale karega (jahan Redis-based shared rate-limiting directly applicable hoga, exactly jaisa poore is note mein discuss hua).

**Exchange:** Order-placement API (jo tune Idempotency notes mein bhi dekha) ke liye rate limiting bhi relevant hai — kisi user ko har second bahut zyada orders place karne se rokna (jo matching engine ko overwhelm kar sakta hai) ek natural application hai in dono concepts (Idempotency + Rate Limiting) ka combined use.

**CodeArena / Retail-Store-Agent:** Kisi bhi API jo expensive computation trigger karti hai (jaise CodeArena mein code-execution jobs submit karna, ya Retail-Store-Agent mein ML predictions/reports generate karna), rate limiting us resource-intensive operation ko abuse se protect karne ke liye directly useful hoga — yahan Token Bucket ka "controlled burst" wala model particularly fit baithega, kyunki legitimate users occasionally multiple submissions kar sakte hain, lekin sustained high-rate abuse ko limit karna zaroori hai.