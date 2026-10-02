# 说瞎话游戏 AI 策略说明

本文档结合源代码详细说明《苗伯说瞎话》游戏中 AI 对手的决策逻辑和策略设计。

## 一、游戏核心规则回顾

**基本玩法**：一副牌带大小王，每人 13 张。玩家自选点数报牌（1-4 张），下家可以选择：
- **跟牌**：报同样点数的牌（张数不限，点数可假）
- **过**：不出牌（手里有真牌也可以过）
- **抓**：喊"说瞎话"，质疑上家报的牌

**关键规则**：
- 抓对了：出牌的人收走整桌牌，**抓的人接着出**
- 抓错了：抓的人收走整桌牌，**被抓的人接着出**
- 三家都过：桌上牌作废，最后出牌的人开新一轮（自选点数先出）
- 先出完手牌且最后一手没被抓的人获胜

---

## 二、AI 人物设定

游戏有 7 个人物，每局随机抽 3 个上桌，各有性格参数：

| 人物 | 方言 | 特点 |
|------|------|------|
| **老李** | 天津 | 实在人，牌好就实说；被逮着就上头 |
| **王姐** | 天津 | 爱诈唬爱多报，嘴最不饶人，出手快 |
| **小赵** | 天津 | 话少算得准，记明牌，专挑快出完的人下手 |
| **铁子** | 东北 | 梭哈型选手，大吹特吹，抓人也痛快 |
| **陈教授** | 苏州 | 几乎不诈唬，一抓一个准 |
| **幺妹** | 四川 | 最爱抓人，被抓了也不服 |
| **发叔** | 广东 | 精明，怕背牌堆蚀本，能过就过 |

### 2.1 性格参数（prof 对象）

```javascript
// 示例：老李的参数
laoli: {
  prof: {
    passReal: .22,      // 手里有真牌时的过牌率
    passBluff: .72,     // 手里没真牌（吹牛）时的过牌率
    bigPlay: .50,       // 孤注一掷（全手一次出完）的概率
    truthMax: .90,      // 出真牌时用最大张数的倾向
    junkMix: .10,       // 真牌里掺假牌的概率
    callBase: .20,      // 基础抓牌概率
    callPerCard: .12,   // 每多一张真牌增加的抓牌概率
    crowd: .35,         // 对"这轮吹过头了"的反应系数
    chatty: .55,        // 爱说话程度
    exitD: .75,         // 收官纪律（不用假牌收官）
    guard: .85,         // 最后一道闸门的抓牌责任
    sniper: .0          // 是否专盯快出完的人（小赵特有）
  },
  talk: {
    play: ['就介个','跟一手','得嘞','稳当的'],
    bluff: ['……跟一个吧','就当有','凑合一手'],
    pass: ['让一手','歇歇'],
    call: ['数不对','不可能','你少算一张'],
    caught: ['邪门','崴泥了','打镲呢吧'],
    clear: ['你瞅瞅','别介']
  }
}
```

### 2.2 每局状态浮动

AI 每局的实际表现会在人设基准上抖动 `0.8~1.25` 倍：

```javascript
function jitterProf(base){
  const p = JSON.parse(JSON.stringify(base));
  const J = () => .8 + Math.random() * .45;  // 0.8~1.25 的随机系数
  ['passReal','passBluff','bigPlay','junkMix','callBase','callPerCard','crowd','chatty']
    .forEach(k => { if(typeof p[k] === 'number') p[k] = Math.min(.95, p[k] * J()); });
  p.truthMax = Math.min(.98, Math.max(.4, p.truthMax * J()));
  p.speed = [Math.round(p.speed[0]*J()), Math.round(p.speed[1]*J())].sort((a,b)=>a-b);
  return p;
}
```

**设计意图**：今天的王姐和昨天的不完全一样，避免玩家摸透固定套路。

---

## 三、AI 决策核心流程

### 3.1 决策顺序

```javascript
function aiPlan(p) {
  // 1. 检查是否能抓（上家刚出牌且不是自己）
  if (canCall && hands[lastPlay.player].length === 0)
    return aiChallenge(p, P, hot) ? {act:'call'} : {act:'pass'};
  
  if (canCall && aiChallenge(p, P, hot)) 
    return {act:'call'};
  
  // 2. 检查是否要过牌
  if (currentRank !== null && Math.random() < passProbability)
    return {act:'pass'};
  
  // 3. 选择出什么牌
  return {act:'play', rank: chosenRank, cards: chosenCards, lying: isLying};
}
```

