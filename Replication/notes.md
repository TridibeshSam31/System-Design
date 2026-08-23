# Replication — Poori Samajh (Enhanced Notes)

> Ye notes tere original Replication notes ka enhanced version hain — kuch bhi remove nahi kiya, bas har point ke saath "iska matlab kya hai", "kyu aisa hota hai" wale explanations add kiye hain.

---

## 0. Sabse pehle basic terminology clear kar

**Read vs Write operation ka farak:**

```
Read = data nikalna (SELECT) — data change nahi hota
Write = data change karna (INSERT / UPDATE / DELETE) — data modify hota hai
```

Ye distinction poore replication topic ki foundation hai — jaisa tu aage dekhega, reads aur writes ko alag alag jagah handle karna hi replication ka core idea hai.

**Redundancy kya hoti hai?**

Redundancy ka matlab hai — same cheez ki extra copies rakhna, taaki agar ek copy fail ho jaaye, to doosri copy se kaam chal jaaye. Ye "backup" jaisa concept hai — engineering mein ye reliability badhane ka fundamental tareeka hai.

---

## Sabse pehle problem.

Maan le tere paas ek PostgreSQL database hai:

```
Backend
   ↓
PostgreSQL
```

Saara data isi ek database mein hai.

**Single database setup ka risk samajh:** Jab saara data ek hi jagah hai, to us database ke saath do fundamental risks judte hain — (1) performance bottleneck, agar bahut zyada traffic ho, aur (2) single point of failure, agar wo database kisi bhi wajah se down ho jaaye, to poori application down ho jaati hai.

Ab traffic bahut badh gaya:

```
1000 requests/sec
        ↓
   PostgreSQL
```

Agar mostly read requests hain:

```
GET /users
GET /products
GET /posts
GET /orders
...
```

toh ek database par bahut read load aa sakta hai.

**Read-heavy traffic ka pattern samajh:** Real-world applications mein, generally reads writes se **kaafi zyada** hoti hain — jaise agar tu ek social media app socho, log posts baar baar dekhte hain (read) lekin nayi post kabhi kabhi hi banate hain (write). Isliye read traffic often bottleneck ban jaata hai — ek single database server ko itni saari read queries ek saath handle karna mushkil ho sakta hai.

Aur doosri problem:

> Agar ye database down ho gaya toh?

**Ye ek genuinely serious sawaal hai:** Agar sirf ek hi database hai aur wo crash ho jaaye (hardware failure, ya koi bug, ya maintenance ke liye restart), to jab tak wo wapas up na ho, poori application non-functional ho jaayegi — koi bhi read ya write kaam nahi karega. Ye business ke liye bahut costly ho sakta hai (downtime = lost revenue, unhappy users).

---

## 1. Replication kya hai?

> Replication = same data ki copies multiple database servers par maintain karna.

Instead of:

```
PostgreSQL
```

we have:

```
              PostgreSQL
                  ↓
        ┌─────────┴─────────┐
        ↓                   ↓
     Replica 1           Replica 2
```

Teeno mein same logical data ki copies hain.

**Ye "same logical data" ka matlab samajh:** In teeno databases (original + 2 replicas) mein content essentially **identical** hai (ya near-identical, agar thoda delay ho — jo aage discuss hoga). Ye Sharding se fundamentally different hai — Sharding mein har node ke paas **alag** data hota hai (data split hota hai), lekin Replication mein har node ke paas **wahi same** data hota hai (data copy hota hai). Ye important distinction hai jo aage detail mein aayega.

---

## 2. Primary aur Replica

Usually ek database ko:

```
Primary
```

kehte hain.

Aur baaki:

```
Replicas / Read Replicas
```

Example:

```
              Primary
             PostgreSQL
             /        \
            ↓          ↓
       Replica 1    Replica 2
```

**"Primary" ka special role kyu hai:** Primary ko "authoritative source" maana jaata hai — sabse latest, sabse correct data yahan pe hota hai. Replicas essentially Primary ka "follow" karte hain, uske changes ko copy karte hain. Ye ek hierarchical relationship hai — Primary "leader" hai, Replicas "followers" hain (isliye "leader-follower replication" bhi bola jaata hai kabhi kabhi).

