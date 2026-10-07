# CLAUDE.md — Tactical Hacking Game ("Operator")

Read section 0 first. It explains how this user works, and it matters as much as the design notes.

## 0. How to work with this user (David)

- David is the designer. You are the programmer and design-consistency checker. Design decisions are his.
- Do NOT invent mechanics, buttons or "fixes" he didn't ask for. This went wrong once: a "Recharge" button was added unprompted and he made it be reverted. Propose, wait for a yes, then build.
- When he writes "thoughts?" he wants discussion only. No code changes.
- Numbers in the code are PLACEHOLDERS. He said "do whatever values for now so I can see how it works." Pick sensible values and say they're placeholders.
- Work incrementally: simplest version first, add one thing at a time, play it, refine. Say out loud every simplification you make.
- Show numbers, never progress bars. Readiness is a rising percentage in plain text. No bars or gauges in the matrix UI.
- Before building, check new ideas against locked rules and point out contradictions. He values this a lot. When he is confused, concrete worked examples with real numbers help.
- Keep explanations short and plain. He gets overwhelmed when things stack up.
- He prefers the finished result delivered directly once direction is clear. Ask only when there is a real fork in design.
- Real hacking and real EW (electronic warfare) terminology is the preferred source of names and mechanics. He liked grounding choices in how actual intrusions work.

## 1. What the game is

A single-player, text-only, terminal-styled hacking game. The player is an "Operator" who jacks into a matrix and hacks nodes owned by a company. The fight IS the hack. You breach the enemy's Firewall, then drain their Connection with payloads, while they do the same to you. There is no map and no grid in combat or exploration. Everything is dense, live-updating text blocks (green on black, monospace). The goal is a screen that looks like chaos to an outsider but is fully legible to the player. Complexity should appear gradually as the player progresses.

Lore framing: the Operator's interface renders raw code-level exchange as combat for human legibility. This is flavor only. No mechanics depend on it.

Art rules: the matrix (combat and exploration) is text only. The Hub is "outside the matrix" and may use real art (pixel-art shopkeeper portrait, item images).

Narrative premise (linear, scripted, NOT written yet): three acts. Defender (you are a recruit protecting a company's nodes) -> Discovery (you learn the company is bad) -> Defector (you turn on it). Story is delivered only outside the matrix: at the Hub through the shopkeeper in a fixed sequence (not reactive to exploration order), plus major reveals after boss fights. In the matrix there are only subtle clues. New Game starts with a tutorial: the shopkeeper explains how to play while opening the story (you're a recruit). Non-attackable nodes in Act 1 are the company's protected infrastructure, which become attackable after the turn. Cleared sectors may later host hostile nodes after the reveal, so a sector's "cleared" tag must be able to change again.

## 2. Screens and navigation

Intro (New Game / Continue / Quit) -> Hub (Shopkeeper / Manage Equipment / Enter Matrix / Quit) -> Sector Select -> Coordinate List -> scan a coordinate -> Engage (hostile only) -> Combat -> back to Coordinate List.

- Back buttons return to the previously visited screen (a navigation history stack), NOT a fixed parent. Implemented with screenHistory / goBack().
- Hub Quit returns to Intro; Intro Quit exits to a completely blank screen. Both ask for confirmation (overlay with confirm/cancel).
- Continue is shown but disabled (no saves exist yet).
- Sector Select shows a per-sector "cleared" tag. Coordinates are text like 12.04.11, a fixed finite number per sector, start unknown, and are revealed by scanning. Scan result is empty / node / hostile. A hostile can be engaged or skipped. A sector is cleared when every coordinate is revealed and every hostile is defeated.
- Hub: Shopkeeper = shop + repair + story delivery on one screen (all "talking to a character"). Manage Equipment = loadout + unit management. Both are placeholders.
- Planned saves: autosave after every combat and after every hub action (purchase, upgrade, equip). 3 slots. Continue always resumes at the Hub, never mid-combat. Cleared progress persists.
- Loss state (designed, not built): on losing you are disconnected from the run, lose money and your gear is damaged and needs repair. No permadeath.
- Intended full battle layout (mockup in battle-screen.html, not in the live combat build): four columns: Operator + log | Operator's 4 units | enemy's 4 units | enemy + contextual action panel. Click the Operator or a unit (hover = brighter border, selected = brighter and thicker border), then an action, then a target. The current combat build shows only Operator vs Enemy (two panels).
- Buttons: "new game" (was "start"), "disconnect" (was "flee"), "skip turn" (see section 9, it is slated for removal). Defend and Reroute buttons were placeholders that were deleted.