### 3.2 思考时间计算

AI 的出手速度根据牌情动态调整：

```javascript
function ponderMs(p, plan) {
  let t = rnd(P.speed[0], P.speed[1]);  // 基础速度
  
  if (plan.act === 'call') t *= 1.35;   // 抓人时多想一会儿
  if (plan.lying)        t *= 1.55;     // 吹牛时心虚，多磨蹭
  if (hands[p].length <= 3) t *= 1.35;  // 快出完时谨慎
  if (pile.length > 12)  t *= 1.15;     // 牌堆大时犹豫
  if (tilt[p])           t *= 0.60;     // 上头了出手急
  
  return Math.min(3400, Math.round(t));
}
```

---

## 四、核心策略详解

### 4.1 策略一：真着收官是无敌的

**博弈论原理**：最后一手若是真牌，抓它的人自己背牌堆，而你照样赢（翻完牌照样判你出完）。

**代码实现**：
```javascript
// 王是收官的胶水：能让这一手真着出完全手，才舍得掺进去
if (jokers.length) {
  if (truth.length + jokers.length >= hand.length && hand.length <= ECFG.max)
    truth = truth.concat(jokers);  // 用王凑成一次出完的真牌
  else if (!truth.length && hand.length <= 2)
    truth = jokers.slice();        // 剩两张时直接用王收官
}
```

**残局优先清零头**：
```javascript
else if (hand.length <= 7 && sets.length > 1)
  rank = sets[sets.length - 1].r;  // 先出最小的对子，把大套留着真着收官
```

---

### 4.2 策略二：收官手「宁可错抓，不可放过」

**博弈论原理**：有人打出最后一手时：
- 他是真的 → 抓不抓都输
- 他是吹的 → 只有抓才有救

**代码实现**：
```javascript
if (left === 0) {  // 上家打出最后一手
  // 下家过了，抓的权利传给下下家 —— 形成责任链
  const behind = (q - p - 1 + 4) % 4;  // 我后面还剩几道闸
  let prob = behind === 0 ? P.guard : P.guard * (behind === 1 ? .72 : .55);
  
  if (q === roundLeaderId) prob *= .65;      // 点数他自己定的，多半早备好了真牌
  else prob = Math.min(.99, prob * 1.3);     // 被点数逼出来的收官，可疑得很
  
  if (P.counts && (known[q][R] || 0) >= k) prob *= .4;  // 小赵记得他手里真有这个数
  prob += .06 * mine;
  if (hot) prob = Math.min(.99, prob + .1);  // 上头了更爱抓
  
  return Math.random() < prob;
}
```

**推论**：说着瞎话收官等于自首。AI 有「收官纪律」参数（`exitD`），宁可少出、宁可过，也不用假牌收官。

---

### 4.3 策略三：挡在收官手前面的「闸门链」

**博弈论原理**：下家过了，抓的权利传给下下家 —— 于是形成责任链：
- 越靠后的闸门责任越大
- 最后一道闸几乎必须抓（`guard` 参数）
- 前面的可以搭便车

**陷阱**：在收官手后面跟着出牌，等于替他锁死这局（他的牌从此没人能翻）。

**代码实现**：
```javascript
// 有人刚打出最后一手：这时只有抓和过两条路
if (canCall && hands[lastPlay.player].length === 0)
  return aiChallenge(p, P, hot) ? {act:'call'} : {act:'pass'};
```

AI 在这个局面下只会抓或过，绝不出牌。

---

### 4.4 策略四：数牌是唯一的铁证

**博弈论原理**：我攥着 `m` 张这个点数的牌（王也算），他报 `k` 张，`k+m > 6` 就是必吹。

**代码实现**：
```javascript
function aiChallenge(p, P, hot) {
  const cap = 4 + (ECFG.wild ? 2 : 0);  // 一副牌最多 4 张同点数，带王则 6 张
  const R = lastPlay.rank, k = lastPlay.cards.length;
  const mine = hands[p].filter(c => fits(c, R)).length;
  
  if (k + mine > cap) return true;  // 铁证：加上我攥着的，数就放不下了
  
  // 小赵记得翻开过的牌都到了谁手里
  let knownElse = 0;
  if (P.counts) {
    for (let i = 0; i < 4; i++)
      if (i !== p && i !== q)
        knownElse += (known[i][R] || 0) + (ECFG.wild ? (known[i].j || 0) : 0);
    if (k + mine + knownElse > cap) return true;  // 铁证二：明牌都对不上
  }
  // ...
}
```

