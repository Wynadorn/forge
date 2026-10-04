# User Prompt
The goal is to **redesign MTG Forge Adventure’s economy so that the different ways of playing, earning gold, buying cards, and entering events form one coherent system**, rather than a collection of independent reward values scattered throughout the code.
 
### Current situation 
 
Right now, we're looking at Forge Adventure as an RPG-like layer around Magic:
 
- You travel around the world.
- You fight duels.
- You complete quests and dungeons.
- You find treasure/rewards.
- You earn gold.
- You spend that gold on boosters, cards, or events.
- Events let you play Limited-style Magic and provide additional rewards.

 ### What we're actually trying to fix 

The problem is that these systems don't necessarily have a common economic model. A reward of 200 gold in one place, 500 in another, card rewards somewhere else, etc. can easily produce an economy where it's difficult to answer basic questions like:
 
> **How much does an hour of playing Adventure actually earn?**
> **How long should it take to afford a draft/sealed event?**
> **Is buying a booster sensible compared with entering an event?**
> **What should a card be worth?**
> **What happens economically if I lose an event?**

 
So we're trying to establish a **deliberate economy first**, and then make the game code implement that economy.
 
***
 
## The basic economic model
 
The model we've been converging on is:
 
**Adventure = your job**
**Gold = your wages**
**Packs/cards = things you spend those wages on**
**Events = an alternative way of spending those wages that also lets you play**
 
The important distinction is that **events aren't supposed to be the job**. Adventure is where you earn your money.
 
We're roughly targeting something like:
 
> **~4,000 gold/hour from Adventure**

The exact number still needs tuning against actual battle duration.
 
That income comes primarily from:
 
- duels
- treasure/found gold
- quest completion
 
rather than handing out valuable random cards as normal income.
 
***
 
## Quests
 
The rough quest model discussed is:
 
**Quest completion: ~2,000g + 1 booster**
 
The booster would ideally correspond to the **current set/event in the town where the quest is started**, rather than being completely random.
 
For example, if a town currently has a particular Limited event running, completing the local quest could give you a booster from that same environment.
 
The remaining ~2,000g/hour would come from actually travelling, fighting, and finding treasure.
 
A typical quest might involve something like:
 
- ~3 fights while travelling
- a dungeon with ~5–9 fights
- perhaps ~7 dungeon fights on average

So a rough quest might involve ~10 battles.
 
The important part is that we **don't double-pay objectives**. If the quest pays you for clearing a particular dungeon, we don't want the dungeon itself independently handing out another huge completion reward.
 
***
 
## Cards vs gold
 
This was one of the more interesting design problems.
 
Originally, giving cards as Adventure loot sounds great:
 
> "I went exploring and found a rare Magic card!"

 
That's very RPG-like.
 
But there's a problem.
 
If cards can be sold, then economically this becomes:
 
> "I went exploring and found 200 gold."

 
Except now the reward is random, because different cards have wildly different values.
 
And that creates another problem: if finding cards is part of the expected income, then the optimal player may simply sell every card they find.
 
So we've arrived at a useful distinction:
 
> **Gold is your wage. Cards are your discoveries.**

 
Normal Adventure rewards should therefore primarily be **predictable gold**.
 
Occasional cards/boosters can still appear as special discoveries, but they shouldn't be necessary to reach the expected ~4,000g/hour.
 
The quest booster is a deliberate exception because it's controlled and tied to the local card environment.
 
***
 
## Packs
 
We're establishing:
 
> **1 booster = 1,000 gold**

 
This becomes one of the fundamental anchors of the entire economy.
 
That means if you simply buy a booster:
 
**1,000g → booster**
 
Nothing else happens.
 
***
 
## Events
 
Then we make events economically comparable to buying packs.
 
For example, a six-pack Sealed event costs:
 
> **6,000g**

 
because you're effectively spending the equivalent of six boosters.
 
But you **keep the cards** from those packs/draft cards.
 
That's important.
 
An event therefore becomes:
 
> **Spend the same money you could have spent acquiring the cards directly, but get to play Magic as well.**

 
The event rewards are therefore not supposed to represent payment for the cards.
 