## 3. Combat model (as built in exploration-sample.html)

Core: each side has Connection (the HP equivalent; 0 = severed = that side loses) protected by Firewall (an access gate). You cannot hurt Connection until the Firewall is breached (Firewall = 0). Both sides also have Power (mana), CPU and RAM (displayed only, see section 9), and a readiness gauge.

Action families (names come from the real intrusion kill chain: exploit gets access, payload does damage, EW counters defenses):
1. EXPLOIT (only vs. an unbreached Firewall; never damages Connection): Brute Force (reliable), Zero-Day (rare, expensive, big), Sustained (damage-over-time on the Firewall; one instance per source, recasting refreshes and does not stack; it stops once breached).
2. PAYLOAD (only after breach; damages Connection): Worm (reliable), Logic Bomb (heavy, immediate), Trojan (delayed detonation after a fuse; while armed the planter is "exposed" and takes 1.5x incoming Worm damage; the enemy can counter it).
3. DISRUPTION (in the Attack menu, after breach): Jam (knocks the enemy's readiness back 50%), Disable (locks the enemy's Worm for 10 s).
4. DEFENSE (usable anytime; NOT gated by breach): Decoy (enemy wastes its next action, then you land a free counter), Harden (halves the next hit you take), ECCM (8 s of immunity to enemy Jam), Evict (removes an enemy-planted Trojan; only enabled when one is on you). Quarantine is listed but disabled, waiting on units.

Placeholder numbers (power cost / cooldown seconds / effect): Brute Force 20/4/20 fw dmg. Zero-Day 40/10/45 fw dmg. Sustained 25/5/8 per 3 s. Worm 20/2/15. Logic Bomb 35/6/35. Trojan 20/4/55 after a 12 s fuse. Jam 15/4/-50% readiness. Disable 20/8/10 s lock. Decoy 15/4. Harden 20/5. ECCM 20/6/8 s. Evict 20/5. Start stats: player Connection 150, Firewall 60, Power 100; enemy Connection 150, Firewall 50, Power 100. Enemy is deliberately slightly weaker (readiness 13/s vs 15/s, Power regen 1.6/s vs 2/s, smaller damage).

Enemy AI (simple on purpose): before your Firewall is breached it uses Exploit on it. After the breach it picks randomly among whatever is ready out of Worm / Jam / Trojan (its Jam is blocked by ECCM; its Trojan can be Evicted; its damage is halved by Harden). It has no Zero-Day, Sustained or Logic Bomb. Every enemy is currently identical.

Pacing in the current build (ATB hybrid): each side's readiness gauge fills continuously (player 15/s, enemy 13/s, shown as a plain percentage). When the player's reaches 100%, the WHOLE battle freezes (Power regen, cooldowns, fuses, enemy gauge) and the Attack / Defense / Skip turn / Disconnect menu opens. The enemy acts the instant its own gauge fills. Power regenerates in real time (+2/s) and also freezes during the player's decision. Each action has a Power cost and a real-second cooldown. The player starts a fight at 100% readiness so the first move is instant. Engaging a hostile coordinate IS the engage action (there is no separate "engage target" prompt). Win = enemy Connection <= 0. Loss = yours <= 0 or Disconnect; the hostile stays on the map so it can be retried.