Primary generally writes handle karta hai:

```
INSERT
UPDATE
DELETE
```

Aur replicas primarily reads handle kar sakte hain:

```
SELECT
```

**Ye division of labor kyu banayi gayi hai:** Writes ko sirf ek jagah (Primary) route karna zaroori hai taaki data consistency maintain rahe — agar multiple jagah simultaneous writes ho, to conflicts (jaise do log ek hi cheez ko alag-alag update karein) ka risk aa jaata hai. Reads ko multiple replicas mein spread karna safe hai kyunki reads data ko change nahi karte — bas dekh rahe hain, isliye jitne chahe utne jagah se read kiya ja sakta hai bina koi conflict ke.

---

## 3. Real request flow

Suppose user signup karta hai:

```
POST /users
```

Request:

```
Backend
   ↓
Primary DB
   ↓
INSERT user
```

**Ye kyu Primary tak jaata hai:** Signup ek write operation hai (naya user create ho raha hai), isliye ye seedha Primary database ko route kiya jaata hai — jaisa upar discuss hua, saare writes Primary se hi guzarte hain.

Then same data replicas tak replicate hota hai:

```
              Primary
                 ↓
        ┌────────┴────────┐
        ↓                 ↓
   Replica 1          Replica 2
```

**Replication process background mein kaise chalta hai:** Primary jab bhi apna data update karta hai, wo apni changes (jaise "naya row insert hua") ek continuous stream ke roop mein replicas ko bhejta hai. Ye process automatically, continuously chalti rehti hai — application code ko manually kuch nahi karna padta, database system khud ye handle karta hai.

Now user profile fetch karta hai:

```
GET /profile
```

Backend read ko replica par bhej sakta hai:

```
Backend
   ↓
Replica 1
   ↓
SELECT
```

So load distribute ho gaya.

**Ye routing decision kahan hoti hai:** Ye typically application/backend code (ya ek database proxy layer) mein decide hota hai — jaise "agar query SELECT hai, to kisi replica ko bhejo; agar INSERT/UPDATE/DELETE hai, to Primary ko bhejo." Ye logic ORM/database driver level pe bhi implement ho sakta hai, ya manually application code mein.

---

## 4. Why replication?

There are two major reasons you should remember.

### 1. Read scalability

Suppose:

```
100K read requests/sec
1K write requests/sec
```

Ek primary ko 100K reads bhi handle karne pad rahe hain.

**Ye numbers realistic scenario dikhate hain:** Notice karo ki reads (100K) writes (1K) se 100x zyada hain — ye bahut common pattern hai real applications mein. Agar sab kuch ek hi Primary database ko jaana pade, to wo bottleneck ban jayega, chahe writes ka load kam hi ho.

Instead:

```
                 Primary
                  ↑
                Writes
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   Replica 1            Replica 2
     Reads                Reads
```

Read traffic distribute kar sakte ho.

**Ye scaling kaise kaam karti hai:** Agar 100K reads ko sirf 2 replicas ke beech baant do, to har replica ko roughly 50K reads handle karni padengi — Primary par sirf 1K writes ka load rahega. Aur agar traffic aur badhe, to bas aur replicas add kar do (horizontal scaling, jaisa Sharding notes mein bhi dekha tha concept). Ye Primary ko free rakhta hai sirf writes handle karne ke liye, jo usually kam hote hain.

### 2. Availability / redundancy

Suppose:

```
Primary 💀
```

Replica available hai:

```
Replica 1 ✅
Replica 2 ✅
```

System failover karke kisi replica ko new primary bana sakta hai.

So replication gives you redundancy and can improve availability.

**Ye second reason pehle se fundamentally alag hai:** Pehla reason (read scalability) performance ke baare mein hai — jab sab kuch normal chal raha ho, tab better throughput. Doosra reason (availability) failure handling ke baare mein hai — jab kuch galat ho jaaye (Primary crash), tab bhi system chalta rahe. Dono independent benefits hain jo replication se milte hain — chahe tujhe sirf ek chahiye ho, tu replication use kar sakta hai, lekin real systems mein aksar dono fayde ek saath milte hain.

