# Stateful vs Stateless, Singleton Pattern & Pub/Sub — Poori Samajh (Enhanced Notes)

> Ye notes tere original notes ( Singleton Pattern, Backend State Management and Pub Subs) ka enhanced version hain — kuch bhi remove nahi kiya, code bhi wahi rakha hai, bas har point ke saath "iska matlab kya hai", "kyu aisa hota hai" wale explanations add kiye hain, aur kahin kahin thoda extra useful content bhi joda hai.

---

## 0. Sabse pehle basic terminology clear kar — "State" kya hota hai?

Isse pehle stateful/stateless samjhein, "state" ka matlab clear kar le:

**State kya hai?**

"State" ka matlab hai — koi bhi data jo **time ke saath change ho sakta hai**, aur jise system ko yaad rakhna padta hai taaki wo sahi tarike se behave kare. Jaise:

```
User ka current balance
Game ka current board position
Kitne messages abhi tak bheje gaye
Kaun kaunse users online hain
```

Ye sab "state" hai — kyunki ye values fixed nahi hain, ye application chalte waqt change hoti rehti hain, aur system ko in values ko kahin na kahin "yaad" rakhna padta hai.

**Server "state rakhta hai" ka matlab kya hai?**

Jab hum bolte hain "server state rakhta hai", to iska matlab hai ki server apni **in-memory variables** (RAM mein) mein ye data store kar raha hai — na ki har baar database se fetch kar raha hai. Isse agla sawaal aata hai: agar server apni memory mein data rakhta hai, to kya hota hai agar request kisi doosre server par chali jaaye? Yahi poore is topic ka core hai.

---

# Part A: Stateful vs Stateless Backends

> Common interview question — ye tera original note bhi bolta hai, aur genuinely, ye backend system design ka ek foundational topic hai jo bahut interviews mein poocha jaata hai.

## 1. Stateless Backend

Normally HTTP backend mein server apni memory mein important state nahi rakhta.

**"Usually" pe focus kar:** Ye ek design choice hai, koi hard rule nahi — zyada tar HTTP backends conventionally stateless design follow karte hain, kyunki isse bahut saare operational fayde milte hain (jo aage dekhenge).

Example:

```
User
 ↓
Load Balancer
 ↓
Server 1 / Server 2 / Server 3
 ↓
Database
```

**Is architecture ka matlab samajh:** Load balancer ek "traffic director" hai jo incoming requests ko multiple servers ke beech distribute karta hai. Stateless design mein har server independent hota hai — koi bhi server kisi bhi request ko handle kar sakta hai, kyunki koi server-specific data nahi hai jiski zarurat pade.

Suppose user ka naam database mein hai:

```
DB:
userId = 10
name = Tridibesh
```

Request Server 1 par gayi:

```
GET /user/10
```

Server 1 database se data nikaal ke response de deta hai.

**Ye step-by-step kaise hota hai:** Server 1 ko request milti hai, wo database se query karta hai (`SELECT * FROM users WHERE id = 10`), data milta hai, response bana ke user ko bhej deta hai. Server ne khud kuch bhi apni memory mein "remember" nahi kiya — sara zaroori data database se aaya.

Next request Server 2 par chali gayi:

```
GET /orders
```

Server 2 ko koi problem nahi, kyunki required state database mein hai.

**Ye kyu koi problem nahi banata:** Server 2 ko us user ke baare mein pehle se kuch bhi "yaad" rakhne ki zarurat nahi thi — jab bhi request aayegi, jo bhi data chahiye hoga, wo directly database se fetch kar liya jayega. Isliye chahe request kisi bhi server pe jaaye, result same milega.

Iske 2 advantages PDF mein diye hain:

### 1. User kisi bhi server se connect ho sakta hai.

```
Request 1 → Server 1
Request 2 → Server 3
Request 3 → Server 2
```

Stickiness ki zarurat nahi.

**"Stickiness" word yahan pehli baar aaya — abhi bas itna samajh:** Stickiness ka matlab hota hai ki ek user ko hamesha ek hi specific server se connect rakha jaaye. Stateless systems mein iski zarurat nahi hoti, kyunki koi bhi server kaam kar sakta hai — is concept ko hum aage detail mein explore karenge.

### 2. Autoscaling easy hai.

Traffic badha:

```
3 servers → 10 servers
```

Traffic kam hua:

```
10 servers → 3 servers
```

Aur load balancer decide kar sakta hai kis server ko traffic dena hai.

**Autoscaling itna easy kyu hota hai stateless mein:** Jab servers ke paas koi unique/specific data nahi hai (sab kuch database mein hai), to naya server add karna trivial hai — bas ek naya server spin up karo, wo turant kaam karna shuru kar sakta hai (usse koi purana data "sync" karne ki zarurat nahi). Isi tarah, kisi server ko band karna bhi safe hai — us server ke paas koi aisa data nahi hai jo kahin aur na ho (sab DB mein persist hai). Ye flexibility production systems ke liye bahut valuable hai, especially jab traffic unpredictable ho (jaise sale events, viral content).

---

## 2. Stateful Backend

Kabhi-kabhi server ko apni memory mein state rakhni padti hai.

PDF ke examples:

```
1. in memory cache
2. real-time game ka state
3. chat application ke latest 10 messages
```

**Ye examples kyu stateful category mein aate hain:** In teeno cases mein, server apne khud ke RAM mein kuch data hold kar raha hai — sirf database pe depend nahi kar raha. Aisa kyu karte hain? Kyunki in-memory data access karna database se access karne se **bahut faster** hota hai (RAM access nanoseconds mein hoti hai, database query milliseconds mein — jo relatively bahut slow hai). Real-time applications (jaise games, chat) ke liye ye speed difference matter karta hai.