They're compensation for:
 
> **your time + effort + performance**

 
This produces a particularly nice rule for event economics:
 
### If you play and lose everything
 
You should still come out roughly equivalent to having simply bought the packs.
 
So:
 
**0 wins → cards, but no economic punishment**
 
Whereas:
 
**1 win → cards + additional reward**
 
And then:
 
**2 wins → more reward**
**3 wins → more again**
etc.
 
The precise reward curve still needs to be determined. But a good progression could be:
0 = cards
1 = cards + 500g (half a pack) 
2 = cards + 1 pack of the event 
3 = cards + 1 pack of the event + 500 gold
 
An immediate concession is different: if you enter and instantly forfeit, you've effectively just bought the packs, so there's no reason to give the "I actually played" reward.
 
***
 
## The card market
 
There's another big problem we uncovered.
 
Forge has a single-card economy, but there's no actual player-driven supply/demand market.
 
So if we simply value cards as:
 
> Common = X
> Uncommon = Y
> Rare = Z
> Mythic = W
 
then something like an extremely common Jumpstart rare could potentially have exactly the same value as a highly desirable chase rare.
 
That doesn't make much sense.
 
There are roughly **35,000 MTG cards**, so manually pricing them isn't realistic. It would require a dataset and some form of automation. 
 
The idea is therefore to automatically classify cards into something like:
 
### Bulk
 
Cards with little market demand/value.
 
### Desirable
 
Cards people actually want, but which aren't major chase cards.
 
### Chase
 
Cards with substantial real-world demand/value.
 
We could derive that from **real market data**. 
 
MTGJSON is also useful because it can provide the card/set/booster data we need.
 
We'd probably want more than simply:
 
> "Current lowest Cardmarket listing"

because one weird listing shouldn't turn an obscure card into a chase card.
 
So we can potentially incorporate:
 
- recent/30-day prices
- price percentiles
- number of listings
- liquidity/confidence
- perhaps retail vs buylist pricing

That said, if this data is available in the first place.
 
The goal isn't necessarily to perfectly reproduce Cardmarket inside Forge.
 
It's to use the real market as a **signal for how desirable cards are**.
 
***
 
## Booster economics
 
Once we have card prices, we can answer another important question:
 
> **What happens if I buy a 1,000g booster and immediately sell everything inside it?**

We call this the **liquidation EV**. This value would also determine the "cost" or penalty to essentially reroll a pack.
 
For example, purely illustratively:
 
**Pack price:** 1,000g
**Expected sell-all value:** 500g
 
Then:
 
**Net EV = 500 − 1,000 = −500g**
 
and:
 
**ROI = 500 / 1,000 = 50%**
 
The actual number isn't decided yet.
 
And importantly, we shouldn't calculate this by assuming every card is equally likely.
 
We want to use the **actual booster collation**:
 
- commons
- uncommons
- rare/mythic slot
- foil slots
- special sheets
- set-specific weirdness
- etc.

 
MTGJSON should help provide the underlying booster configuration.
 
Then we can calculate:
 
> **Liquidation EV for every set/booster**

 
and use that to see whether our card buy/sell prices produce a sensible economy.

***
 
## Code - > The economy class
 
This is where we got to architecturally.
 
Rather than fixing all of this by hunting through the Forge code for hundreds of:
 
```Java 
player.Gold += 200;
```
 
we want to establish a central economic boundary.
 
Something conceptually like:
 
```java
IEconomy
```
 
with methods such as:
 
```java
Reward CalculateBattleReward(BattleContext context);

Reward CalculateQuestReward(QuestContext context);

Reward CalculateDungeonReward(DungeonContext context);

Reward CalculateTreasureReward(TreasureContext context);

Reward CalculateEventReward(EventContext context);

int GetCardBuyPrice(Card card);

int GetCardSellPrice(Card card);

int GetBoosterPrice(Booster booster);

double GetBoosterLiquidationEV(Set set);
```
 
The important thing is that the **Adventure system still knows how to run an adventure**.
 
The **Quest system still knows what a quest is**.
 