---

## 5. Replication vs Sharding

Ye bahut important interview question hai.

### Sharding

Data divide karta hai.

```
10M users

Shard 1 → users 1–3M
Shard 2 → users 3–6M
Shard 3 → users 6–10M
```

Ek shard ke paas sirf data ka ek portion hai.

### Replication

Data copy karta hai.

```
Primary → ALL data
Replica 1 → ALL data
Replica 2 → ALL data
```

So:

```
Sharding
→ split data

Replication
→ copy data
```

**Ye distinction ek line mein lock kar:** Sharding mein har node "different data ka piece" hai. Replication mein har node "same data ki poori copy" hai. Ye do completely different problems solve karte hain — Sharding se tu **total storage/compute capacity** badhata hai (kyunki data multiple machines mein split hai), Replication se tu **availability aur read-throughput** badhata hai (kyunki same data multiple jagah available hai).

**Ek common confusion jo yahan clear karna zaroori hai:** Bahut log sochte hain sharding aur replication "competing" solutions hain — jaise "sharding karu ya replication?" — lekin ye galat framing hai. Ye dono orthogonal (independent) concerns hain jo alag alag problems solve karte hain, aur real systems mein saath saath use hote hain — jaisa agla section dikhata hai.

---

## 6. Dono ko combine bhi kar sakte ho

Real systems mein ye important hai.

Suppose:

```
                 Database
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Shard 1     Shard 2     Shard 3
        │           │           │
      Primary     Primary     Primary
       / \          / \         / \
      ↓   ↓        ↓   ↓       ↓   ↓
    Rep1 Rep2    Rep1 Rep2   Rep1 Rep2
```

Meaning:

```
Sharding → data ko divide kar raha hai
Replication → har shard ki copies bana raha hai
```

This is a very common distributed database architecture pattern.

**Is combined architecture ko step-by-step samajh:** Pehle data ko shards mein divide karte hain (jaise users ko 3 shards mein baant diya, based on user_id range ya hash) — ye scalability (storage/compute) ke liye hai. Phir har individual shard ko khud replicate karte hain (uska apna Primary + Replicas) — ye us specific shard ki availability aur read-throughput ke liye hai. Isse tujhe dono fayde milte hain simultaneously — poore system ki horizontal scalability (sharding se) aur har piece ki fault-tolerance/read-scaling (replication se). Ye exactly wahi pattern hai jo tune Sharding wale notes mein bhi dekha tha (Section 11: "Sharding vs Replication" — And real systems often use both).

---

## 7. Synchronous vs Asynchronous Replication

Ab important concept.

Suppose:

```
Primary
   ↓
Replica
```

User writes:

```
balance = ₹500
```

Primary update ho gaya.

Question:

> Replica ko update hone ka wait karna hai ya nahi?

**Ye sawaal itna crucial kyu hai:** Ye ek fundamental design decision hai jo poore system ke behavior ko affect karta hai — speed vs consistency ka trade-off (jo tujhe CAP theorem wale notes se bhi familiar lagega). Ye decision batata hai ki jab user ko "success" response milta hai, tab replicas ki state kya hai.

### Asynchronous Replication

Primary:

```
Write
 ↓
Primary updated
 ↓
Response to user ✅
```

Replica ko update baad mein mil sakta hai.

```
Primary
   ↓
   ↓ (later)
Replica
```

**Ye flow kya guarantee (ya nahi) deta hai:** User ko turant response mil jaata hai jaise hi Primary update hoti hai — Primary ko replica ke response ka wait nahi karna padta. Replica ko update **eventually** milega (isliye ye ek "eventually consistent" approach hai), lekin exact timing guaranteed nahi hai — kuch milliseconds se lekar (rare cases mein) seconds tak lag sakta hai.

### Advantage

Fast.

**Ye kyu fast hai:** Kyunki Primary ko sirf apna khud ka write complete karna hai, kisi network round-trip ka wait nahi karna (jo replica ko contact karne mein lagta). Isliye response time minimal hai.

### Problem

Replication lag.

Immediately replica se read kiya:

```
Primary → ₹500
Replica → ₹1000
```

