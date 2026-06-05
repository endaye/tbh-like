# TBH-like Godot Game: Steam Inventory and Market Plan

## 1. Product Direction

The project should not directly clone **TBH: Task Bar Hero**. The safer and more sustainable direction is to build an original desktop companion idle ARPG inspired by the same broad category:

- A tiny always-available desktop RPG window.
- Automatic combat and progression.
- Loot, classes, skills, and builds.
- Windows and macOS support.
- Online identity, leaderboards, Steam Inventory, and Steam Community Market integration.

The reference game's public Steam page positions it as a small idle RPG that runs from the taskbar, with pixel heroes, automatic loot, classes, skills, gear, multiple acts/difficulties, and Steam Market-tradeable items.

Reference: https://store.steampowered.com/app/3678970/TBH/

## 2. Confirmed Platform Goals

- Engine: Godot 4.
- Platforms: Windows and macOS first.
- Online features: Steam login, online progression, leaderboards, Steam Inventory, and Steam Community Market support.
- Economy: Steam Inventory / Steam Market integration, not a self-hosted marketplace.
- Gameplay shape: tiny desktop idle ARPG with auto-combat, loot, gear, class/skill builds, and long-term progression.

## 3. Recommended High-Level Architecture

```text
Godot Client
  -> GodotSteam / Steamworks SDK
  -> Steam Inventory
  -> Steam Community Market

Godot Client
  -> Game Backend API
  -> PostgreSQL
  -> Redis
  -> Steam Inventory Web API
```

The client should never be the final authority for valuable progression or market-facing item creation. Any item that affects ranking, monetization, or trading must be granted or validated through a trusted backend or through Steam's own inventory grant rules.

Steam Inventory supports two broad modes:

- Serverless mode, where the client talks to Steam Inventory directly for inventory retrieval, purchases, playtime drops, exchanges, and similar flows.
- Trusted-server mode, where a backend that knows game state uses privileged Steam Web APIs to grant or modify items.

For this project, trusted-server mode is the safer default because the game includes leaderboards and market-facing items.

Reference: https://partner.steamgames.com/doc/features/inventory?language=english

## 4. Steam Inventory and Market Constraints

Steam ItemDefs can mark items as:

- `marketable: true` - sellable on the Steam Community Market.
- `tradable: true` - tradable through Steam Trading.

Steam Inventory also supports:

- `item` - normal inventory item.
- `bundle` - expands into multiple items.
- `generator` - randomly grants one item from a weighted list.
- `playtimegenerator` - grantable through `ISteamInventory::TriggerItemDrop`.
- `tag_generator` - applies randomized item tags such as rarity, class, element, or visual effect.

Important constraint: dynamic item properties are not a good foundation for market-facing combat stats. Steam's documentation states that dynamic properties are cleared when an item is traded, and they are not currently visible when inspecting an item in Steam Inventory or on the Steam Community Market.

This means a complex ARPG item like:

```text
Attack +37
Critical Chance +8.4%
Fire Damage +12%
Skill Level +1
```

should not be represented only through dynamic properties if it is intended to be sold on the Steam Market.

References:

- https://partner.steamgames.com/doc/features/inventory/schema?language=english
- https://partner.steamgames.com/doc/features/inventory/itemtags
- https://partner.steamgames.com/doc/features/inventory/dynamicproperties?language=english

## 5. Recommended Economy Model

Use a two-layer item system.

### Layer A: Steam Market Items

These are items intended for Steam Inventory, Steam Trading, and Steam Community Market visibility.

Good candidates:

- Character skins.
- Weapon skins.
- Pets.
- Cosmetic effects.
- Profile badges or season trophies.
- Fixed-template rare items.
- Limited seasonal collectibles.

These items should use Steam ItemDefs and tags such as:

```text
type:sword
rarity:legendary
class:warrior
element:fire
season:s1
fx:flames
```

They can be filtered and searched cleanly in Steam inventory and market views.

### Layer B: Game Combat Items

These are internal gameplay items that can have rich randomized stats.

Good candidates:

- Weapons with numerical affixes.
- Armor with randomized stats.
- Skill-modifying equipment.
- Upgrade states.
- Reforged or crafted builds.

These should live in the game backend database and be synchronized to the client as gameplay state. They should not be directly marketable unless their variability is heavily constrained.

