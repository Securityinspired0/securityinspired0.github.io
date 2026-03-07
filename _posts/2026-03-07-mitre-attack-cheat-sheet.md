---
title: "MITRE ATT&CK: The Cheat Sheet That Makes You a Better Defender"
date: 2026-03-07
categories:
  - blog
tags:
  - mitre-attack
  - threat-intelligence
  - blue-team
  - incident-response
  - frameworks
excerpt: "Understanding MITRE ATT&CK doesn't just make you sound smart in meetings — it fundamentally changes the way you think about attacks and incidents."
---

If you've ever heard the term "MITRE ATT&CK" thrown around in a cybersecurity conversation and nodded along pretending you understood — this post is for you. 😄

And here's the thing nobody tells you: understanding this framework doesn't just make you sound smart at meetings. It actually **changes the way you think about attacks and incidents.** That's the real value.

Let me break it down the way I wish someone had broken it down for me.

---

## So What Even Is MITRE ATT&CK?

MITRE is a non-profit research organisation that works closely with the US government. Back around 2013, they noticed something frustrating — security teams were great at blocking individual attacks, but nobody had a shared way of describing *how hackers actually behave.*

So they built one. A giant, publicly available knowledge base of real hacker behaviour, pulled from real attacks that actually happened.

They called it **ATT&CK** — which stands for:

> **A**dversarial **T**actics, **T**echniques **&** **C**ommon **K**nowledge

Think of it as a **Pokédex for hacker moves.** Every known method a hacker uses to break in, move around, steal data, or cause damage — it's catalogued here.

---

## The Part That Actually Matters: TTPs

The heart of MITRE ATT&CK is something called **TTPs** — Tactics, Techniques, and Procedures. Once you understand these three things, the whole framework clicks.

### 🎯 Tactics — The Goal

A Tactic is simply what an attacker is **trying to achieve** at a given moment. It's the goal, the objective, the "why."

For example:
- *"I want to get inside this network"* — that's a Tactic (**Initial Access**)
- *"I want to steal passwords"* — that's a Tactic (**Credential Access**)
- *"I want to hide from security tools"* — that's a Tactic (**Defense Evasion**)

MITRE ATT&CK has **14 Tactics** in total, and they roughly follow the stages of a real attack from start to finish — from initial planning all the way to stealing data or causing damage.

### 🔧 Techniques — The Pattern

A Technique is **how** attackers achieve their goal. But here's what makes it special: it's not just theory. Techniques are **patterns spotted across many real attacks.**

Think of it like a detective who has investigated 100 burglaries and notices that a large number of them involved someone climbing through a window. They give that pattern a name. That name is the Technique.

A classic example: **Phishing.** Researchers looked at thousands of incidents and noticed attackers kept sending fake emails to trick people into giving up access. That recurring pattern became a named Technique under the Initial Access Tactic.

### 📋 Procedures — The Fingerprint

A Procedure is where it gets really specific. It's the **exact steps a particular hacker group used** in a real, documented attack.

It answers: *"How did THIS group specifically carry out THAT technique in THAT incident?"*

For example:

> APT28, a Russian hacker group, sent fake Microsoft login emails to Ukrainian government officials. The emails contained a link to a convincing fake login page. Once officials entered their credentials, APT28 captured them and used those passwords to move deeper into the network.

That's a Procedure. It has a named group, a specific target, specific steps, and it's drawn from real forensic evidence — almost like a crime scene report.

---

## How They All Connect

Here's the simplest way to see how Tactics, Techniques, and Procedures fit together:

| | Question it answers | Example |
|---|---|---|
| **Tactic** | What does the attacker want? | Steal passwords (Credential Access) |
| **Technique** | What general method do they use? | Fake login pages (Credential Phishing) |
| **Procedure** | What exactly did THIS group do? | APT28 sent fake Microsoft emails to Ukrainian officials |

One flows into the next. The Tactic is the goal, the Technique is the pattern used to reach that goal, and the Procedure is the real-world evidence of it happening.

---

## The 14 Tactics (The Attack Story)

Think of these as chapters in a heist movie, roughly in the order an attacker follows them:

1. 🔭 **Reconnaissance** — Spy and gather info before attacking
2. 💣 **Resource Development** — Build tools and set up infrastructure
3. 🚪 **Initial Access** — Get a foot in the door
4. ⚙️ **Execution** — Run malicious code
5. 🪝 **Persistence** — Make sure they can come back
6. 📈 **Privilege Escalation** — Gain more powerful access (become admin)
7. 👻 **Defense Evasion** — Hide from antivirus and security tools
8. 🔑 **Credential Access** — Steal passwords
9. 🔍 **Discovery** — Look around and understand the network
10. 🐍 **Lateral Movement** — Move to other computers in the network
11. 📦 **Collection** — Gather the data they want to steal
12. 📡 **Command & Control** — Communicate with hacking tools remotely
13. 🚚 **Exfiltration** — Sneak the stolen data out
14. 💥 **Impact** — Break, encrypt, or destroy things

---

## Why Does This Change How You Respond to Incidents?

Here's the thing that doesn't get said enough: **knowing MITRE ATT&CK changes your mindset.**

Without it, when something goes wrong you ask: *"What happened?"*

With it, you ask: *"Where in the attack chain did this happen? What Tactic does this map to? What likely comes next?"*

That shift is huge. Instead of reacting to individual alerts in isolation, you start **seeing attacks as a story with a beginning, middle, and end.** You can anticipate the next move. You can look for things that haven't triggered an alert yet but probably will.

Security teams use MITRE ATT&CK to:

- **Find gaps** — *"We have no detection for Lateral Movement techniques. That's a blind spot."*
- **Simulate attacks** — Red teams use it to test defences using the same playbook real attackers use
- **Investigate incidents** — *"This log entry maps to T1566 — Phishing. Let's trace what happened next."*
- **Speak the same language** — When someone says T1078, every security professional knows that means Valid Accounts. No ambiguity.

---

## Final Thought

MITRE ATT&CK isn't just a framework for researchers in a lab somewhere. It's a practical tool that, once understood, rewires how you approach every security conversation, every alert, and every incident.

You don't need to memorise all 200+ techniques. Start with the 14 Tactics. Learn the Techniques one at a time as you encounter them. And when you read a threat report and spot a Procedure — trace it back up the chain.

That habit alone puts you ahead of most people in the room. 🛡️