Example game:

```
Server 1

games = [
   Game A,
   Game B
]
```

Server 1 ko Game A ki current state pata hai.

**Ye "current state" mein kya hota hai:** Ek chess game ki state mein hota hai — board position, whose turn hai, kaunse moves khele gaye, kitna time bacha hai, etc. Ye poori state agar har move ke baad database mein read/write ki jaaye, to real-time gameplay bahut slow ho jayega. Isliye ye state server ki memory mein rakhi jaati hai, taaki instant access ho sake.

Ab user ki next request Server 2 par chali gayi:

```
User
 ↓
Load Balancer
 ↓
Server 2
```

Server 2 ke paas:

```
games = []
```

Toh usko Game A ka state nahi pata.

**Yahi asli problem hai, isko deeply samajh:** Server 2 ki apni khud ki alag memory hai — Server 1 ke andar jo `games` array hai, wo Server 2 ke paas exist hi nahi karta (ye alag processes hain, alag machines par bhi ho sakte hain). Isliye agar Game A ka data sirf Server 1 ki memory mein hai, aur request Server 2 pe chali jaaye, to Server 2 ko us game ke baare mein kuch pata nahi — jaise wo game exist hi nahi karta uske liye. Ye ek critical failure hai jo user experience ko poori tarah tod sakta hai (jaise game mid-way band ho jaaye).

---

## 3. Stickiness

Is problem ke liye PDF stickiness batata hai.

Meaning:

> Jo user kisi specific room/game mein interested hai, usko same server se connected rakho.

**Ye kaise achieve hota hai practically:** Load balancer ko configure kiya jaata hai ki wo request ke kisi attribute (jaise user ID, session ID, ya cookie) ko dekh ke hamesha usi server ko route kare jahan pehle wo user route hua tha. Isliye ye "sticky" hai — connection ek baar establish hone ke baad, "chipak" (stick) jaata hai us server se.

Example:

```
User A
   ↓
Load Balancer
   ↓
Server 1
```

Ab User A ki requests:

```
User A → Server 1
User A → Server 1
User A → Server 1
```

Server 1 ke paas uska state already hai.

**Ye kyu zaroori hai stateful case mein:** Kyunki Server 1 ke paas already User A ke game/chat ki state hai memory mein, isliye agar future requests bhi Server 1 pe hi jaayen, to server ko hamesha wo state milegi jiski zarurat hai — koi "state missing" wali problem nahi hogi.

### Important exception

PDF specifically bolta hai ki in-memory cache ke case mein stickiness zaroori nahi, kyunki cache miss hone par doosra server data fetch kar sakta hai.

Lekin game/chat jaise cases mein stickiness required ho sakti hai.

**Ye exception kyu hai, deeply samajh:** In-memory cache ka poora point hi ye hai ki wo "optional speedup" deta hai — agar cache mein data mile (cache hit), to fast response. Agar na mile (cache miss), to server bas original source (jaise database) se data fetch kar leta hai — thoda slow ho jaata hai, lekin **correctness break nahi hoti**, sirf performance thodi kam ho jaati hai. Isliye agar request Server 2 pe chali jaaye jahan cache mein wo data nahi hai, Server 2 bas DB se fetch kar lega — koi functional problem nahi hogi.

Lekin game/chat ke case mein, wo state **sirf** memory mein hai — koi backup/fallback (jaise database) nahi hai jahan se wo mil sake turant. Agar Server 2 ko wo state nahi mili, to game ka data hi "gayab" ho jaayega uske perspective se — ye ek hard failure hai, sirf performance issue nahi. Isliye yahan stickiness genuinely zaroori hai.

---

## 4. Ab question: JavaScript process mein state kaise rakhen?

> Good question to ask at this point is - How to store state in a JS project?

PDF ek simple example leta hai: `games`.

```typescript
interface Game {
  whitePlayer: string;
  blackPlayer: string;
  moves: string[];
}
```

Aur ek array:

```
games = []
```

Is array mein currently running games rakhe ja sakte hain.

**Ye simplest possible approach hai:** Bas ek in-memory array (ya object/map) bana lo jismein data rakh sako. Ab agla question hai — is array ko multiple files ke beech kaise share karein, taaki alag alag parts of code isko read/write kar sakein.

Ek file `store.ts` mein:

```
games[]
```

export karo.

Phir `index.ts` games add karega.

`logger.ts` games ko read karega.

Dono same exported array use karenge. PDF isi approach ko demonstrate karta hai.

**Code se dikhata hai:**

`store.ts` — Exports the game array:

```typescript
interface Game {
  whitePlayer: string;
  blackPlayer: string;
  moves: string[];
}

export const games: Game[] = [];
```

`index.ts` — pushes to games array:

```typescript
import { games } from "./store";
import { startLogger } from "./logger";

startLogger();

setInterval(() => {
  games.push({
    "whitePlayer": "harkirat",
    "blackPlayer": "jaskirat",
    moves: []
  })
}, 5000)
```

`logger.ts` — uses the games array:

```typescript
import { games } from "./store";

export function startLogger() {
  setInterval(() => {
    console.log(games);
  }, 4000)
}
```

**Ye code kya kar raha hai, step by step:**
1. `store.ts` ek array `games` declare aur export karta hai — ye centralized "storage" hai.
2. `index.ts` har 5 second mein ek naya game is array mein push karta hai (`setInterval` se simulate ho raha hai jaise real events aa rahe hon).
3. `logger.ts` har 4 second mein us array ko console mein print karta hai, taaki tu dekh sake ki state update ho rahi hai ya nahi.
4. Dono files same `games` array ko import kar rahe hain `store.ts` se — JavaScript modules mein, jab tu ek array/object export karta hai, to har jagah wahi **reference** (memory location) share hota hai, alag copies nahi banti. Isliye jab `index.ts` push karega, `logger.ts` ko bhi wahi updated array dikhega.