**界面辅助**：v1.7 起，轮到真人能抓时，界面替真人把这笔账算好（AI 一直在心里算，这才公平）。

---

### 4.5 策略五：先手经济

**博弈论原理**：新一轮的先手几乎全靠抓牌赢来（抓对了你先出）或者烧堆继承（最后出牌的人先出）。先手意味着自选点数、真着顺牌。

**代码实现**：
```javascript
// 再过一手就烧堆，等于白送他一轮先手
if (passStreak === 2 && lastPlay && hands[lastPlay.player].length <= 4)
  pp *= .4;  // 有人要跑，不能歇着

// 快烧堆（连过两家）而最后出牌的人手牌又少时，AI 会主动抓或出
if (passStreak === 2 && left <= 4) prob += .15;  // 再过就烧堆白送他先手
```

**模拟数据**：12 局里，老李抢到 60 个先手、真人机器人只有 31 个 —— 这就是「爱过牌的人赢不了」的结构性原因。

---

### 4.6 策略六：吹牛的性价比

**博弈论原理**：被逮一次平均背 7 张牌。纯吹（手里没真牌硬吹）是自杀。性价比最高的是**掺着吹**：真牌里混一张零头，怀疑窗口最小。

**代码实现**：
```javascript
// 真牌里掺假牌的概率
let mix = hot ? P.junkMix + .30 : P.junkMix;
if (pile.length <= 4) mix += .10;                // 堆小，掺一张假的便宜
if (pile.length > 12) mix *= .5;                 // 堆大，被逮着背不起
if (!lead && roundClaimed + cards.length >= capT) mix *= .25;  // 这点数快报满了，别再掺

if (Math.random() < mix && cards.length < ECFG.max) {
  const junk = hand.filter(c => !cards.includes(c) && !(ECFG.wild && isJoker(c)))
                   .sort((x, y) => by[rk(x)].length - by[rk(y)].length)[0];
  if (junk !== undefined) cards.push(junk);
}
```

**牌堆大小影响**：牌堆越大越要少吹（`junkMix` 随堆衰减），牌堆小的时候掺一张几乎白赚。

---

### 4.7 策略七：对真人的信用系统

**博弈论原理**：AI 记着真人的旧账（跨局）：吹牛被逮率高的多抓、一贯老实的少抓。

**代码实现**：
```javascript
if (q === mySeat && SEATK[q] === 'human') {  // 对本机真人看信用
  // 白纸记录 ≈ 一般人的吹牛率 (~0.35)，别一上来就当嫌疑人
  const c = MEM.humanCaught + humCaughtS,   // 被抓到的次数
        h = (MEM.humanHonest || 0) + humHonestS;  // 老实的次数
  prob *= Math.min(1.35, .5 + 1.4 * ((c + .7) / (c + h + 2)));
}
```

**乘数范围**：封在 `0.5~1.35` 之间。

**给真人的武器**：先攒信用，再在关键一手背刺。

---

## 五、过牌决策逻辑

```javascript
if (currentRank !== null) {  // 要不要过
  const real = hand.filter(c => fits(c, currentRank)).length;
  let pp = real ? P.passReal : P.passBluff;  // 有真牌/没真牌的过牌率
  
  if (hot) pp *= .5;              // 上头了爱出牌
  if (pile.length > 10) pp += .18; // 牌堆大，多歇歇
  if (hand.length <= 4) pp *= .3;  // 快出完了，珍惜每次出牌机会
  if (threat !== undefined) pp *= .6;  // 有人要跑，不能歇着
  
  // 再过一手就烧堆，等于白送他一轮先手
  if (passStreak === 2 && lastPlay && hands[lastPlay.player].length <= 4)
    pp *= .4;
  
  // 没真牌时的硬纪律：这点数还装得下几张
  if (!real) {
    const after1 = roundClaimed + 1;
    if (after1 > capT)        pp = .98;      // 报不下了还吹是自杀
    else if (after1 > capT - 2) pp = Math.max(pp, .88);  // 人家刚甩了一大把，你跟一张谁信
  }
  
  if (Math.random() < pp) return {act: 'pass'};
}
```