Replica stale hai.

**Ye problem practically kab surface hoti hai:** Agar write hone ke turant baad koi read replica se ho, to us read ko purana data mil sakta hai — kyunki replica ko abhi tak Primary ka latest update nahi mila. Ye tab dangerous ho sakta hai jab application logic assume kare ki write ke turant baad wahi data read hoga (jaise "signup karo, phir turant profile dikhao" — agar profile read replica se ho, purana/empty data dikh sakta hai).

---

## 8. Replication Lag

Ye term yaad rakh.

> Replication lag = primary par write hone aur replica par same write available hone ke beech ka delay.

Example:

```
10:00:00
Primary = ₹500

10:00:00
Replica = ₹1000

10:00:01
Replica = ₹500
```

That 1 second is replication lag.

**Is example ko deeply samajh:** Time `10:00:00` par, Primary already update ho chuka hai (₹500), lekin Replica abhi bhi purana data (₹1000) dikha raha hai — is exact moment par, agar koi Replica se read kare, use galat/stale data milega. Time `10:00:01` par (1 second baad), Replica ko finally update mil jaata hai. Ye 1-second ka gap hi "replication lag" hai.

**Replication lag real world mein kitna hota hai:** Ye kai factors pe depend karta hai — network speed, kitna data change hua, replica kitni door hai (geographically), database ka load, etc. Typically ye milliseconds mein hota hai, lekin heavy load ya network issues ki wajah se ye seconds tak bhi badh sakta hai — jo user-facing problems create kar sakta hai.

---

## 9. Synchronous Replication

Ab opposite.

Primary write karta hai:

```
Write
 ↓
Primary
 ↓
Replica
 ↓
Replica confirms
 ↓
Response
```

Primary user ko response dene se pehle replica ka confirmation wait kar sakta hai.

**Ye flow kaise different hai asynchronous se:** Yahan Primary sirf apna khud ka write complete karke response nahi de deta — wo pehle replica(s) ko bhi wahi write bhejta hai, aur unse confirmation (acknowledgment) ka wait karta hai ki unhone bhi safely wo data store kar liya hai. Sirf tabhi Primary user ko "success" response deta hai.

### Advantage

Replica zyada immediately consistent hota hai.

**Ye guarantee kaise milti hai:** Kyunki jab tak user ko response nahi milta, tab tak Primary ye ensure kar chuka hota hai ki replica bhi updated ho chuka hai — isliye us write ke baad turant bhi agar replica se read ho, use latest data hi milega (koi replication lag ka risk nahi, at least us specific replica ke liye jisne confirm kiya).

### Disadvantage

Latency badh sakti hai.

**Ye latency kahan se aati hai:** Ab response dene ke liye Primary ko ek extra network round-trip ka wait karna padta hai (Primary → Replica → confirmation wapas Primary tak) — ye time add hota hai total response time mein, especially agar replica geographically door ho (jaisa CDN notes mein bhi discuss hua tha — distance = latency).

Aur agar replica unavailable hai:

```
Primary
   ↓
Replica ❌
```

toh write slow/fail ho sakti hai depending on the configuration.

**Ye ek serious operational risk hai:** Agar Primary ko replica ke confirmation ka wait karna hi hai, aur replica kisi wajah se down/unreachable ho jaaye, to Primary bhi effectively "stuck" ho jaata hai — ya to wo wait karta rahega (jisse writes bahut slow ho jaayenge), ya configuration ke hisaab se write hi fail ho sakta hai. Isliye synchronous replication ek availability risk introduce karta hai jo asynchronous mein nahi hota — ye exact wahi CP vs AP wala trade-off hai jo tune CAP theorem mein padha tha.

---

## 10. Simple comparison

**Asynchronous:**

```
Client
  ↓
Primary
  ↓
Response immediately
  ↓
Replica later
```

**Synchronous:**

```
Client
  ↓
Primary
  ↓
Replica confirmation
  ↓
Response
```

So:

> Async → faster, but replica can be stale

> Sync → stronger consistency, but higher latency/dependency