## 6. If Combat Items Must Be Marketable

If the design requires marketable combat gear, randomness should be discretized into finite templates instead of unlimited numerical rolls.

Example:

```text
Base: Redflame Sword
Rarity: Legendary
Class: Warrior
Primary Template: Crit A
Secondary Template: Fire B
Season: S1
```

This makes the item expressible through Steam ItemDefs and tags. It also keeps Steam Market listings understandable. The tradeoff is higher content-authoring cost and less fine-grained ARPG loot randomness.

## 7. Godot Integration Plan

Use **GodotSteam GDExtension** as the first integration path for Steamworks in Godot 4.

Current public information shows GodotSteam GDExtension 4.4+ supports Windows, Linux, Android ARM64, and Mac universal builds, and is based on Steamworks SDK 1.64.

Reference: https://godotengine.org/asset-library/asset/2445

Client responsibilities:

- Initialize Steam.
- Read Steam user identity.
- Load Steam Inventory.
- Display marketable/tradable items.
- Trigger Steam overlay or web flows where appropriate.
- Submit gameplay commands to the backend.
- Display combat, loot, builds, and desktop companion behavior.

Backend responsibilities:

- Validate Steam authentication.
- Own gameplay state.
- Compute rewards.
- Submit trusted inventory grants where needed.
- Maintain leaderboard authority.
- Audit economy-related events.

## 8. Desktop Window Considerations

Godot supports window features relevant to this game type:

- Borderless windows.
- Always-on-top windows.
- Transparent windows, depending on platform and GPU/compositor support.
- Status indicator / notification area integration.

Windows and macOS should be treated as separate platform targets for desktop behavior. Taskbar/menu-bar positioning, tray behavior, transparency, monitor scaling, window snapping, and background execution will need platform-specific testing.

References:

- https://docs.godotengine.org/en/4.5/classes/class_window.html
- https://docs.godotengine.org/en/4.4/classes/class_statusindicator.html

## 9. Suggested Development Phases

### Phase 1: Local Gameplay Prototype

- Tiny Godot window.
- Auto-combat loop.
- Basic hero, monster, skill, and loot systems.
- Local-only mock inventory.
- No Steam Economy integration yet.

Goal: prove the core idle RPG loop feels good.

### Phase 2: Steam Identity and Inventory Prototype

- Add GodotSteam.
- Initialize Steamworks.
- Load player Steam identity.
- Fetch Steam Inventory.
- Create private Steam ItemDefs for internal testing.
- Test non-marketable inventory items first.

Goal: prove Steamworks integration on Windows and macOS.

### Phase 3: Trusted Backend

- Steam auth validation.
- Player profile and progression.
- Server-authoritative reward generation.
- Leaderboard submission from backend-owned state.
- Audit logs for item and reward events.

Goal: prevent client-side manipulation from affecting ranking or economy.

### Phase 4: Marketable Item Beta

- Define a small set of marketable cosmetic or collectible items.
- Enable Steam Inventory visibility.
- Test tags, icons, localization, and market filtering.
- Keep combat power out of Steam Market for the first public test.

Goal: validate Steam Market behavior with low economic risk.

### Phase 5: Expanded Economy

- Add seasonal drops.
- Add rare cosmetics.
- Consider fixed-template marketable combat items only after telemetry and economy controls exist.
- Add fraud monitoring and support tooling.

Goal: grow the market without undermining gameplay balance.

## 10. Key Risks

- Client trust: any client-generated drop or score can be abused.
- Economy inflation: too many marketable drops will destroy item value.
- Market readability: overly complex random gear will be hard to evaluate on Steam Market.
- Dynamic properties: not suitable as the sole representation of market-facing item stats.
- Cross-platform desktop behavior: Windows and macOS windowing details need separate QA.
- Steamworks setup dependency: Inventory, Market, ItemDefs, icons, localization, and partner configuration must be tested inside Steamworks, not only in local Godot builds.

## 11. Current Recommendation

Start with a small, original desktop idle ARPG prototype, then add Steam identity and private Inventory testing before building any public economy.

For the first market version, make Steam Market items cosmetic or collectible. Keep deep combat stats in the backend until the economy is proven stable. If marketable combat items are required later, represent them as finite tagged templates rather than open-ended random stat rolls.