Ye approach basic level pe kaam karta hai — lekin next section mein iska limitation samjhenge.

---

## 5. Lekin sirf state nahi, functionality bhi chahiye

Problem ye hai ki sirf:

```
games[]
```

rakhna enough nahi hota.

> This will work, but a lot of times you need to attach functionality to state as well.

**"Functionality attach karna" ka matlab kya hai:** Sirf data rakhna kaafi nahi — us data ko **safe aur controlled tarike se manipulate** karne ke liye functions bhi chahiye. Jaise agar koi bhi file directly `games.push(...)` kar sakti hai, to koi validation, error-handling, ya business logic enforce nahi ho sakti (jaise "same game ID do baar add na ho", ya "move add karne se pehle check karo ki game exist karta hai ya nahi").

Hume functions bhi chahiye:

```
addGame()
getGames()
addMove()
logState()
```

Isliye PDF `GameManager` class banata hai.

**Class-based approach kyu better hai:** Class ke through, hum data (`games` array) aur us data pe operate karne wale functions (`addGame`, `addMove`, etc.) ko ek jagah **group** kar sakte hain. Ye Object-Oriented Programming ka basic principle hai — "encapsulation" — jahan data aur uspar kaam karne wale methods ek unit ke roop mein bundle rehte hain.

Conceptually:

```
GameManager
│
├── games[]
│
├── addGame()
├── getGames()
├── addMove()
└── logState()
```

Yani:

> GameManager state bhi rakhta hai aur us state ko manipulate karne ke functions bhi deta hai.

**Iska matlab: State + Behavior ek jagah.** Isse code zyada organized, testable, aur maintainable ban jaata hai — kyunki agar future mein `addGame` logic change karni ho (jaise validation add karni ho), to sirf ek jagah (class ke andar) change karna padega, poore codebase mein nahi dhundhna padega.

### Static attributes — ek zaroori concept jo aage chahiye hoga

> In JavaScript, the keyword `static` is used in classes to declare static methods or static properties. Static methods and properties belong to the class itself, rather than to any specific instance of the class.

**Ye concept kyu samajhna zaroori hai:** Normal class properties har instance ke saath alag hoti hain — matlab agar `new GameManager()` do baar call karo, to dono instances ki apni-apni alag `games` array hogi. Lekin `static` properties **class ke saath directly attach** hoti hain, instance ke saath nahi — matlab chahe kitne bhi instances bana lo, static property sirf ek hi baar exist karti hai, aur sab instances usi ek copy ko share karti hain.

Example of a class with static attributes:

```javascript
class Example {
  static count = 0;

  constructor() {
    Example.count++; // Increment the static property using the class name
  }
}

let ex1 = new Example();
let ex2 = new Example();
console.log(Example.count); // Outputs: 2
```

**Ye example kya demonstrate karta hai:** `count` ek static property hai — chahe `ex1` aur `ex2` do alag instances hain, dono ka constructor `Example.count` ko hi increment karta hai (na ki apni khud ki alag copy ko). Isliye final output `2` hai — dono instances ne mil ke ek hi shared counter ko update kiya. Ye exact concept aage Singleton pattern ke implementation mein use hoga — hum ek static property use karenge jo "single shared instance" ko track karegi.

> 💡 There are other ways of storing state in a TS project as well, redux being a popular one. Yes, you can use redux in the backend as well.

**Ye important side-note hai:** Redux, jo aksar frontend (React) mein state management ke liye use hota hai, backend mein bhi conceptually use ho sakta hai — kyunki fundamentally Redux bhi ek "centralized state store with controlled updates" pattern hai, jo backend state management ke liye bhi valid approach hai. Isse tujhe pata chalta hai ki GameManager/Singleton koi "only correct way" nahi hai — ye ek approach hai, aur alternatives (jaise Redux, ya external state stores) bhi exist karte hain.

---

# Part B: Classes and Singleton Pattern

## 6. Class banate hain jo state store aur manage kare

Let's create a class that:

```
1. Stores games
2. Exposes functions that let you mutate the state
```

```typescript
interface Game {
  id: string;
  whitePlayer: string;
  blackPlayer: string;
  moves: string[];
}

export class GameManager {
  private games: Game[] = [];

  public addGame(game: Game) {
    this.games.push(game);
  }

  public getGames() {
    return this.games;
  }

  // e5e7
  public addMove(gameId: string, move: string) {
    const game = this.games.find(game => game.id === gameId);
    if (game) {
      game.moves.push(move);
    }
  }

  public logState() {
    console.log(this.games);
  }
}
```

**Har part ka matlab samajh:**
- `private games: Game[] = []` — `private` keyword ka matlab hai ye property sirf class ke andar hi accessible hai, bahar se directly access nahi kar sakte (jaise `gameManager.games.push(...)` allowed nahi hoga). Ye encapsulation enforce karta hai — bahar se sirf defined methods (`addGame`, `addMove`, etc.) ke through hi state change ho sakti hai.
- `addGame(game: Game)` — naya game array mein add karta hai.
- `getGames()` — poore games array ko return karta hai (read access).
- `addMove(gameId, move)` — pehle `find()` se us specific game ko dhundhta hai jiski `id` match karti hai, phir agar mil jaaye to uske `moves` array mein naya move push karta hai. `// e5e7` ek chess move ka example hai (comment ke roop mein), jo dikhata hai ki `move` parameter mein aisi values aa sakti hain.
- `logState()` — debugging ke liye poora state console mein print karta hai.