---

## 六、出牌选择逻辑

### 6.1 选择点数（rank）

```javascript
const lead = rank === null;
if (lead) {
  const sets = Object.keys(by)
    .map(r => ({r: +r, n: by[r].length}))
    .sort((x, y) => y.n - x.n);  // 按套的长度排序
  
  if (!sets.length)
    rank = jokers.length ? (Math.random() * 13 | 0) : (Math.random() * 13 | 0);
  else if (hand.length <= ECFG.max && sets[0].n + jokers.length >= hand.length)
    rank = sets[0].r;  // 一把梭：全手一次真着出完，稳赢
  else if (hand.length <= 7 && sets.length > 1)
    rank = sets[sets.length - 1].r;  // 残局先清零头小对，把大套留着真着收官
  else
    rank = sets[0].r;  // 平时出最长的套
}
```

### 6.2 选择张数（cards）

```javascript
// 孤注一掷（全手一次出完）
if (hand.length <= ECFG.max && Math.random() < P.bigPlay) {
  cards = hand.slice(0, Math.min(ECFG.max, hand.length));
}
// 出真牌
else if (truth.length) {
  const k = Math.random() < P.truthMax
    ? Math.min(truth.length, ECFG.max)
    : Math.min(truth.length, ECFG.max, 1 + (Math.random() * 3 | 0));
  cards = truth.slice(0, k);
  // ... 掺假牌逻辑
}
// 纯吹牛（手里没真牌）
else {
  const pool = hand.filter(c => !(ECFG.wild && isJoker(c)));
  const sorted = pool.sort((x, y) => by[rk(x)].length - by[rk(y)].length);
  let bn = Math.random() < .7 ? 1 : 2;
  if (pile.length > 10) bn = 1;  // 堆大，瞎话吹小点
  cards = sorted.slice(0, Math.min(sorted.length, bn));
  if (!cards.length) cards = hand.slice(0, 1);
}
```

---

## 七、抓牌决策逻辑（完整版）

```javascript
function aiChallenge(p, P, hot) {
  const cap = 4 + (ECFG.wild ? 2 : 0), R = lastPlay.rank, k = lastPlay.cards.length;
  const q = lastPlay.player;
  const mine = hands[p].filter(c => fits(c, R)).length;
  
  // 铁证：加上我攥着的，数就放不下了
  if (k + mine > cap) return true;
  
  // 小赵记得翻开过的牌都到了谁手里
  let knownElse = 0;
  if (P.counts) {
    for (let i = 0; i < 4; i++)
      if (i !== p && i !== q)
        knownElse += (known[i][R] || 0) + (ECFG.wild ? (known[i].j || 0) : 0);
    if (k + mine + knownElse > cap) return true;  // 铁证二：明牌都对不上
  }
  
  const left = hands[q].length;
  
  // === 上家打出最后一手 ===
  if (left === 0) {
    const behind = (q - p - 1 + 4) % 4;  // 我后面还剩几道闸
    let prob = behind === 0 ? P.guard : P.guard * (behind === 1 ? .72 : .55);
    
    if (q === roundLeaderId) prob *= .65;  // 点数他自己定的，多半早备好了真牌
    else prob = Math.min(.99, prob * 1.3);  // 被点数逼出来的收官，可疑得很
    
    if (P.counts && (known[q][R] || 0) >= k) prob *= .4;  // 小赵记得他手里真有这个数
    prob += .06 * mine;
    if (hot) prob = Math.min(.99, prob + .1);
    
    return Math.random() < prob;
  }
  
  // === 普通情况的抓牌概率 ===
  let prob = P.callBase + mine * P.callPerCard;  // 我攥着越多，他越可疑
  
  const before = Math.max(0, roundClaimed - k),
        room = Math.max(0, cap - mine - knownElse - k);
  if (before > room) prob += P.crowd * Math.min(4, before - room);  // 这轮吹过头了，气氛可疑
  
  // 这一手要是真的，这个点数全场的余量还剩多少
  const slack = Math.max(0, cap - mine - knownElse - k);
  if (k >= 3) {
    if (slack <= 0)      prob += .18;  // 挤到刚好放下：所有真牌都得在他手里，可疑
    else if (slack === 1) prob += .08;
    else                 prob *= .7;   // 宽裕的大把多半是真套，别瞎抓白背牌
  } else {
    prob += .05 * Math.max(0, 2 - slack);
    if (before >= 4) prob += .12;  // 大把之后还跟小的，往往是硬挤，重点盯
  }
  
  if (left <= 2) prob += .15 + (P.sniper ? .10 : 0);  // 快出完的人，重点照顾
  else if (P.sniper && left <= 4) prob += .06;
  
  // 对本机真人看信用
  if (q === mySeat && SEATK[q] === 'human') {
    const c = MEM.humanCaught + humCaughtS,
          h = (MEM.humanHonest || 0) + humHonestS;
    prob *= Math.min(1.35, .5 + 1.4 * ((c + .7) / (c + h + 2)));
  }
  
  if (P.counts && (known[q][R] || 0) >= k) prob *= .5;  // 明知他有真牌，少冤枉
  if (hot) prob += .16;
  
  prob *= Math.max(.55, 1.15 - .028 * pile.length);  // 堆越大，抓错越亏
  if (hands[p].length <= 4) prob *= .7;  // 自己快赢了，背不起一堆牌
  else if (pile.length <= 3) prob *= 1.1;
  
  if (passStreak === 2 && left <= 4) prob += .15;  // 再过就烧堆白送他先手
  
  return Math.random() < prob;
}
```

