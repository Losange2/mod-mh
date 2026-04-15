# mod-mh — Quest JSON batch processor (C#)

This repository contains a **C# console tool** (`Program.cs`) that **batch-processes** quest JSON files (named `q*.json`) from a `./quests` folder and **rewrites** them with:

- generated monster setups (target + companions) for specific quest types,
- normalized balancing values (HealthTable / AttackTable),
- rewards and unlock conditions based on internal data tables.

---

## What it contains (data tables)

The program hardcodes multiple dictionaries/sets that act like a small “game database”:

- **`questInfo`**: maps a quest ID → *(enemy difficulty tier, quest level)*  
  Example values: `("Village", "QL3")` or `("Master", "QL6")`.

- **`GetHealthTable()` / `GetAttackTable()`**: converts that *(tier, level)* pair into numeric **HealthTable** and **AttackTable** values used by enemies in the JSON.

- **`monsterIdByName`**: maps monster names → internal monster IDs.

- **`monsterIconByName`**: maps monster names → icon IDs (with a fallback icon value if missing).

- **`mapMonsters`**: maps a map ID → the list of monsters that are allowed/available on that map.

- **`questRewards`**: maps quest ID → *(Zenny, VillagePoints, RankPoints/HRP)* reward values.

- **`questConditions`**: maps quest ID → two integers representing unlock/access conditions.

- **`huntingQuests`**: a set of quest IDs that are considered hunt/kill/capture quests.  
  Only these quests receive the random target monster + map logic.

---

## What it does at runtime

### 1) Ensures the output directory exists

```csharp
outputDir = "quests";
Directory.CreateDirectory(outputDir);
```

### 2) Reads all existing quest files

```csharp
Directory.GetFiles("./quests", "q*.json");
```

For each file:

- Extracts the quest ID from the filename (e.g. `q104.json` → `104`)
- Parses the JSON using `System.Text.Json.Nodes` so it can edit fields directly

---

## Hunting quest logic (`huntingQuests`)

If the quest is a hunting quest (`huntingQuests.Contains(questId)`), it:

1. Picks a random target monster from all monsters in `monsterIdByName`.
2. Finds which maps can host that monster using `mapMonsters`.
3. Chooses a random valid map among those; if none match, it falls back to **map 10**.
4. Updates quest JSON fields such as:
   - `QuestData.Monsters[0].Id` (target monster)
   - `QuestData.TargetMonsters[0]`
   - `QuestData.Icons[0]`
   - `QuestData.Map`
5. Fills the other monster slots (`Monsters[1]..[6]`) with random “companion” monsters from the same map (or `0` if none).
6. Sets small monster spawn behavior:
   - If map 10 → `SpawnType = 0`
   - Else → random `SpawnType` from `0..50`
7. Applies quest unlock conditions from `questConditions` when available.

---

## Logic applied to all quests

For all quests (hunting or not), it:

1. Computes `healthTable` / `attackTable` from `questInfo`.
2. Writes those values into:
   - `EnemyData.Monsters[i].HealthTable` / `EnemyData.Monsters[i].AttackTable` for `i = 0..6`
   - `EnemyData.SmallMonsters.HealthTable` / `EnemyData.SmallMonsters.AttackTable`
3. Applies rewards when defined:
   - Writes Zenny, Points, and HRP into `QuestData.Reward`.

---

## Output

- Writes the modified JSON back to (pretty-printed):

```text
quests/q{questId}.json
```

- Prints a summary like:

```text
"X files created, Y skipped."
```

> Note: `skipped` is declared but never incremented, so it will always print `0`.

---

## Important behavior notes / caveats

- A new `Random` is created inside the loop (`new Random()` per file), which can reduce randomness if many files are processed very quickly (same timing seed).  
  Typically you would create a single `Random` outside the loop.

- It overwrites files in the `quests` output directory using the same naming scheme (`q{questId}.json`), effectively regenerating quest content based on these rules.

---

## Need details about the JSON format?

If you tell me what game/mod format these JSON files correspond to (or share a sample `q*.json`), I can describe exactly what each edited field means in that format.