**Is comparison ko interview mein kaise use karna hai:** Ye ek classic trade-off hai jo tujhe explicitly bolna chahiye jab bhi replication discuss ho — "there's no universally 'better' option, it depends on the use-case's tolerance for staleness vs latency." Jaise financial transactions ke liye shayad synchronous behtar ho (correctness critical hai), lekin ek social media "likes count" ke liye asynchronous kaafi hai (thoda stale chalega, speed zyada matter karti hai).

---

## 11. Failover

Suppose:

```
              Primary
                 💀
                /  \
               ↓    ↓
          Replica1 Replica2
```

System ko decide karna padega:

> Kaunsa replica new primary banega?

Suppose Replica 1 selected:

```
Replica 1
     ↓
New Primary
```

Then:

```
              New Primary
                  │
                  ↓
              Replica 2
```

Is process ko broadly:

```
Failover
```

kehte hain.

**Failover process ko step-by-step samajh:**
1. System (ya monitoring tool) detect karta hai ki Primary down ho gaya hai (usually health-checks ke through — periodically Primary ko "ping" karte rehna).
2. Ek replica ko naya Primary "promote" kiya jaata hai — ye decision automatically (kisi consensus algorithm se) ya manually (human intervention se) ho sakta hai.
3. Baaki replicas ab is naye Primary ko follow karna shuru kar dete hain.
4. Application/backend ko bhi update karna padta hai ki writes ab kis naye Primary ko route karni hain.

Ye poora process **downtime minimize** karne ke liye hota hai — bina failover ke, Primary crash matlab poori application down, jab tak koi manually fix na kare.

---

## 12. But failover mein problem bhi hai

Suppose replication async thi.

Primary ke paas latest:

```
id = 100
```

Replica 1 ke paas:

```
id = 98
```

Primary crash.

Replica 1 primary ban gaya.

IDs 99 and 100 ka data potentially missing ho sakta hai.

That's one reason replication strategy matters.

**Ye ek genuinely serious data-loss scenario hai, deeply samajh:** Kyunki replication asynchronous thi, Primary jo latest 2 writes (id 99 aur 100) kar chuka tha, wo abhi tak Replica 1 tak nahi pahunche the (replication lag ki wajah se). Jab Primary crash ho gaya aur Replica 1 naya Primary bana, wo un 2 writes ke baare mein poori tarah unaware hai — jaise wo writes kabhi hue hi nahi. Agar Primary crash hone se pehle usne user ko "success" bhi bol diya tha un writes ke liye, to ab wo data permanently lost hai — user ko lagega uska data save ho gaya, lekin actually wo gayab ho chuka hai. Ye ek real, serious risk hai async replication + failover ke combination mein.

**Ye kyu synchronous replication ke saath kam hota:** Agar replication synchronous hoti, to Primary tab tak "success" response nahi deta jab tak replica confirm na kar de — isliye jo bhi writes user ko confirm hue the, wo guaranteed roop se replica mein bhi maujood hote, chahe Primary crash ho jaaye. Yahi trade-off hai — synchronous replication is data-loss risk ko kam karta hai, lekin latency/availability ki cost pe (jaisa upar discuss hua).

Abhi isko distributed consensus ke level tak mat le ja. Tere current System Design scope mein replication lag + failover ka intuition enough hai.

**Ye scope-limiting note zaroori hai:** Is problem ka "proper" solution (jaise Raft, Paxos consensus algorithms) bahut deep, academic topics hain jo production-grade distributed databases (jaise CockroachDB, etcd) implement karte hain. Tere current level ke liye, bas itna samajhna kaafi hai ki "async replication + failover = potential data loss risk" — ye concept-level intuition interview ke liye sufficient hai.

---

## 13. Read-after-write problem

Ye interview mein useful hai.

User:

```
POST /profile
```

Primary:

```
name = Tridibesh
```

Immediately:

```
GET /profile
```

Agar GET replica par gaya:

```
Replica
name = old value
```

User ko lagega:

> "Maine update kiya tha, phir old data kyun?"

Because:

```
Primary updated
       ↓
replication lag
       ↓
Replica stale
```

This is called a read-after-write consistency problem.