## 4. OPEN DESIGN FORK: pacing model (unresolved, was mid-discussion at export)

Problems found by playing:
- Three overlapping pacing layers: readiness gauge + per-action cooldowns + Power cost/regen.
- "Skip turn" is a leftover from the older turn-timer build; its original purpose (cashing in unused turn seconds as Power) no longer exists.
- If readiness is full but nothing is usable (everything on cooldown or too expensive), the game freezes with an empty menu, and since Power also freezes it can never recover (soft-lock; only Disconnect exits).
Options discussed in order:
 a. Remove skip turn. Only open the decision menu when at least one action is usable; otherwise the clock keeps running while cooldowns tick and Power regenerates (the action-RPG "everything's on cooldown while mana regens" feel).
 b. Slow real-time combat with pause-in-menus (cooldowns + mana).
 c. User's latest direction: readiness can sit at 100% while the battle KEEPS RUNNING (Power regens, cooldowns tick, the enemy keeps acting; if the enemy Jams you while you wait, you lose that readiness). The game pauses only once the player opens a submenu (Attack/Defense) to choose. This is classic ATB "wait mode". Backing out of a submenu without choosing resumes the clock.
 d. User then asked about dropping cooldowns and using Power cost alone ("you're open the entire time, so you should balance your power use"). Analysis: most cooldowns (<=6 s) are already shorter than the ~6.7 s gauge refill, so only Zero-Day (10 s) and Disable (8 s) ever matter. Power-only would work. Costs: RAM (meant to speed up EW cooldowns) loses its job; Zero-Day's rarity must come from price alone; banking Power for burst turns possible. Recommendation given: try Power-only first, and re-add a cooldown to Zero-Day only if banking proves too strong. The user had NOT confirmed.
Ask the user which model to try before changing pacing code. Tell him the content (Firewall/Connection, action lists, enemy kit, exploration, hub) does not depend on the pacing model, so this is a rework of the tick loop, not a redesign.

## 5. Units (designed, NOT built)

