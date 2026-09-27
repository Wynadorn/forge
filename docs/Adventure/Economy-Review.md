# Adventure Mode Economy Review (WIP)

_Working notes for rebalancing rewards, event payouts, and shop pricing. Not a finalized design doc — numbers below are current-state defaults gathered from code/config for analysis._

## 1. Event Entry Fees

Entry to an event is paid via **one of three alternative methods** (player picks one, not combined):
- Gold (`eventRules.goldToEnter`)
- Mana Shards (`eventRules.shardsToEnter`)
- A Challenge Coin (Bronze/Silver/Gold, depending on event format) — intended as the **new-player / no-cost-yet** entry path, consumed as an item, no gold/shards charged.

Source: [EventScene.java](/forge-gui-mobile/src/forge/adventure/scene/EventScene.java#L64-L110) (dialog options `enterWithGold` / `enterWithShards` / `enterWithCoin`, mutually exclusive per condition).

Default base costs (before `townPriceModifier` scaling), from `AdventureEventData` event rule setup (~[AdventureEventData.java](/forge-gui-mobile/src/forge/adventure/data/AdventureEventData.java#L1040-L1084)):

| Format | Gold | Shards (original) | Shards (fixed, see §12) | Accepted Coin |
|---|---|---|---|---|
| Constructed | 1500 | 25 | 75 | Silver Challenge Coin |
| Draft | 3000 | 50 | 150 | Gold Challenge Coin |
| Sealed | 6000 | 100 | 300 | Silver Challenge Coin |
| Jumpstart | 200 | 5 | 10 | Bronze Challenge Coin |

Final cost = `base * townPriceModifier` (town prestige/discount factor), gold and shards scale, but **coin entry does not scale** (flat 1 coin regardless of town).

**Update:** the original shard costs implied a 60g/shard (40g/shard for Jumpstart) exchange rate, which did not match the actual gold↔shard market rate available at the Shard Trader (see §12). Fixed by deriving `shardsToEnter` from `goldToEnter` using the trader's buy rate.

Open question: since coin-entry bypasses gold/shard cost entirely, and event card rewards are unaffected by payment method, entering via coin is effectively "free" access to the card reward — need to check whether coins are meant to be rare/hard to acquire, otherwise this could be the actual imbalance rather than the "always award cards" setting.

## 2. Event Card Rewards

- Draft/Sealed: reward tiers keyed on match wins (0/1-3/2-3/3). Booster packs on lower tiers, Challenge Coin on 2+ wins, full drafted deck / sealed pool historically only on 3-0.
- New setting `ADV_EVENTS_ALWAYS_AWARD_CARDS` (default true) — when enabled, the drafted deck / sealed pool is awarded regardless of win count (see [AdventureEventData.java](/forge-gui-mobile/src/forge/adventure/data/AdventureEventData.java#L734-L780)).
- Jumpstart already always awards its packs, but marks them `isNoSell = true`. **Draft/Sealed reward decks are NOT marked no-sell** — meaning guaranteed cards can be immediately liquidated for gold, unlike Jumpstart. This is an inconsistency worth resolving.

## 3. Booster / Pack Composition

Standard booster = 15 cards ([SealedTemplate.java](/forge-core/src/main/java/forge/item/SealedTemplate.java#L17-L23)): 10 common, 3 uncommon, 1 rare/mythic, 1 basic land.

- Draft: 3 boosters ≈ 45 cards drafted (kept ~40 in deck + extras)
- Sealed: 6 boosters ≈ 90 cards

## 4. Card Sell/Buy Prices

Base buy prices by rarity ([CardUtil.java](/forge-gui-mobile/src/forge/adventure/util/CardUtil.java#L399-L408)):

| Rarity | Buy price |
|---|---|
| Basic Land | 5 |
| Common | 50 |
| Uncommon | 150 |
| Rare | 300 |
| Mythic | 500 |
| default/other | 600 |

Sell price = `buyPrice * sellFactor` (+20% if foil), then `* (2.0 - townPriceModifier)`.

`sellFactor` by difficulty ([DifficultyData](/forge-gui-mobile/src/forge/adventure/data/DifficultyData.java), config.json defaults):

| Difficulty | Start Gold | Start Shards | rewardMaxFactor | sellFactor | Gold Loss on Defeat | Life Loss on Defeat |
|---|---|---|---|---|---|---|
| Easy | 500 | 5 | 1.5x | 0.6 | 2% | 10% |
| Normal | 250 | 2 | 1.0x | 0.5 | 10% | 20% |
| Hard | 125 | 0 | 0.5x | 0.25 | 30% | 30% |
| Insane | 0 | 0 | 0.0x | 0.05 | 50% | 30% |

Sell values at Normal (sellFactor 0.5): Common 25, Uncommon 75, Rare 150, Mythic 250.

Approx. sell value of one full booster (Normal): `10*25 + 3*75 + 1*~160(avg rare/mythic) + 1*2.5(land) ≈ 640 gold`.

Booster/item purchase price: 1000 gold default (both).

## 5. Random Duel/Encounter Rewards

Defined per-enemy in adventure world JSON (e.g. [Innistrad enemies.json](/forge-gui/res/adventure/Innistrad/world/enemies.json#L40-L46)). Reward entries use `RewardData` (`type: gold`, `probability`, `count`, `addMaxCount`).

Samples (early game, difficulty ~0.1):
- Avacynian Preacher: 10 gold, 30% chance, up to +80 bonus
- Bog Hermit: 10 gold, 30% chance, up to +40 bonus
- Borderland Ranger: 40 gold, 70% chance, up to +50 bonus

Mid/late-game enemies trend 50-150+ gold per win. Bonus range scales via `addMaxCount * rewardMaxFactor` (difficulty-dependent).

## 6. Draft/Sealed EV Check (Normal difficulty, town modifier = 1.0, gold-paid entry)

- Draft: entry 3000 gold + 50 shards → ~45 cards ≈ 1920 gold sell value → net **-1080 gold, -50 shards** even with guaranteed cards on every record.
- Sealed: entry 6000 gold → ~90 cards ≈ 3840 gold sell value → net **-2160 gold** even with guaranteed cards.
- Coin-paid entry: same card payout, **zero gold/shard cost** — the real "cheap card injection" path, not the gold-paid path.

At Normal, gold-paid entries are not exploitable even with always-award-cards on. Coin-paid entries deserve scrutiny (see open question in section 1) — if coins are easy to farm, that's the actual leak, independent of the always-award-cards setting.

## 8. Verification: Entry Fee vs. Pack Value (Draft/Sealed)

Pack counts come from `CardBlock.cntBoostersDraft` / `cntBoostersSealed`, defined per-block in [blocks.txt](/forge-gui/res/blockdata/blocks.txt) as `Name, draftCount/sealedCount/landSet, sets`. 146 of 152 blocks use the standard `3/6` ratio, matching the flat entry fees (Draft 3000g = 3 packs @ 1000g, Sealed 6000g = 6 packs @ 1000g) — confirms entry fee == shop price of the packs you keep, with event participation (coins/bonus packs on higher win counts) as the added-value bonus for the time investment.

6 blocks use a `5/9` ratio instead (Arabian Nights, Antiquities, The Dark, Fallen Empires, Homelands, Portal Three Kingdoms). Initially flagged as a possible imbalance (more packs for the same flat fee), but **verified this is compensated by smaller pack sizes** in those old sets' edition files:

| Set | Booster contents | Card count |
|---|---|---|
| Arabian Nights | 6 Common, 2 UncommonRare | 8 |
| Antiquities | 6 Common, 2 UncommonRare | 8 |
| The Dark | 6 Common, 2 UncommonRare | 8 |
| Fallen Empires | 5 Common, 2 Uncommon, 1 Rare | 8 |
| Homelands | 6 Common, 2 UncommonRare | 8 |
| Portal Three Kingdoms | 5 Common, 2 Uncommon, 1 Rare, 2 BasicLand | 10 |

vs. standard booster (15 cards: 10 Common, 3 Uncommon, 1 Rare/Mythic, 1 Basic Land, [SealedTemplate.java](/forge-core/src/main/java/forge/item/SealedTemplate.java#L17-L23)).

Rough per-pack sell value comparison (Normal difficulty, sellFactor 0.5): standard pack ≈ 640g; an 8-card old-set pack (6 commons + 2 uncommon/rare) ≈ `6*25 + 2*~115avg ≈ 380g`. So Draft with 5 old-set packs ≈ 1900g total (close to the 3-pack standard-set draft's ~1920g), and Sealed with 9 old-set packs ≈ 3420g (below the 6-pack standard-set sealed's ~3840g). **Conclusion: not an exploit — the 5/9 ratio is intentionally compensating for smaller/cheaper packs, and the flat entry fee remains reasonably fair across both pack styles.**

## 10. Verification: `rewardDeck` = Full Opened Pool, Not the Built Deck

Checked whether the Sealed/Draft `rewardDeck` reward is only the trimmed match deck (~40 cards) or the entirety of what was opened:

- **Sealed**: [`generateSealedPool()`](/forge-gui-mobile/src/forge/adventure/data/AdventureEventData.java#L316-L354) opens every booster in `packConfiguration` into `humanPool`, then builds `rewardDeck` from all of it (`rewardDeck.getOrCreate(DeckSection.Main).addAll(humanPool)`). `registeredDeck` (used for matches) holds the same pool in its Sideboard, separately from `rewardDeck` — pruning/building your match deck never mutates `rewardDeck`. Confirmed: full pool (all packs) is awarded, not just the built deck.
- **Draft**: [`AdventureDeckEditor.completeDraft()`](/forge-gui-mobile/src/forge/adventure/scene/AdventureDeckEditor.java#L687) copies `rewardDeck` from `registeredDeck` at the moment picking ends (`registeredDeck.copyTo("Draft Deck")`), which happens before any deck-building/pruning UI. Confirmed: all drafted cards are awarded, not the pruned match deck.

Both match the intended design: pay entry fee ≈ shop price of packs opened, keep everything from those packs, event performance (coins, bonus packs) is the extra reward for time invested.

## 12. Shard/Gold Conversion Rate Fix

The Shard Trader ([ShardTraderScene.java](/forge-gui-mobile/src/forge/adventure/scene/ShardTraderScene.java)) is an unrestricted town building where any player can freely convert gold ↔ shards:

- **Buy**: 100 gold → 5 shards, flat, no difficulty scaling → **20 gold/shard**.
- **Sell**: 100 * `shardSellRatio` gold for 5 shards → 16g/shard (Normal, ratio 0.8), scaling 19/16/12/6 g-per-shard across Easy/Normal/Hard/Insane.

Original event entry fees implied a much higher shard value than the trader's buy rate:

| Format | Gold entry | Shard entry (orig.) | Implied gold/shard | Trader buy rate | Gap |
|---|---|---|---|---|---|
| Constructed | 1500 | 25 | 60g | 20g | 3x |
| Draft | 3000 | 50 | 60g | 20g | 3x |
| Sealed | 6000 | 100 | 60g | 20g | 3x |
| Jumpstart | 200 | 5 | 40g | 20g | 2x |

**This was an arbitrage exploit**: a player could buy 50 shards for 1000 gold at the trader, then enter a Draft event "for shards" that would otherwise cost 3000 gold directly — a 66% discount by laundering gold through shards, with zero opportunity cost since the trader has no gating/cooldown.

**Fix applied**: [ShardTraderScene.java](/forge-gui-mobile/src/forge/adventure/scene/ShardTraderScene.java#L16-L20) now exposes `SHARD_BUY_GOLD_COST`, `SHARD_BUY_QUANTITY`, and `GOLD_PER_SHARD` (=20) as public constants. [AdventureEventData.AdventureEventRules](/forge-gui-mobile/src/forge/adventure/data/AdventureEventData.java#L1040-L1080) now computes `shardsToEnter = goldToEnter / ShardTraderScene.GOLD_PER_SHARD` instead of using independent hardcoded values, so paying via gold or via shards (bought at the trader) now cost the same effective amount — no more arbitrage. Genuinely quest-earned shards (not purchased) remain a real discount since they weren't paid for with gold at all.

New shard entry costs: Constructed 75, Draft 150, Sealed 300, Jumpstart 10.

## 13. Known Gaps / Inconsistencies To Resolve

1. Coin-based entry does not scale with `townPriceModifier` and provides same card reward as full-price gold entry — need coin drop-rate data to assess risk.
2. Draft/Sealed guaranteed reward decks are sellable (no `isNoSell` flag), unlike Jumpstart's always-sellable-blocked packs — inconsistent treatment.
3. Card sell price is rarity-based only, ignoring real per-card power/market value — pack "value" is an approximation, bomb-heavy pools sell the same as bulk-heavy pools of the same rarity mix.
4. `townPriceModifier` scales entry fee but not reward value — cheap/low-prestige towns reduce cost without reducing payout.

## Next Steps (not yet decided)

- Get concrete Challenge Coin drop rates/sources to close the loop on point 1.
- Decide whether to mark Draft/Sealed low-tier "always awarded" rewards as no-sell (mirroring Jumpstart) or leave sellable but reduce guaranteed pack count on low win counts.
- Consider tying sell price more closely to card price data (paper price) rather than flat rarity tiers, if feasible.