**Ye problem naam se hi samajh aati hai:** "Read-after-write" — matlab jab tu write karne ke turant baad wahi data read karta hai, to expectation hoti hai ki tujhe apna khud ka latest write dikhega. Lekin agar us read ko galti se kisi stale replica pe route kar diya jaaye, to ye expectation break ho jaati hai — user ko lagta hai system "buggy" hai, jabki actually ye ek known trade-off hai async replication ka.

**Real-world impact samajh:** Ye ek bahut common, user-facing bug pattern hai — jaise koi apna profile photo update kare, page refresh kare, aur purani photo dikhe (kyunki us read ne stale replica hit kiya). Ye user experience ko kharab karta hai chahe technically system "eventually" sahi ho jayega.

One possible solution is to route a user's critical read to the primary for some period/operation, depending on the system's consistency requirements.

**Ye solution kaise kaam karta hai:** Idea ye hai ki har cheez ko blindly replica pe route na karo — kuch specific, "critical" reads (jaise "just after this user wrote something") ko temporarily Primary pe hi route karo, taaki latest data guaranteed mile. Baaki general reads replicas pe rahen (jahan thoda staleness acceptable hai). Ye ek selective/hybrid approach hai jo har application ki apni consistency requirements ke hisaab se customize hoti hai.

---

## 14. Tere projects mein mapping

### Chat App

Messages:

```
Primary DB
    ↓
Replicas
```

Writes primarily primary par.

Reads replicas se scale ho sakti hain.

But recent message immediately read karne par replication lag ka issue consider karna padega.

**Ye kaise practically apply hota hai:** Agar tu apne chat app mein message history load kar raha hai (purane messages), to replicas se read karna bilkul theek hai — koi urgency nahi hai us data ke bilkul latest hone ki. Lekin agar koi user abhi-abhi message bheja hai aur turant "sent" confirmation ke saath us message ko chat window mein dikhana hai, to us specific read ko Primary se karna behtar hoga (ya application-level mein already available data use karo, jo already Primary se aaya tha), taaki replication lag ki wajah se message "missing" na dikhe.

### Exchange

Yahan blindly bolna:

> "Orderbook ko replicas mein daal do."

Wrong.

Tere Exchange project mein matching engine ka live orderbook in-memory state hai. Replication of the persistent database is a separate concern from replicating the matching engine's live state.

Ye distinction important hai.

**Is distinction ko deeply samajh, kyunki ye ek common galti hai:** Ye notes ki teri Stateful vs Stateless wali discussion se directly connect hota hai — orderbook ek **stateful, in-memory** data structure hai jo matching engine ke andar live rehta hai (fast matching ke liye), na ki ek regular database table jise tu normal database replication se replicate kar sako. Database replication (jo humne poore notes mein discuss kiya) us **persistent** data ke liye hai (jaise completed orders, trade history, user accounts) — ye standard PostgreSQL-jaisi replication se handle hota hai. Lekin matching engine ka live, in-memory orderbook state ek completely different problem hai — agar isko replicate karna ho (fault-tolerance ke liye), to alag techniques (jaise event-sourcing, ya specialized in-memory replication) chahiye hongi, standard DB replication kaam nahi karegi. Interview mein ye distinction bolna dikhata hai ki tujhe pata hai "replication" ek generic term nahi, balki context-specific implementation details matter karti hain.

---

## 15. Replication + CAP

Ab CAP se connection dekh:

Suppose:

```
Primary
   ❌ network partition
   |
   X
   |
Replica
```

Ab distributed system ko consistency/availability trade-offs deal karne pad sakte hain.

So CAP samajhne ke baad replication padhna useful tha.

But replication ≠ CAP.

Replication ek architecture technique hai.

CAP distributed systems ka trade-off theorem hai.

**Ye connection ko concretely samajh:** Jab Primary aur Replica ke beech network partition ho jaaye (exactly wahi scenario jo tune CAP theorem notes mein Node A/Node B ke saath dekha tha), to system ko decide karna padta hai — kya replica ab bhi requests serve kare (chahe potentially stale ho — AP wala choice), ya requests reject/delay kare jab tak partition resolve na ho (CP wala choice)? Ye exact wahi trade-off hai jo CAP theorem describe karta hai, bas ab ek concrete, real technique (replication) ke context mein apply ho raha hai.