The **Event system still knows how to run an event**.
 
But they don't independently invent their own economy.
 
They ask the economy:
 
> "This happened. What is the appropriate economic consequence?"

 
***
 
## Make the numbers configurable
 
I'd also put the actual economic parameters into something like:
 
```java
public sealed class EconomyConfig
{
    public int PackPrice { get; init; } = 1000;

    public int QuestGoldReward { get; init; } = 2000;

    public int NormalBattleReward { get; init; }
    public int EliteBattleReward { get; init; }
    public int BossReward { get; init; }

    public bool QuestIncludesBooster { get; init; } = true;
}
```
 
Those numbers are **examples**, not final values.
 
The important thing is that we can change:
 
> 175g → 190g → 210g

 
without hunting through the entire game for every place that happens to award battle gold.
 
***
 
## And make rewards composable
 
Rather than having separate mechanisms for every possible reward, we can have something along the lines of:
 
```java
Reward
    ├── GoldReward
    ├── CardReward
    ├── BoosterReward
    └── ItemReward
```
 
Then a quest could simply produce:
 
```text
Reward
 ├─ 2,000 gold
 └─ 1 booster from the town's event set
```
 
while an event could produce:
 
```text
Reward
 └─ 1,500 gold
```
 
or whatever the eventual curve dictates.
 
That also makes it much easier to test.
 
***
 
## Why centralizing it matters
 
Once everything goes through the Economy layer, we can actually **simulate the entire economy**.
 
For example:
 
```text
Adventure session
    ↓
10 battles
    ↓
treasure
    ↓
quest
    ↓
~4,000g/hour
    ↓
player saves
    ↓
6,000g
    ↓
enters Sealed
    ↓
0 wins → cards, no extra reward
1 win  → cards + X
2 wins → cards + Y
...
```
 
And separately:
 
```text
1,000g
   ↓
booster
   ↓
actual MTG collation
   ↓
card market prices
   ↓
sell everything
   ↓
liquidation EV
```
 
We can then ask quantitative questions like:
 
- How many hours of Adventure for a Sealed event?
- How many hours for a particular deck?
- How much gold enters the economy per hour?
- How much gold leaves it?
- What's the average value of opening a pack?
- Are events economically attractive without becoming an infinite-money strategy?
- How quickly does a new character acquire cards?
- How much does luck affect progression?
- Does selling cards destroy the economy?
- Are chase cards actually meaningfully more valuable than bulk?
- Does an event reward compensate appropriately for actually playing?

 
That's the real objective.
 
We're essentially trying to turn Forge Adventure's collection of **gold rewards + card rewards + shops + packs + events + quests + dungeons** into one coherent economic simulation that we can tune rather than a pile of unrelated numbers.
 
And the nice part is that **once that Economy boundary exists, we can experiment quite aggressively without rewriting the rest of Adventure.**

# Plan
Centralized Adventure Economy

**TL;DR:** Introduce a new `forge.adventure.economy` package (`IEconomy`/`DefaultEconomy`/`EconomyConfig`) in forge-gui-mobile that becomes the single source of truth for gold/card/booster/event math. Existing reward-generation call sites (battle wins, quests, treasure, shops, events) get rewired to ask this layer for a `Reward`/`Array<Reward>` instead of inventing their own numbers. Card-market classification (bulk/desirable/chase from real prices) is explicitly deferred — the pricing hooks are built now using the existing rarity-based price table as a placeholder so they can be swapped later without touching callers.

**Steps**