- Up to 4 slots: max 2 offense + 2 defense. Bought at the shop. Role is fixed at acquisition; upgrades deepen the role rather than reassign it.
- Model: additional machines the Operator runs in parallel (DDoS-style: many machines doing the same thing adds up), not a squad of specialist classes. The old Attacker/Blocker/Healer/Support job idea was dropped.
- Offense units add parallel pressure on the Firewall (they can run Sustained Exploit; each source, meaning the Operator or each unit, has its own non-stacking instance). Defense units reroute/block, mirroring what the enemy does to you.
- Units do NOT draw from the Operator's Power pool.
- Units use the same stat template (Connection, Firewall, Power, CPU, RAM) and can be targeted like anything else.
- Mostly automatic. The Operator can redirect a unit for a manual order (select unit -> action -> target, as in battle-screen.html); afterward that unit is briefly unavailable. Surface this in the log as a status line like "unit 01 not responding" (avoid the literal phrase "cooldown active").
- The enemy can field up to 4 units too.
- Hijacking: you can hijack enemy units and they can hijack yours. Counters: Signal Encryption (prevention, an anti-hijack defense) and Quarantine (reaction: cut off a hijacked unit, yours or an enemy's; open question whether that is one action or two).
- Obsolete rule: "+10 seconds of turn time per unit (hijacked ones too)". It belonged to the turn-timer version and means nothing under ATB. Decide what replaces it, or drop it.
- THREE UNIT QUESTIONS WERE ASKED AND NOT YET ANSWERED: (1) When a unit's readiness fills: act automatically right away / wait for command / auto-act but overridable in your window? (2) What replaces the +10 s rule? (3) How to supply units for testing with no shop: 2 offense + 2 defense / 1 + 1 / pick on Manage Equipment?

## 6. Scan (designed, NOT built in the current builds; only Sample 1 had an older version)

Passive, always on, tiered against the enemy's Firewall tier. Baseline tier reveals core enemy stats AND the enemy's next action (foresight; so a weak Operator can see they are outclassed and Disconnect). Higher tiers reveal deeper stats (Power, CPU/RAM). During an active breach everything is revealed. Once revealed, a stat updates live. Enemy stats start completely hidden. Foresight is "prep, not haste": information, not extra actions. The enemy "charging up" is surfaced by YOUR Scan, not announced by the enemy. Scan tier is a hardware upgrade. The enemy needs to telegraph big moves for foresight to matter, which isn't built. Old idea: an enemy log panel unlocking at Scan tier 3.

## 7. Decisions and the reasons behind them (do not undo)

1. No map or grid in combat or exploration. The matrix has no physical space; state is shown as stat blocks and text lists. Only the Hub may use art.
2. The old Shield/Armor/HP chain and Plasma/Laser/Missiles weapons were RETIRED after playtesting (battle-sample.html): one weapon dominated (Plasma spam), EW felt pointless, and the Operator had no way to react. Physical weapons also clashed with the cyberspace theme.
3. HP became Connection: "who stays connected" is the hacking-native win condition. Firewall is the access gate (breach first, then damage), matching real intrusions.
4. Exploit and Payload are separate tool families. Exploit (Brute Force etc.) never touches Connection; Payload does the damage. This mirrors the real kill chain and games like Uplink and Cyberpunk 2077 (breach tools vs. quickhack tools).
5. Each action family gets three options with different COMMITMENT SHAPES (reliable / burst / delayed or sustained), not three sizes of the same thing. That avoids a dominant spam option.
6. Jam and Disable moved from Defense into Attack because they hinder the enemy. Defense (Decoy/Harden/ECCM/Evict) is not gated by breach, since protecting yourself shouldn't depend on your own offense having succeeded.
7. Cut or merged for simplicity: Overheat folded in as a future upgrade tier of Disable (not built); Backup Power and power channels cut; Cooling folded into Power; Connection status+strength merged; Radar and Scan unified; the Range band system removed from the playable builds (it only mattered for the retired weapons).
8. Power is the mana pool and regenerates continuously in real time (the user's explicit design, "always ticking"). It also freezes during the player's decision window (explicit). Lesson learned: if regen is lower than the cheapest action's cost, the player gets trapped at a floor. Keep regen above the cheapest action cost. Power regen rate is upgradeable at the shop (noted).
9. Readiness is a text percentage (no bars). A first-move instant start replaces a cold start.
10. The enemy mirrors the player's kit but is simpler and weaker on purpose, so the player's tools are tested first.
11. Units stay simple and the Operator holds the complexity. Reactive Power rerouting was shelved as "too slippery" for the Operator to juggle; defense units and Harden take over that role.
12. Fixed hub story sequence, autosave-everything and Continue-always-to-Hub were chosen to keep the system simple and never lose progress.
13. Design method: playtest a small sample, find what breaks, rebuild. The old foundation was thrown away after Sample 1 for exactly that reason. Add one system at a time.

## 8. What is finished and working

- Full navigation shell: Intro, Hub, Shopkeeper/Equipment placeholders, Sector Select, Coordinate List, Combat, history-based Back, confirm overlays, blank exit screen.
- Procedural sectors (3 sectors x 6 coordinates, 40% empty / 30% node / 30% hostile), scan-to-reveal, cleared tracking.
- Complete combat loop vs. a simple AI: Exploit -> breach -> Payload, Jam/Disable, Decoy/Harden/ECCM/Evict, enemy Jam and Trojan, Sustained ticks, Trojan fuses and exposure, decision-window freeze, first-move-instant, win/lose handling that feeds back into exploration.

## 9. Unfinished, partial or broken

- Pacing fork unresolved (section 4); skip turn still exists; empty-menu soft-lock is possible.
- CPU and RAM are displayed but have NO mechanical effect in the current builds (battle-sample-3.html used them to shorten cooldowns; the ATB builds hard-code seconds). Intended meaning: CPU = how often you act (tempo), RAM = how fast your utility/EW options recharge.
- Minor display bugs: the Trojan fuse and Disable lock timers show raw floating-point seconds (for example 11.799999). Wrap them in Math.ceil() like the ECCM and enemy-Trojan timers.
- btnLeaveCombat in exploration-sample.html is permanently disabled and unused.
- Sectors and coordinates are random on every page load; nothing is saved.
- Nodes are placeholders (they only say "data retrieved"). No rewards, currency, shop, repair or loss penalties exist. Shopkeeper and Equipment screens are stubs.
- Every enemy is identical; no scaling by sector.
- The combat engine is copy-pasted across battle-sample-4.html and exploration-sample.html (and older variants in the other files). Consolidate it before adding units.
- Units, Scan/foresight, enemy telegraphing, Quarantine, Signal Encryption and hijacking are all unbuilt.

## 10. Ideas discussed but not built

- Music and sound effects (later pass).
- New Game tutorial led by the shopkeeper ("you're a recruit"); probably needs a scripted first fight.
- New Game will need a save-slot step once saves exist.
- Shop: currency and how it is earned, parts and unit purchasing, pricing, Power-regen upgrades, Overheat as a Disable upgrade tier, repair mechanics, loadout-assembly screen. Pixel-art shopkeeper and item art.
- Mission-type flavor on Payload (retrieve info vs. disable a network): after the fight the mission is simply done.
- Enemy variety, bosses, enemy units, enemy EW depth.
- Pattern memorization (learn enemy-type behavior across fights) as a possible later system.
- Sectors turning hostile after the story's turn, with a status like "cleared - activity detected".
- Gradual UI reveal as the player progresses.

## 11. What we were about to do next

1. Settle the pacing fork (section 4), remove skip turn, and fix the empty-menu soft-lock.
2. Ask David the three unit questions (section 5), then build units.
3. Build the save system (3 slots, autosave rules above), which also makes Continue and New Game meaningful.
4. Hub content: shopkeeper, shop and currency, repair, loadout screen, then the tutorial.
5. Scan/foresight and enemy telegraphing.
6. Narrative writing, music and SFX.
Suggested engineering step when you start: split the single-file HTML into index.html + css/ + js/ modules (state, combat, exploration, hub, ui), keeping the numbers in one config object (the ACTIONS table) so balance tweaks are easy.

## 12. Files

All files are single-file HTML with inline CSS and JS. There is no build step and no dependencies. Open in a browser to run.
- exploration-sample.html: CURRENT MAIN BUILD. Intro -> Hub -> Sector Select -> Coordinates -> ATB-hybrid combat, fully connected. Start here.
- battle-sample-4.html: standalone version of the ATB combat engine (readiness gauges, decision-window freeze, Defense menu, smarter enemy). The same engine is embedded in exploration-sample.html.
- battle-sample-3.html: reference. The older TURN-BASED version of the Connection/Firewall combat: 10-second turn timer, Skip with leftover-time-to-Power conversion, CPU/RAM tiers shortening cooldowns, Jam as an extra turn, Defense = Decoy/Jam/Disable. Useful as a record of how the system evolved.
- battle-sample.html: reference only, OBSOLETE foundation: Shield/Armor/HP with Plasma/Laser/Missiles, Scan reveal tiers, EW waves. Kept as a record of what was tested and discarded.
- battle-screen.html: reference. Static four-column layout mockup with the select-actor -> action -> target interaction, hover/selected borders and a unit "not responding" cooldown demo. Reuse its interaction model for units.
- (Not part of this game: two unrelated early escape-room files, a Python CLI and a JS/HTML port, were uploaded at the start of the chat and are not included.)