---

## 7. Bad approach — Har file apna khud ka instance banaye

> Create saparate instance of `GameManager` in every file that needs it.

**"Bad" kyu bola gaya:** Ye approach superficially theek lagti hai — har file jahan bhi `GameManager` chahiye, wahan `new GameManager()` likh do. Lekin isme ek fundamental problem hai jo aage dikhegi.

`GameManager.ts`:

```typescript
interface Game {
  id: string;
  whitePlayer: string;
  blackPlayer: string;
  moves: string[];
}

export class GameManager {
  private games: Game[] = [];

  public addGame(game: Game) {
    this.games.push(game);
  }

  public getGames() {
    return this.games;
  }

  public addMove(gameId: string, move: string) {
    const game = this.games.find(game => game.id === gameId);
    if (game) {
      game.moves.push(move);
    }
  }
}
```

`logger.ts`:

```typescript
import { GameManager } from "./GameManager";

const gameManager = new GameManager();

export function startLogger() {
  setInterval(() => {
    gameManager.logState();
  }, 4000)
}
```

`index.ts`:

```typescript
import { GameManager } from "./GameManager";
import { startLogger } from "./logger";

const gameManager = new GameManager();

startLogger();

setInterval(() => {
  gameManager.addGame({
    id: Math.random().toString(),
    "whitePlayer": "harkirat",
    "blackPlayer": "jaskirat",
    moves: []
  })
}, 5000)
```

## Problem with `new GameManager()`

Maan le:

`index.ts`:

```typescript
const gameManager = new GameManager()
```

Aur `logger.ts`:

```typescript
const gameManager = new GameManager()
```

Ab kya hua?

```
index.ts
   ↓
GameManager A
   ↓
games[]

logger.ts
   ↓
GameManager B
   ↓
games[]
```

A aur B ke games alag hain.

**Ye kyu hota hai:** Har baar jab tu `new GameManager()` likhta hai, JavaScript ek **completely alag, fresh object** banata hai apni khud ki alag memory ke saath. Isliye chahe class definition same ho, `index.ts` mein bana instance aur `logger.ts` mein bana instance — dono independent objects hain, unke `games` arrays completely alag hain, ek doosre se koi connection nahi.

Agar `index.ts` ne game add kiya:

```
GameManager A
games = [Game 1]
```

Toh B ke paas:

```
GameManager B
games = []
```

PDF isi ko bad approach bolta hai.

**Ye practically kya problem create karta hai:** `index.ts` mein games add ho rahe hain (`GameManager A` ki memory mein), lekin `logger.ts` jo `logState()` call kar raha hai wo `GameManager B` ka state print kar raha hai — jo hamesha empty rahega! Isliye logger kabhi bhi actual games nahi dikhayega, chahe `index.ts` mein kitne bhi games add ho rahe hon. Ye exact wahi problem hai jo humne pehle (Section 2 mein) Server 1 vs Server 2 ke context mein dekhi thi — bas ab ye do alag instances ke beech ho rahi hai, same process ke andar.

---

## 8. Singleton Pattern

Ab requirement:

> Puri application mein GameManager ki ek hi instance ho.

Concept:

```
          GameManager
               ↑
        SAME INSTANCE
          ↙       ↘
     index.ts   logger.ts
```

Isko Singleton Pattern kehte hain.

**Singleton Pattern ki official definition samajh:** Singleton ek design pattern hai jo guarantee karta hai ki ek class ka sirf **ek hi instance** poore application mein exist kare, aur us instance ko access karne ka ek global point (usually ek method jaisa `getInstance()`) provide karta hai. Ye pattern software engineering mein bahut common hai, especially jab tujhe ek shared resource (jaise ek database connection, ek configuration object, ya yahan state manager) manage karna ho.

### Slightly Better approach

> Export a single instance of `gameManager` from `GameManager.ts` and use it everywhere.

**Ye ek intermediate solution hai:** Idea ye hai ki instance ko `new GameManager()` se banane ki jagah, `GameManager.ts` file ke andar hi ek baar instance bana lo aur usko export kar do — phir har jagah wahi ek exported instance import karo, naya instance na banao. Ye kaam karta hai, lekin isme ek weakness hai — koi bhi developer galti se phir bhi `new GameManager()` likh sakta hai kisi doosri file mein, aur wapas wahi purani problem ho sakti hai.

### Even better approach — Singleton Pattern

> Completely prevent any developer from ever creating a new instance of the `GameManager` class.

**Ye asli, robust solution hai:** Yahan idea ye hai ki hum sirf "convention" pe depend na karein (ki koi galti se `new` na kare), balki **code level pe hi enforce** kar dein ki naya instance banana **possible hi na ho** — chahe koi developer chahe bhi.

PDF ka implementation:

```typescript
class GameManager {

    private static instance;

    private constructor() {}

    static getInstance() {

        if (!GameManager.instance) {
            GameManager.instance = new GameManager();
        }

        return GameManager.instance;
    }
}
```

**Ye code line-by-line, deeply samajh:**

1. `private static instance;` — Ye ek **static** property hai (matlab class ke saath attach hai, kisi instance ke saath nahi — jaise humne upar `Example.count` mein dekha tha). Ye us ek single instance ko store karegi jo poori application use karegi. `private` hone ki wajah se ye bahar se directly access nahi ki ja sakti.