**Phase A — Core architecture** (foundation for everything else)
1. New package `forge-gui-mobile/src/forge/adventure/economy/`: `EconomyConfig`, `IEconomy`, `DefaultEconomy`, `Economy` singleton accessor, plus `BattleContext`/`QuestContext`/`EventContext`/`TreasureContext`.
2. Port `CardUtil.java`'s `getCardPrice`/`getBoosterPrice` bodies into `DefaultEconomy`; `CardUtil` becomes a thin delegator (no call-site churn).
3. Add a default `economy.json` + loader (mirrors existing `Config`/`ConfigData` pattern).
4. Add TestNG + `src/test` to forge-gui-mobile (doesn't exist today).

**Phase B — Battle rewards** (*depends on A*)
5. Derive Normal/Elite/Boss tier from `EnemyData.boss`/`difficulty` (threshold ≈0.5, based on observed data).
6. `EnemySprite.java` / `MapStage.java` `setWinner()`: strip literal `"gold"` RewardData entries from the ~150 `enemies.json` blocks at runtime, append `Economy.calculateBattleReward()` instead. Card/item/shard entries stay as "discoveries."

**Phase C — Quest rewards** (*depends on A, parallel with B*)
7. Resolve the quest-giver POI's active event/set via `AdventureEventController`.
8. Quest completion grants flat 2000g + 1 set-matched booster via `Economy.calculateQuestReward()`, replacing literal JSON gold.

**Phase D — Treasure** (*depends on A, parallel*)
9. Chest pickup in `RewardSprite`/`MapStage` uses `Economy.calculateTreasureReward()`.

**Phase E — Card buy/sell wiring** (*depends on A, parallel*)
10. `AdventurePlayer.cardSellPrice()` and shop buy flow call `Economy.getCardSellPrice()`/`getCardBuyPrice()`/`getBoosterPrice()`.

**Phase F — Events** (*depends on A, builds on C's POI/event resolution*)
11. Verify `EventStatus.Abandoned` is only reachable pre-match (true "immediate concession").
12. Entry fee = `packPrice × numPacks` via `Economy.getEventEntryFee()`, replacing hardcoded per-format constants in `AdventureEventRules`.
13. Add a gold layer to Draft/Sealed brackets (today only Jumpstart has one) implementing 0 / +500g / +1 event pack / +1 event pack+500g, merged into `AdventureEventData.giveRewards()`.

**Phase G — Liquidation EV + tests** (*depends on A*)
14. `getBoosterLiquidationEV(setCode)` via repeated `BoosterGenerator.getBoosterPack()` runs priced with the placeholder rarity table.
15. TestNG coverage: battle tiers, quest reward shape, event win-curve (incl. concession = 0 bonus), price delegation, EV sanity (0 < EV < packPrice).

**Relevant files** — see full list with method references in [plan.md](c:/Program Files (vm)/vscode/data/user-data/User/workspaceStorage/fbe3addef5fe401de8d59f4716a36330/GitHub.copilot-chat/chat-session-resources — actually stored at `/memories/session/plan.md`.

**Verification**
1. `mvn -pl forge-gui-mobile test` for the new TestNG suite.
2. Manual: fight normal/elite/boss enemies and check gold tiers; complete a quest for 2000g+booster; abandon a Sealed event immediately (no bonus) vs. win 1/2/3 matches (curve matches).
3. Spot-check liquidation EV against a hand-computed value for one small set.

**Decisions**
- Reuse existing `Reward`/`Array<Reward>` model instead of inventing a new composable hierarchy — it's already Card/Gold/Item/Life/Shards/CardPack.
- No separate `CalculateDungeonReward` — no such hook exists today; folding into quest/battle avoids double-paying.
- Card-market classification (MTGJSON-driven bulk/desirable/chase) is explicitly **out of scope** — separate follow-up; liquidation EV uses rarity-based pricing as a stand-in.
- Battle gold overrides JSON immediately (per your choice) rather than a gradual opt-in flag.

**Further Considerations**
1. Need to confirm exact per-adventure override convention for `economy.json` against `AdventureOverrides`/`Config` during Phase A.
2. Need to verify in Phase F whether `Abandoned` status is truly pre-match-only in `EventScene`, or if a mid-event abandon could currently slip through as a "win-curve" loss.

## Decisions locked in with user
- Card-market (bulk/desirable/chase) classification from MTGJSON prices is a SEPARATE follow-up task.
  This plan only builds the `IEconomy` pricing hooks (getCardBuyPrice/getCardSellPrice/getBoosterPrice/
  getBoosterLiquidationEV) using the EXISTING rarity-based pricing as the stand-in value function, so the
  real classifier can be swapped in later without touching call sites.
- Brief's example numbers become real EconomyConfig defaults now (4000g/hr target composition, 1000g/pack,
  quest=2000g+booster, event win-curve 0/500g/1 event pack/1 event pack+500g).
- Battle gold: ignore/override json values in code immediately. Derive tier (Normal/Elite/Boss) from EnemyData's existing
  `boss` flag and `difficulty` float (threshold ~0.5 for Elite, based on observed distribution: most mobs
  0.1-0.3, elites ~0.8, true bosses have boss=true). Literal "gold" RewardData entries on enemies are ignored
  at runtime (not hand-edited out of enemies.json's ~150 entries); card/item/shards entries in JSON will need to be addressed separately.
- Quest booster set: quests are mostly started inside a POI, and that POI is guaranteed to have an active
  event (per user). So resolve booster set via AdventureEventController's existing per-POI event/cardBlock
  for that quest-giver POI.
- Add TestNG + src/test to forge-gui-mobile for the new Economy logic (none exists there today).

## Research findings (file:method references)
- Reward model: forge-gui-mobile/src/forge/adventure/util/Reward.java (Type: Card/Gold/Item/Life/Shards/CardPack) -
  already composable, KEEP as the output currency instead of inventing a parallel hierarchy.
- Reward data/rolling: forge-gui-mobile/src/forge/adventure/data/RewardData.java `generate(...)` - data-driven
  gold/card/item rolling from JSON, used everywhere (enemies, quests, events, shops).
- Battle win flow: forge-gui-mobile/src/forge/adventure/stage/MapStage.java `setWinner()` -> `getReward()` ->
  forge-gui-mobile/src/forge/adventure/character/EnemySprite.java `getRewards()` (loops EnemyData.rewards[] +
  calls RewardData.generate). EnemyData (data/EnemyData.java) has `boss` boolean, `difficulty` float.
- Quest rewards: data/AdventureQuestData.java (`reward`, `rewardDescription`, `getTargetPOI()`), granted via
  grantRewards[] through scene/MenuScene.java (~L97-102) -> RewardScene. Quest's POI accessible via
  `AdventureQuestData.getTargetPOI().getID()`.
- Treasure: character/RewardSprite.java (default fallback 10-110 gold), picked up in MapStage.java (~L1140).
- Gold storage/mutation: player/AdventurePlayer.java - `gold` field, `getGold()`, `addGold(int)` (private,
  L1012), `giveGold(int)`/`takeGold(int)` (L1108/1112), `win()` (L1098, only adds shard today), `defeated()`
  (gold -= gold*difficultyData.goldLoss), `cardSellPrice(PaperCard)` (L1272), `doBulkSell` (L1307),
  `addReward(Reward)` (L975).
- Difficulty modifiers: data/DifficultyData.java - startingMoney=10, sellFactor=0.2f, goldLoss=0.2f,
  rewardMaxFactor=1f, shardSellRatio=0.8f. These stay as difficulty-scaling INPUTS to Economy, not duplicated.
- Card/booster pricing (to be centralized): util/CardUtil.java - `getCardPrice(PaperCard)` (L379, rarity
  table: BasicLand 5/Common 50/Uncommon 150/Rare 300/MythicRare 500/default 600, or custom price list),
  `getBoosterPrice(Deck)` (L409, fixed 1000 fallback or custom list by edition code), `getRewardPrice(Reward)`
  (L426, has a `// TODO: Heitor - Price by card count and type of boosterPack.` at L440).
  Custom price list: util/AdventureReadPriceList.java (reads world/cardprices.txt, optional fluctuation).
- Card sell: player/AdventurePlayer.java `cardSellPrice()` = getCardPrice*sellFactor, +20% foil, scaled by
  town price modifier `(2.0f - townPriceModifier)`.
- Shop pricing/reputation: character/ShopActor.java, data/ShopData.java,
  pointofintrest/PointOfInterestChanges.java (`getTownPriceModifier()`, `getShopPriceModifier()`, ±20 rep,
  0.5%/rep).
- Booster collation (forge-core, reusable for liquidation EV): item/SealedTemplate.java (slot list),
  item/generation/BoosterSlots.java (slot name constants), item/generation/BoosterGenerator.java
  `getBoosterPack(SealedTemplate)` (the actual collation engine incl. foil logic), item/BoosterSlot.java.
  Adventure-specific overrides: util/AdventureOverrides.java `getBoosterTemplate(setCode)`.
- Rarity: forge-core card/CardRarity.java enum; PaperCard.getRarity().
- No MTGJSON / real price infra anywhere in Java runtime (only Python dev tools: forge-gui/tools/
  editionCreator.py, EditionTracking.py). Confirmed greenfield for later classifier work.
- Events: data/AdventureEventData.java - `AdventureEventRules` ctor sets goldToEnter per format (current code:
  Draft=3000/50shards, Sealed=6000/100shards, Jumpstart=200/5shards, Constructed=1500/25shards - Sealed
  already = 6*1000 matching brief's "6 packs = 6000g" anchor). Per-bracket rewards built in
  `setupDraftRewards()`/`setupSealedRewards()`/`setupJumpstartRewards()` (~L397-495) populating
  `AdventureEventReward[] rewards` (minWins/maxWins bracket array, exactly one bracket applies - NOT
  cumulative). Finalized/granted in `giveRewards()` (~L734-800+): iterates rewards[], converts cardRewards/
  itemRewards/rewards(RewardData) into `Array<Reward>` via RewardData.generate. Only Jumpstart currently has
  a gold layer (100/200/500 per win bracket) - Draft/Sealed have NO gold layer today, need one added per
  brief's curve (0g / +500g / +1 event pack / +1 event pack+500g).
  Entry fee charged in scene/EventScene.java (~L119-124) via DialogData.ActionData -> MenuScene giveGold/
  takeGold. Town price modifier applies to entry fee too.
  Event<->POI link: util/AdventureEventController.java `createEvent(String pointID)`,
  `nextEventDate` keyed by pointID, `getEventSeed(pointID)` (deterministic per POI per day).
  EventStatus enum incl. Abandoned (need to verify in impl phase that Abandoned = pre-match abandon only,
  matching "immediate concession" semantics - currently unverified, flagged as Phase F task).
- No separate "dungeon complete" reward exists distinct from per-battle/treasure rewards (confirms the
  "don't double-pay" requirement is structurally already true - just need tier tuning, not new hook removal).

## Architecture

New package `forge.adventure.economy` in forge-gui-mobile/src/forge/adventure/economy/:
- `EconomyConfig.java` - POJO loaded like other adventure data (reuse util/Config.java /
  data/ConfigData.java loading pattern); one shipped default file (e.g.
  forge-gui/res/adventure/common/economy.json) with per-adventure override capability via existing
  override mechanism (mirror AdventureOverrides pattern). Fields (defaults from brief):
  - packPrice = 1000
  - questGoldReward = 2000, questIncludesBooster = true
  - normalBattleGoldMin/Max = 100/150, eliteBattleGoldMin/Max = 200/300, bossBattleGoldMin/Max = 400/600
  - eliteDifficultyThreshold = 0.5f (difficulty >= this => Elite tier, unless boss==true => Boss)
  - treasureGoldMin/Max = 50/150
  - targetGoldPerHour = 4000 (documentation/simulation constant, not directly consumed by a single code path)
  - eventWinGold[0..3] = {0, 500, 0, 500} and eventWinExtraPacks[0..3] = {0, 0, 1, 1} (bracket r2/r3 get
    +1 event-set booster, r1/r3 get +500g) - exact mapping finalized in Phase F against existing
    minWins/maxWins bracket objects already in AdventureEventData.
  - immediateConcessionForfeitsPlayBonus = true
- `IEconomy.java` interface:
  - `Array<Reward> calculateBattleReward(BattleContext ctx)`
  - `Array<Reward> calculateQuestReward(QuestContext ctx)`
  - `Array<Reward> calculateTreasureReward(TreasureContext ctx)`
  - `Array<Reward> calculateEventReward(EventContext ctx)`
  - `int getEventEntryFee(EventContext ctx)`
  - `int getCardBuyPrice(PaperCard card)` / `int getCardSellPrice(PaperCard card)`
  - `int getBoosterPrice(String setCode)`
  - `double getBoosterLiquidationEV(String setCode)` (simulates N BoosterGenerator.getBoosterPack() runs,
    sums getCardSellPrice-equivalent per card - uses existing rarity pricing as stand-in value function until
    real classifier lands)
  - Note: no separate CalculateDungeonReward - intentionally folded into Quest/Battle per "don't double-pay".
- `DefaultEconomy.java implements IEconomy` - config-driven implementation; internal pure-Java helper methods
  (plain ints, no gdx types) for testability; gdx `Array<Reward>` only at the public API boundary.
- Context value objects: `BattleContext` (EnemyData tier info), `QuestContext` (town/POI id for booster set),
  `EventContext` (format, wins, numPacks, isImmediateConcession), `TreasureContext` (optional biome/tier).
- `Economy` static accessor (singleton, e.g. `Economy.instance()`) analogous to existing `Current`/`Config`
  singletons, so call sites don't need DI plumbing changes.
- `CardUtil.getCardPrice/getBoosterPrice/getRewardPrice` become thin delegators to `Economy.instance()` so
  existing call sites keep working without individual edits.

## Steps (phases)

**Phase A - Core architecture (foundation, blocks B-F)**
1. Create `forge.adventure.economy` package: `EconomyConfig`, `IEconomy`, `DefaultEconomy`, `Economy` accessor,
   context classes.
2. Port `CardUtil.getCardPrice`/`getBoosterPrice` bodies into `DefaultEconomy`; make `CardUtil` delegate.
3. Add economy.json default config file + loading (mirror `Config.java` pattern).
4. Set up forge-gui-mobile `src/test` + TestNG dependency in forge-gui-mobile/pom.xml (mirror forge-game/
   forge-gui pom testng setup).

**Phase B - Battle rewards** (*depends on A*)
5. Add tier derivation (Normal/Elite/Boss) from `EnemyData.boss`/`difficulty` in `BattleContext` construction.
6. In `EnemySprite.getRewards()` / `MapStage.setWinner()` flow: call `Economy.calculateBattleReward(ctx)`,
   merge with non-gold entries from `EnemyData.rewards[]` (filter out `type=="gold"` at the point rewards are
   generated), append Economy's gold Reward.

**Phase C - Quest rewards** (*depends on A, parallel with B*)
7. Resolve quest-giver POI's current event/cardBlock via `AdventureEventController` for booster set matching.
8. At quest completion (AdventureQuestController / grantRewards flow), replace literal gold RewardData with
   `Economy.calculateQuestReward(ctx)` producing 2000g + 1 set-matched booster; keep any non-gold
   grantRewards entries (narrative item rewards) untouched.

**Phase D - Treasure** (*depends on A, parallel with B/C*)
9. `RewardSprite`/`MapStage` treasure pickup gold path calls `Economy.calculateTreasureReward(ctx)` instead of
   the hardcoded 10-110 default.

**Phase E - Card sell/buy wiring** (*depends on A, parallel*)
10. `AdventurePlayer.cardSellPrice()` calls `Economy.getCardSellPrice()`; shop buy flow
    (`CardUtil.getRewardPrice`/BuyButton) calls `Economy.getCardBuyPrice()`/`getBoosterPrice()`.

**Phase F - Events** (*depends on A, benefits from C's POI/event-set resolution code*)
11. Verify `EventStatus.Abandoned` trigger point in EventScene (pre-match only) to confirm it already matches
    "immediate concession = no play bonus"; adjust if abandon is currently allowed mid-event.
12. `EventScene`/`AdventureEventData` entry fee uses `Economy.getEventEntryFee(ctx)` = `packPrice * numPacks`
    (replacing the per-format `baseGoldEntry` constants in `AdventureEventRules`).
13. Add gold layer to Draft/Sealed reward brackets (today only Jumpstart has one) via
    `Economy.calculateEventReward(ctx)` merged into `AdventureEventData.giveRewards()`'s per-bracket
    RewardData, implementing the 0/+500g/+1 pack/+1 pack+500g curve without double-counting the
    already-kept sealed pool/draft deck.

**Phase G - Liquidation EV + simulation tests** (*depends on A; can start early for pure-logic parts*)
14. Implement `getBoosterLiquidationEV(setCode)` using `BoosterGenerator.getBoosterPack()` + existing rarity
    pricing.
15. TestNG tests: battle reward tier bounds, quest reward composition, event win-curve bracket mapping
    (including immediate-concession = 0 bonus), card buy/sell price delegation, liquidation EV sanity
    (EV < packPrice, EV > 0).

## Relevant files
- `forge-gui-mobile/src/forge/adventure/economy/` (new) - EconomyConfig, IEconomy, DefaultEconomy, Economy,
  BattleContext, QuestContext, EventContext, TreasureContext
- `forge-gui-mobile/src/forge/adventure/util/CardUtil.java` - delegate getCardPrice/getBoosterPrice/getRewardPrice
- `forge-gui-mobile/src/forge/adventure/character/EnemySprite.java` - getRewards() battle gold filter+merge
- `forge-gui-mobile/src/forge/adventure/stage/MapStage.java` - setWinner()/getReward(), treasure pickup
- `forge-gui-mobile/src/forge/adventure/character/RewardSprite.java` - treasure default reward
- `forge-gui-mobile/src/forge/adventure/data/EnemyData.java` - tier source (boss, difficulty)
- `forge-gui-mobile/src/forge/adventure/data/AdventureQuestData.java` - getTargetPOI(), quest completion
- `forge-gui-mobile/src/forge/adventure/util/AdventureQuestController.java` - quest completion flow
- `forge-gui-mobile/src/forge/adventure/scene/MenuScene.java` - grantRewards[] handling (~L97-102)
- `forge-gui-mobile/src/forge/adventure/player/AdventurePlayer.java` - cardSellPrice(), addReward(), win()
- `forge-gui-mobile/src/forge/adventure/util/AdventureEventController.java` - per-POI event/cardBlock resolution
- `forge-gui-mobile/src/forge/adventure/data/AdventureEventData.java` - AdventureEventRules entry fee,
  setupDraftRewards/setupSealedRewards/setupJumpstartRewards, giveRewards()
- `forge-gui-mobile/src/forge/adventure/scene/EventScene.java` - entry fee deduction, Abandoned status
- `forge-core/src/main/java/forge/item/generation/BoosterGenerator.java` - liquidation EV simulation engine
- `forge-gui-mobile/pom.xml` - add TestNG dependency + test source dir

## Verification
1. New TestNG suite (Phase G) run via `mvn -pl forge-gui-mobile test` covering reward tiers, event curve,
   liquidation EV sanity.
2. Manual in-game checks: fight a normal/elite/boss enemy and confirm gold matches configured tier ranges;
   complete a quest and confirm 2000g + set-matched booster; enter and immediately abandon a Sealed event and
   confirm no play-bonus; win 1/2/3 matches in a Sealed event and confirm bracket gold/packs match curve.
3. Spot-check `Economy.getBoosterLiquidationEV()` output against hand-computed EV for a known small set.

## Decisions
- Reward model reused (not replaced) - `forge.adventure.util.Reward`/`Array<Reward>` already composable.
- No `CalculateDungeonReward` - folded into Quest/Battle to avoid double-paying objectives.
- Card-market classification (bulk/desirable/chase from MTGJSON) explicitly OUT of scope - separate follow-up
  task; liquidation EV uses existing rarity-based pricing as placeholder value function.
- Battle gold tier thresholds (Elite >= 0.5 difficulty) are a starting heuristic against observed
  enemies.json data distribution, tunable via EconomyConfig.

## Further Considerations
1. Exact per-adventure override mechanism for economy.json (global file vs per-adventure merge) needs a look
   at `AdventureOverrides`/`Config` during Phase A to match existing conventions exactly.
2. EventScene Abandoned-status trigger point needs verification in Phase F (is abandon only available
   pre-match, or can a player abandon mid-event and still dodge the "played" bonus incorrectly).
