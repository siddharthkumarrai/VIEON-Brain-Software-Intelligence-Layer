# VIEON — Brain / Software Intelligence Layer

This document covers the **software brain** — the part of VIEON that
makes it an intelligence system instead of a command-triggered toy.
Hardware (ESP32, VC-02, OLED) is the body. This is what makes decisions.

---

## 1. The Core Distinction — Why Path A Is Not Enough

**Path A (VC-02 fixed corpus):**
```
Fixed phrase → matched against ≤150 pre-trained patterns → fixed hex ID → fixed action
```
This is pattern matching. It has no understanding. It cannot handle
anything you didn't explicitly train. "Wave your hand" works only if
you trained that exact phrase. It will never handle "can you check if
my laptop's battery is low" unless you hardcoded that exact sentence
as a command — and even then, "is my laptop's battery low" (a
rephrasing) would fail to match.

**Path B (the Brain):**
```
Whatever you say → transcribed to text → reasoned about by an LLM
→ LLM decides which capability, on which node, with which parameters
→ dispatched → executed → result reasoned about again → spoken reply
```
This is reasoning. The LLM is making a live decision based on natural
language and the current state of the system — not matching a fixed
pattern. New abilities get added by registering a new capability, not
by retraining a wake-word corpus.

**VC-02 still has a job** — it's your low-latency, zero-cost, always-on
wake-word + a handful of safety-critical commands (e.g. "stop", "sleep")
that must work even if the Brain, WiFi, or cloud LLM is down. Everything
else routes through the Brain.

---

## 2. Five Layers

```
┌─────────────┐   ┌─────────────┐   ┌───────────────┐   ┌────────────┐   ┌──────────┐
│  PERCEPTION │ → │  COGNITION  │ → │ ORCHESTRATION │ → │ EXECUTION  │ → │ FEEDBACK │
│ audio → text│   │  the Brain  │   │  the Hub (HRP)│   │  the Node  │   │ TTS/OLED │
└─────────────┘   └─────────────┘   └───────────────┘   └────────────┘   └──────────┘
```

| Layer | Responsibility | Lives on |
|---|---|---|
| Perception | Audio capture → transcript | ESP32-CAM mic / phone mic / PC mic + STT engine |
| Cognition | Decide *what* to do and *why* | Brain server (this doc) |
| Orchestration | Decide *which node* does it, *when* | Hub (already in your main README's HRP design) |
| Execution | Actually do it | Robot / Mobile / PC nodes |
| Feedback | Tell the human what happened | TTS reply, OLED status |

The Brain does **not** talk to hardware directly. It talks to the Hub.
The Hub talks to nodes. This separation is what lets you swap a phone
for a different phone, or add a second robot, without touching the
Brain's reasoning logic at all.

---

## 3. The Capability Registry — This Is the Actual Design Problem

An LLM can only decide to do things it knows are possible. The
Capability Registry is the list of "things any node can currently do,"
handed to the LLM as its available tools at request time. This maps
directly onto the `vieon.plugin.json` Universal Plugin Contract already
sketched in your main README — this doc just makes it concrete for the
Brain's use.

Each online node publishes a manifest like:

```json
{
  "node_id": "robot-esp32-01",
  "node_type": "robot",
  "capabilities": [
    {
      "name": "wave_hand",
      "description": "Wave the robot's right arm in a greeting gesture",
      "parameters": {}
    },
    {
      "name": "walk",
      "description": "Walk in a direction for a number of steps",
      "parameters": {
        "direction": { "type": "string", "enum": ["forward", "backward", "left", "right"] },
        "steps": { "type": "integer" }
      }
    },
    {
      "name": "sit",
      "description": "Lower into a seated resting position",
      "parameters": {}
    }
  ]
}
```

```json
{
  "node_id": "mobile-pixel-01",
  "node_type": "mobile",
  "capabilities": [
    {
      "name": "check_battery",
      "description": "Report current battery percentage and charging state",
      "parameters": {}
    },
    {
      "name": "open_app",
      "description": "Launch a named app on the phone",
      "parameters": { "app_name": { "type": "string" } }
    }
  ]
}
```

The Brain merges every currently-online node's manifest into one tool
list before every LLM call. A node that's offline (per the Hub's live
state) simply isn't offered as an option — the LLM never gets to "hallucinate"
calling something unavailable, because it's not even given the tool.

---

## 4. The Reasoning Loop

This is the actual decision cycle, once per voice interaction:

```
1. Transcript arrives: "VIEON, check if my laptop battery is low"
2. Brain builds the live tool list from the Capability Registry
3. Brain calls the LLM with:
     - system prompt: "You are VIEON, a robot's decision-making core.
       Given the user's request and the available tools, decide which
       tool(s) to call. If nothing fits, say so honestly."
     - the transcript
     - the tool list
4. LLM returns a structured function call:
     { "tool": "check_battery", "node_id": "mobile-pixel-01", "params": {} }
5. Brain sends this to the Hub for execution (Hub applies its own HRP
   scoring/routing on top of this — the Brain picks *what*, the Hub
   confirms *which specific node instance* and *when*)
6. Node executes, returns a result: { "battery_pct": 14, "charging": false }
7. Brain calls the LLM again (or uses a template) to turn that result
   into a natural spoken reply: "Your laptop's at 14 percent and not
   charging, sir."
8. Reply goes out via TTS / OLED
```

Steps 3 and 7 are two separate LLM calls with two separate jobs —
**deciding what to do**, and **explaining what happened**. Don't
collapse these into one call; keeping them separate makes both prompts
simpler and easier to debug when something goes wrong.

---

## 5. What Makes This "Self-Controlling" Rather Than Scripted

- No `if "wave" in transcript: wave()` anywhere in this system. The LLM
  reads the capability descriptions and decides for itself which one
  fits, including for phrasings you never anticipated.
- The Brain has no hardcoded node addresses — it only ever sees
  currently-online capabilities from the Hub, so the system's behavior
  changes automatically as hardware comes online/offline.
- Adding a new ability to the robot means: write the ESP32-side
  function, add one manifest entry describing it in plain language.
  Nothing about the Brain's code changes.

---

## 6. What's Next (Software-Only, No New Hardware Needed)

You said you're on brain/software design right now — here's the order
that lets you build and test this entirely without new wiring:

1. **Mock the nodes.** Write fake capability manifests and fake
   execution responses (just return canned JSON) for robot/mobile/PC.
   You don't need real hardware to test whether the Brain reasons
   correctly.
2. **Build the Brain server** (starter code below) and test it with
   typed text input first — skip STT entirely for now.
3. **Verify the reasoning loop** — throw varied phrasings at it
   ("can you walk forward a bit", "take 3 steps ahead", "move
   forward") and confirm the LLM consistently picks `walk` with
   sensible params, using only your mocked manifests.
4. Only after that's solid: wire in real STT, then real nodes one at
   a time (VC-02 for wake-word + safety commands stays exactly as
   already built).

---

## 7. Starter Code

See `brain_server.py` in this same delivery — a runnable FastAPI
skeleton implementing the Capability Registry + the two-call reasoning
loop above, with mocked nodes so you can test the decision-making
logic today, with zero hardware attached.