**Ye distinction bhi zaroori hai:** Replication ek **specific implementation technique** hai (kaise data ki copies banayi jaayein, kaise sync ki jaaye). CAP theorem ek **broader theoretical framework** hai jo batata hai ki distributed systems mein fundamentally kya trade-offs possible hain. Replication implement karte waqt, tu automatically CAP ke trade-offs face karta hai — lekin replication khud CAP nahi hai, ye CAP ke concepts ko practically implement karne ka ek tareeka hai.

---

## 16. Interview questions

### Q1. Why use database replication?

Strong answer:

> To improve read scalability and provide redundancy/failover. Read replicas can handle read traffic while the primary handles writes.

### Q2. Sharding vs replication?

> Sharding partitions data across nodes, while replication creates copies of the same data across nodes.

### Q3. What is replication lag?

> The delay between a write being committed on the primary and that write becoming available on a replica.

### Q4. Why can read replicas return stale data?

> Because replication may be asynchronous, so the replica may not have received the latest write yet.

### Q5. What happens if primary fails?

> A replica can be promoted to primary through a failover process, but the system must account for replication lag and potentially missing recent writes.

*Extra tip in-depth follow-ups ke liye:*

**Q6. Synchronous aur Asynchronous replication mein kab kaunsa choose karega?**

> Answer: For systems where data loss is unacceptable — like financial transactions — synchronous replication is safer, since it guarantees the replica has the data before confirming success to the user, at the cost of higher latency. For systems where speed matters more than perfect immediate consistency — like social media feeds or like-counts — asynchronous replication is preferable, accepting brief staleness for better performance.

**Q7. Read-after-write consistency problem kaise solve karega?**

> Answer: A common approach is routing certain critical reads (typically right after a write by the same user) directly to the primary instead of a replica, ensuring the user always sees their own latest write, while letting general reads continue to use replicas for scalability.

---

## Final mental model

```
                    APPLICATION
                         │
                ┌────────┴────────┐
                ↓                 ↓
              WRITE              READ
                ↓                 ↓
             PRIMARY         READ REPLICA
                │                 ↑
                └──────→──────────┘
                  replication
```

Remember these 5 lines:

```
Replication = copies of data.

Primary = generally handles writes.

Replica = can handle reads.

Async replication = replica may temporarily be stale.

Failover = promoting another node when primary fails.
```

That's enough for the replication foundation. Next, Idempotency is the right topic before we hit Rate Limiting.

---

### Bonus: Quick self-check before interview

Khud se ye sawaal poochh aur bina notes dekhe answer de:

1. Replication kya problem solve karti hai, aur iske do major reasons kya hain?
2. Primary aur Replica ka role kya hai — kaun writes handle karta hai, kaun reads?
3. Sharding aur Replication mein fundamental farak kya hai, aur ye dono ek saath kaise use hote hain?
4. Synchronous aur Asynchronous replication mein trade-off kya hai?
5. Replication lag kya hai, aur ye real-world mein kaise problems create kar sakta hai (jaise read-after-write)?
6. Failover kya hai, aur async replication ke saath failover mein data-loss ka risk kyu hota hai?
7. Tere Exchange project mein orderbook ko database replication se kyu replicate nahi kar sakte?

Agar in sabka answer bina ruke de paya, to Replication topic solid hai.

---

### Extra: Ye concept tere projects se kaise connect hota hai

**Loan-Prediction-Model / Retail-Store-Agent:** Agar in projects mein tu PostgreSQL use kar raha hai (jaisa tera stack hai — Prisma + PostgreSQL), aur future mein scale karna pade (bahut saare predictions/reports read kiye ja rahe hon), to read replicas ek natural next step honge — model predictions ya historical data read-heavy operations ke liye replicas se serve ho sakte hain, jabki naye data points/training data writes Primary ko jaayenge.

**Messaging Platform:** Jaisa Section 14 mein discuss hua, chat messages ke liye replication + read-after-write ka consideration directly applicable hai — especially jab tu apne Socket.IO backend ko scale karega multiple servers ke across.