2. `private constructor() {}` — Ye sabse important line hai! Normally, JavaScript/TypeScript mein `constructor` public hota hai, matlab koi bhi bahar se `new GameManager()` likh sakta hai. Lekin yahan constructor ko `private` bana diya gaya hai — iska matlab hai **class ke bahar se `new GameManager()` likhna hi disallowed ho jaata hai** (TypeScript compiler error dega). Ye poore singleton pattern ka "lock" hai.

3. `static getInstance()` — Ye ek static method hai jo instance return karta hai. Iske andar logic hai:
   - Pehle check karo — kya `GameManager.instance` already exist karta hai?
   - Agar nahi (`!GameManager.instance` true hai), to naya instance banao (constructor ke andar se, kyunki hum class ke andar hi hain, isliye private constructor call karna allowed hai) aur usko `GameManager.instance` mein store kar do.
   - Agar already exist karta hai, to seedha wahi purana instance return kar do — naya nahi banao.

**Ye pattern effectively kya guarantee karta hai:** Pehli baar `getInstance()` call hone par, ek instance banega. Uske baad, chahe `getInstance()` kitni bhi baar, kahin se bhi call ho, **hamesha wahi ek instance** return hoga. Aur `new GameManager()` seedha likhna ab possible hi nahi hai (constructor private hai) — isliye system-level pe ye enforce ho gaya ki sirf ek hi instance kabhi exist karega.

Ab:

```
GameManager.getInstance()
```

har jagah same GameManager instance return karega.

---

## 9. Iska fayda

Ab:

```
index.ts
   ↓
GameManager.getInstance()
   ↓
       GameManager
          ↓
        games[]
```

Aur:

```
logger.ts
   ↓
GameManager.getInstance()
   ↓
       SAME GameManager
          ↓
        games[]
```

So index.ts game add karega toh logger.ts bhi wahi updated state dekh sakta hai.

PDF ka final implementation isi approach ko use karta hai.

**Ye kaise implement hota hai actual code mein:**

`GameManager.ts` (full implementation):

```typescript
interface Game {
  id: string;
  whitePlayer: string;
  blackPlayer: string;
  moves: string[];
}

export class GameManager {
  private static instance: GameManager; // Create a static instance of the class
  private games: Game[] = [];

  private constructor() {
    // Private constructor ensures that a new instance cannot be created from outside the class
  }

  public static getInstance(): GameManager {
    if (!GameManager.instance) {
      GameManager.instance = new GameManager();
    }
    return GameManager.instance;
  }

  public addGame(game: Game) {
    this.games.push(game);
  }

  public getGames() {
    return this.games;
  }

  public addMove(gameId: string, move: string) {
    const game = this.games.find(game => game.id === gameId);
    if (game) {
      game.moves.push(move);
    }
  }

  public logState() {
    console.log(this.games);
  }
}

// Usage GameManager.getInstance().addGame()
```

`logger.ts`:

```typescript
import { GameManager } from "./GameManager";

export function startLogger() {
  setInterval(() => {
    GameManager.getInstance().logState();
  }, 4000)
}
```

`index.ts`:

```typescript
import { GameManager } from "./GameManager";
import { startLogger } from "./logger";

startLogger();

setInterval(() => {
  GameManager.getInstance().addGame({
    id: Math.random().toString(),
    "whitePlayer": "harkirat",
    "blackPlayer": "jaskirat",
    moves: []
  })
}, 5000)
```

> Try creating a new instance of the `GameManager` class. Notice it wont let you.

**Ye khud try karke dekh:** Agar tu kahin bhi likhega `new GameManager()`, TypeScript compiler tujhe error dega kyunki constructor `private` hai. Ye compile-time enforcement hai — matlab galti runtime pe nahi, balki code likhte waqt hi pakdi jaayegi, jo bahut better hai (early error detection).

**Is poore change ka summary:** Pehle (`new GameManager()` wala approach) do alag files mein do alag, disconnected `games` arrays thi. Ab (`GameManager.getInstance()` wala approach) sirf ek hi `games` array hai jo poori application share karti hai — chahe kitni bhi files `getInstance()` call karein, sab ko wahi ek shared state milegi.

---

# Part C: Pub Sub + Singleton

## 10. Ab ek naya problem — Pub/Sub ki zarurat

Ab PDF ek naya problem deta hai.

Suppose 1 million+ users hain aur users stock prices ke updates subscribe karna chahte hain.

Example:

```
User A → BTC
User B → BTC
User C → ETH
```

Server ko track karna hai:

```
BTC → A, B
ETH → C
```

**Ye problem ka nature samajh:** Ye ek "who is interested in what" wala problem hai — bahut saare users hain, aur har user kisi specific cheez (jaise ek particular stock) mein interested hai. Jab wo cheez update hoti hai (jaise BTC ka price change hota hai), to sirf un users ko notify karna hai jo BTC mein interested hain, sabko nahi.

But multiple backend processes ho sakte hain.

**Ye extra complexity kyu add karta hai:** Agar tera system sirf ek server pe chal raha hota, to ye problem simple hoti — bas ek in-memory mapping rakh lo. Lekin real systems mein (jaise 1M+ users handle karne ke liye) multiple backend servers/processes chalte hain (horizontal scaling, jo tune Sharding notes mein bhi padha tha). Isliye ye track karna ki "kaun kis stock mein interested hai" ek distributed problem ban jaata hai.

Yahan Pub/Sub use hota hai.

**Pub/Sub ka simple meaning:**

**Publisher**

Event bhejta hai.

```
BTC price changed
```

**Subscriber**

Event receive karna chahta hai.

```
BTC mein interested users
```