---

## 八、上头系统（Tilt）

**触发条件**：收了牌堆的人会上头，接下来一两手边乱出牌边骂街。

```javascript
// 上头时牌风变化
if (tilt[p]) {
  // 火气消一格
  tilt[p]--;
  
  // 上头期间出手急
  t *= .60;
  
  // 骂街概率增加
  if (hot && Math.random() < .60) chat(p, 'tilt', 1);
}

// 火气值计算（决定骂街脏度）
const heat = Math.min(.85, (rage[p] || 0) * .3 + Math.max(0, hands[p].length - 13) * .04);
// 没背过牌堆基本干净，背两三堆之后满嘴跑火车；手牌攒多了也焦躁
```

---

## 九、机器人实验结论

四种风格的机器人对着这套 AI 各打 24~48 局：

| 类型 | 胜率 | 输的原因 |
|------|------|----------|
| 纯吹型（55% 纯吹率） | 4% | 12 局被逮 26 次 |
| 老实型 | 0% | 没先手 |
| 掺吹型 | 8% | 送先手 |
| 爱抓型 | 8% | 送先手 |
| 均衡型 | 17% | — |

**结论**：没有单维度的必胜策略 —— 真人能同时做对几件事，天花板比任何一个机器人高。

---

## 十、版本演进（AI 相关）

| 版本 | AI 改进 |
|------|---------|
| v1.7 | 按博弈论重做本地 AI：收官纪律、收官手宁可错抓不可放过、绝不在收官手后面跟牌、残局先清零头留大套、王当收官胶水、快烧堆时不白送先手、小赵记明牌、对真人的信用乘数封顶 |
| v1.12 | 三个人的抓牌欲望整体上调（基础概率、报大把的嫌疑、上头的冲动都加了），实测抓牌频率翻倍 |
| v1.27 | AI 对大把出牌的判断重做：从「出得多就可疑」改成按容量余量算 —— 这个点数全场还装得下他报的数就多半是真套，挤到刚好放下才重点抓 |
| v1.28 | 跟牌纪律再收紧：别人甩大把之后，没真牌的 AI 基本不再跟小张硬吹（硬下限 88% 过牌），抓牌端也重点盯「大把之后的小跟」 |

---

## 十一、总结

这套 AI 系统的核心设计思想：

1. **基于博弈论的硬结论**：每条策略都有数学推导支撑，不是拍脑袋的启发式
2. **性格差异化**：7 个人物有不同的参数组合，形成可识别的打法风格
3. **状态浮动**：每局表现有随机波动，避免被摸透
4. **跨局记忆**：对真人建立信用档案，让玩家的风格被「记住」
5. **公平透明**：AI 的数牌能力在界面上对真人可见（数牌雷达），不靠信息不对称取胜

**目标**：AI 够狠，但赢它靠的是打法，不是运气。