**Publisher-Subscriber pattern ka core idea:** Ye ek messaging pattern hai jahan "publishers" events bhejte hain bina ye jaane ki kaun unhe receive karega, aur "subscribers" un events ko receive karte hain jinme unki interest hai, bina ye jaane ki wo event kisne bheja. Ye dono parties ko **decouple** kar deta hai — publisher ko subscriber ke baare mein directly kuch pata nahi hota, aur vice versa. Ek middleman (jaise Redis Pub/Sub) in dono ke beech events route karta hai.

PDF ke according Pub/Sub allows one or many processes to:

```
Publish events
Subscribe to events
```

---

## 11. Stock example

Suppose server ke users:

```
Server 1

BTC → User A
BTC → User B
ETH → User C
```

Server 1 ke `PubSubManager` ko ye information maintain karni hai.

User A BTC subscribe karta hai:

```
BTC
 ↓
User A
```

User B bhi:

```
BTC
 ↓
User A
User B
```

**Ye mapping kya represent karti hai:** Ye ek "stock → subscribers list" wali mapping hai — har stock ke against, uske interested users ki list. Jab User B bhi BTC subscribe karta hai, to bas list mein add ho jaata hai — existing subscribers untouched rehte hain.

Ab BTC ka event aaya:

```
BTC price = $100,000
```

Pub/Sub se event receive hua.

`PubSubManager` dekhega:

> BTC ke subscribers kaun hain?

```
A
B
```

Then event un users ke sockets ko forward karega.

**"Sockets ko forward karega" ka matlab:** Real-time updates dene ke liye, users ke browsers/apps ke saath ek persistent connection (WebSocket) khula hota hai. Jab BTC ka naya price aata hai, server us price update ko directly un users ke WebSocket connections pe bhej deta hai — isse users ko bina page refresh kiye, real-time mein price update dikh jaata hai.

PDF exactly ye responsibility PubSubManager ko deta hai.

---

## 12. PubSubManager + Singleton

Ab imagine ek server ke andar multiple files hain.

Hume nahi chahiye:

```
PubSubManager A
PubSubManager B
PubSubManager C
```

because subscriptions alag-alag ho jayengi.

**Ye exactly wahi problem hai jo GameManager ke saath dekhi thi:** Agar multiple files apna khud ka `PubSubManager` instance banayein, to har instance ki apni alag `subscriptions` mapping hogi — matlab User A ka subscription ek instance mein record hoga, lekin jab actual BTC event process karne wala code kisi doosre instance ko use kar raha hoga, use User A ka pata hi nahi chalega. Ye exact same root cause hai jo GameManager mein tha.

We want:

```
Server / Process
       ↓
 ONE PubSubManager
       ↓
subscriptions
```

Isliye:

> PubSubManager ko Singleton banaya gaya.

PDF mein PubSubManager ke andar:

```
subscriptions
redisClient
```

rakhe gaye hain, aur `getInstance()` same manager return karta hai.

**Ye do properties kyu chahiye:**
- `subscriptions` — in-memory mapping jo track karta hai ki us specific server (process) ke andar kaun-kaunse users kaunse stocks mein interested hain.
- `redisClient` — Redis se connection, jiske through actual stock-price events publish/receive hote hain (jab multiple servers involved hon, tab Redis ek central message broker ka kaam karta hai jo saare servers ke beech events distribute karta hai).

---

## 13. Bas PDF ka pura concept

Isko ye diagram yaad rakh:

```
STATEFUL vs STATELESS
        │
        ├── Stateless
        │      ↓
        │   State DB mein
        │      ↓
        │   Any server
        │      ↓
        │   Easy scaling
        │
        └── Stateful
               ↓
          State server memory mein
               ↓
          Same server needed
               ↓
            Stickiness


STATE MANAGEMENT
        ↓
    games[]
        ↓
   GameManager
        ↓
 Singleton Pattern
        ↓
 One instance


PUB/SUB
   ↓
Publish events
   +
Subscribe to events
   ↓
PubSubManager
   ↓
Singleton
   ↓
Track subscriptions
   ↓
Redis Pub/Sub
```

**Poora chain ek saath kaise fit hota hai:** Stateful backend hone ka matlab hai server apni memory mein kuch data rakhta hai (jaise subscriptions ya game state) — jisse stickiness ka concept aata hai. Us in-memory state ko properly manage karne ke liye Singleton Pattern use hota hai (taaki poori application mein ek hi, consistent state ho, alag-alag disconnected copies na banein). Aur jab ye state real-time events (jaise stock prices) ke saath deal kar raha ho jo multiple users tak pahunchane hain, Pub/Sub pattern use hota hai — jismein PubSubManager (khud ek Singleton) subscriptions track karta hai aur Redis ke through events ko route karta hai.

---

## 14. Pub Sub + Singleton (Implementation) — Practical Setup

Ye section original PDF mein tha lekin tere converted text mein sirf brief mention tha — poora implementation detail yahan hai taaki tujhe reference ke liye complete picture mile.

### Starting the pub sub

Start a pub sub (redis is a decent one):

```bash
docker run -d -p 6379:6379 redis
```

**Ye command kya karta hai:** Ye ek Redis server ko Docker container ke andar start karta hai, aur uska port `6379` (Redis ka default port) tere local machine ke port `6379` se map kar deta hai — taaki tu apne application se `localhost:6379` pe Redis se connect kar sake.

Try a simple publish subscribe in two terminals:

```bash
docker exec -it d1da6bcf089f /bin/bash
redis-cli
```

**Ye kya karta hai:** `docker exec -it <container-id> /bin/bash` us running Redis container ke andar ek interactive shell khol deta hai. Uske baad `redis-cli` Redis ka command-line client start kar deta hai, jisse tu directly Redis commands (jaise `SUBSCRIBE`, `PUBLISH`) run kar sakta hai, manually test karne ke liye ki pub/sub kaise kaam karta hai.

### Init a simple node.js project

```bash
npm init -y
npx tsc --init
npm install redis
```

**Har command ka matlab:**
- `npm init -y` — ek naya Node.js project initialize karta hai (`package.json` bana deta hai, `-y` flag sab defaults accept kar leta hai).
- `npx tsc --init` — TypeScript configuration file (`tsconfig.json`) banata hai.
- `npm install redis` — Node.js ka official `redis` npm package install karta hai, jo tujhe JavaScript se Redis ke saath interact karne deta hai.

### Creating the PubSubManager

```typescript
// Import the necessary module from the 'redis' package
import { createClient, RedisClientType } from 'redis';

export class PubSubManager {
  private static instance: PubSubManager;
  private redisClient: RedisClientType;
  private subscriptions: Map<string, string[]>;

  // Private constructor to prevent direct construction calls with the `new` operator
  private constructor() {
    // Create a Redis client and connect to the Redis server
    this.redisClient = createClient();
    this.redisClient.connect();
    this.subscriptions = new Map();
  }

  // The static method that controls the access to the singleton instance
  public static getInstance(): PubSubManager {
    if (!PubSubManager.instance) {
      PubSubManager.instance = new PubSubManager();
    }
    return PubSubManager.instance;
  }

  public userSubscribe(userId: string, stock: string) {
    if (!this.subscriptions.has(stock)) {
      this.subscriptions.set(stock, []);
    }
    this.subscriptions.get(stock)?.push(userId);

    if (this.subscriptions.get(stock)?.length === 1) {
      this.redisClient.subscribe(stock, (message) => {
        this.handleMessage(stock, message);
      });
      console.log(`Subscribed to Redis channel: ${stock}`);
    }
  }

  public userUnSubscribe(userId: string, stock: string) {
    this.subscriptions.set(stock, this.subscriptions.get(stock)?.filter((sub) => sub !== userId) || []);

    if (this.subscriptions.get(stock)?.length === 0) {
      this.redisClient.unsubscribe(stock);
      console.log(`UnSubscribed to Redis channel: ${stock}`);
    }
  }

  // Define the method that will be called when a message is published to the channel
  private handleMessage(stock: string, message: string) {
    console.log(`Message received on channel ${stock}: ${message}`);
    this.subscriptions.get(stock)?.forEach((sub) => {
      console.log(`Sending message to user: ${sub}`);
    });
  }

  // Cleanup on instance destruction
  public async disconnect() {
    await this.redisClient.quit();
  }
}
```

**Is poore class ko line-by-line samajh:**

1. **`private static instance` aur `private constructor()`** — Ye exact wahi Singleton pattern hai jo humne `GameManager` mein dekha tha. Sirf ek hi `PubSubManager` instance kabhi exist karega.

2. **`redisClient` aur `subscriptions`** — Constructor ke andar, `createClient()` se ek naya Redis client banaya jaata hai aur `connect()` se usse connect kiya jaata hai. `subscriptions` ek `Map` hai — is data structure ka use samajh: Map mein key `stock` (jaise "BTC") hoti hai, aur value ek array of `userId` hoti hai (wo saare users jo us stock mein interested hain).

3. **`userSubscribe(userId, stock)`** — Jab koi user kisi stock ko subscribe karta hai:
   - Pehle check karo ki kya us stock ke liye already koi entry hai Map mein. Agar nahi, to ek empty array bana ke set kar do.
   - Us user ki `userId` ko us stock ke subscribers array mein push kar do.
   - **Important optimization:** `if (this.subscriptions.get(stock)?.length === 1)` — ye check karta hai ki kya ye is stock ka **pehla** subscriber hai. Agar haan, to tabhi Redis ko actually subscribe karo (`this.redisClient.subscribe(stock, ...)`). Agar pehle se hi koi subscriber tha (matlab length already 1 se zyada ho jaayegi is push ke baad), to Redis subscription already active hai, dobara subscribe karne ki zarurat nahi. Ye efficient hai — Redis subscription sirf ek baar per stock hoti hai, chahe kitne bhi users interested hon.

4. **`userUnSubscribe(userId, stock)`** — User ko us stock ke subscribers list se filter karke hata deta hai. Agar us stock ka koi bhi subscriber nahi bacha (`length === 0`), to Redis subscription bhi cancel kar do (`unsubscribe`) — kyunki ab koi interested nahi hai, isliye events sunte rehna resource waste hai.

5. **`handleMessage(stock, message)`** — Jab Redis se koi event aata hai (jaise "BTC price changed"), ye function call hota hai. Ye us stock ke saare subscribers ko dhundhta hai (`this.subscriptions.get(stock)`), aur har ek ko message "bhejta" hai (yahan console.log hai, real system mein ye WebSocket ke through actual message bhejega).

6. **`disconnect()`** — Cleanup method, jab application band ho rahi ho to Redis connection ko safely close karne ke liye.

### Create a simple index.ts file to simulate users

```typescript
import { PubSubManager } from "./PubSubManager";

setInterval(() => {
  PubSubManager.getInstance().userSubscribe(Math.random().toString(), "BTC");
}, 5000)
```

**Ye simulation kya kar raha hai:** Har 5 seconds mein, ek naya random `userId` generate hota hai aur usko "BTC" stock ke liye subscribe kar diya jaata hai — ye simulate karta hai ki naye users continuously BTC mein interest dikha rahe hain. `PubSubManager.getInstance()` use ho raha hai (na ki `new PubSubManager()`), taaki hamesha wahi ek shared singleton instance use ho — isliye saare subscriptions ek hi jagah accumulate honge, chahe ye code kisi bhi file se call ho.

---

## 15. Interview Questions — Poori List with Deeper Answers

### Q1. Stateful aur Stateless backend mein kya difference hai?

> Answer: A stateless backend doesn't hold important data in server memory — it relies on the database for state, so any server can handle any request, and horizontal scaling/autoscaling is easy. A stateful backend keeps some state in memory (like an in-memory cache, game state, or recent chat messages) for performance or real-time needs, which sometimes requires session stickiness so a user's requests always reach the same server.

*Extra tip:* Ye bolna ki dono ke apne trade-offs hain — stateless scaling mein easy hai, stateful performance/real-time needs ke liye zaroori hai. Best systems dono ka mix use karte hain (jaise ek stateless main API, aur ek stateful WebSocket layer real-time features ke liye).

### Q2. Stickiness kab zaroori hoti hai, aur kab nahi?

> Answer: Stickiness is needed when the in-memory state has no fallback — like real-time game state or recent chat messages that only exist on one server. It's not needed for something like an in-memory cache, because a cache miss can just fall back to fetching from the database or original source without breaking correctness.

### Q3. Singleton pattern kya hai, aur ise implement kaise karte hain?

> Answer: Singleton is a design pattern that ensures a class has only one instance throughout the application, with a global access point to that instance. It's implemented using a private constructor (to prevent direct instantiation with `new`) and a static `getInstance()` method that creates the instance only once and returns the same instance on subsequent calls.

*Extra tip:* Agar poocha jaaye "private constructor kyu zaroori hai", bol — "Agar constructor public hota, to koi bhi `new GameManager()` likh ke ek alag instance bana sakta tha, jo poore singleton guarantee ko tod deta — isliye private constructor is enforcement ka core hai."

### Q4. Agar tu private constructor use na kare, sirf ek exported instance bana ke use kare, to kya farak padega?

> Answer: It would mostly work in practice, but it relies on developer discipline — nothing stops someone from writing `new GameManager()` elsewhere and accidentally creating a second, disconnected instance. A true singleton with a private constructor enforces this at the compiler level, making the mistake impossible rather than just discouraged.

### Q5. Pub/Sub pattern kya hai, aur ye kab use karte hain?

> Answer: Pub/Sub (Publish-Subscribe) is a messaging pattern where publishers send events without knowing who will receive them, and subscribers receive events they're interested in without knowing who sent them. It's useful for real-time systems where many clients need to be notified about specific events — like stock price updates, chat messages, or notifications — especially across multiple backend processes/servers.

### Q6. PubSubManager ko Singleton kyu banaya gaya?

> Answer: If each file created its own `PubSubManager` instance, subscriptions would be tracked separately across instances — a user subscribing via one instance wouldn't be visible to the instance handling incoming Redis events. Making it a singleton ensures all subscriptions for that server/process are tracked in one consistent place.

### Q7. `userSubscribe` mein ye check kyu hai — `if (this.subscriptions.get(stock)?.length === 1)`?

> Answer: This checks if this is the first subscriber for that stock. If so, it means the server needs to actually subscribe to that Redis channel for the first time. If there were already subscribers before this one, the Redis subscription is already active, so there's no need to subscribe again — this avoids redundant Redis subscriptions.

---

## 16. Ek line mein pura concept

Isko yaad rakh:

> Server apni memory mein state (jaise subscriptions ya game data) rakhta hai stateful hone ki wajah se; is state ko safe aur consistent tarike se manage karne ke liye Singleton Pattern use hota hai (taaki poori application mein sirf ek hi shared copy ho); aur jab ye state real-time events ke saath multiple users tak pahunchani ho, to Pub/Sub pattern (jaise Redis ke through) use hota hai.

```
              State needed in memory
                      ↓
              Stateful backend
                      ↓
           Needs consistent access
                      ↓
              Singleton Pattern
                      ↓
        One shared instance everywhere
                      ↓
     (For real-time multi-user broadcast)
                      ↓
                  Pub/Sub
                      ↓
        Publisher → Redis → Subscribers
```

---

### Bonus: Quick self-check before interview

Khud se ye sawaal poochh aur bina notes dekhe answer de:

1. Stateless server ka matlab kya hai, aur iske do main advantages kya hain?
2. Stickiness kya hai, aur ye kis scenario mein zaroori hai vs nahi?
3. `new GameManager()` har jagah use karna kyu problematic hai?
4. Singleton pattern kaise implement hota hai — private constructor aur static method ka role kya hai?
5. Pub/Sub pattern kya solve karta hai, aur Publisher/Subscriber roles kaise define hote hain?
6. `PubSubManager` mein Redis subscription sirf "pehle subscriber" par kyu hoti hai, har subscriber par nahi?
7. Tera chat app project mein Singleton aur Pub/Sub kahan use ho sakte hain?

Agar in sabka answer bina ruke de paya, to ye topic solid hai.

---

### Extra: Ye concept tere Chat App aur Retail-Store-Agent projects se kaise connect hota hai

**Chat App:** Tera messaging platform project jismein Socket.IO use ho raha hai, wahan bhi similar problem aa sakti hai — agar multiple WebSocket servers hain, to "kaun konse room/conversation mein connected hai" track karna PubSubManager jaisi hi ek singleton-based approach maang sakta hai, jahan Redis Pub/Sub events ko saare servers ke beech relay kare.

**General principle:** Kisi bhi jagah jahan tu BullMQ, Redis, ya Socket.IO use kar raha hai real-time/background processing ke liye, wahan "shared state jo poori application mein consistent rahe" ki zarurat aa sakti hai — aur Singleton Pattern us consistency ko enforce karne ka ek reliable tareeka hai.