[Home](../README.md) · [← Previous: 7.3 Local LLMs](../07-ai-ml-workstation/03-local-llms-ollama-openwebui.md) · **08 — Productivity & Security** · [Next: 8.1 Keyboard →](01-keyboard-automation.md)

---

# 08 — Productivity and Security

Ergonomics (keyboard, automation), data durability (3-2-1 backups), and security hygiene (firewall, sandboxing, hardware keys).

## Steps

| #   | Step                                                                           | Time    | Skippable?                                                 |
| ---:| ------------------------------------------------------------------------------ | ------- | ---------------------------------------------------------- |
| 8.1 | [Keyboard automation (KRunner, KeyD, AutoKey) + Activities](01-keyboard-automation.md) | 15 min  | Partially — KeyD remap is high-ROI, AutoKey optional       |
| 8.2 | [3-2-1 backup strategy (Restic + rclone + offsite)](02-backup-strategy-3-2-1.md) | 25 min  | **No. Timeshift is not a backup.**                         |
| 8.3 | [Firewall, sandboxing, YubiKey, ClamAV](03-firewall-sandboxing.md)             | 20 min  | Partially — UFW enable + Firejail on Chromium is the 80/20 |

Total: ~60 minutes.

## Framing

The whole guide up to here makes the laptop **fast and useful**. This section makes it **durable and defensible**:

- **Ergonomics** compounds — a well-remapped keyboard and text expansion save 10–20 minutes a day, which is ~80 hours a year. Worth 15 minutes once.
- **Backups** are asymmetric — the cost is 25 minutes now plus $30/year for offsite storage; the payoff is "single-laptop-fire doesn't end your year". There is no other item in this guide with as skewed a risk/cost ratio.
- **Security** for a personal laptop is about the 80/20: UFW on + browser sandboxing + don't run as root + SSH keys in an agent. The remaining 20 % (AppArmor policy tuning, SELinux, auditd, full mandatory access control) is for Defense-in-Depth shops, not a marine-robotics dev laptop.

## What we do NOT cover

- **LUKS** — see [00 Executive Summary § non-goals](../00-executive-summary.md#what-this-guide-does-not-cover-and-why).
- **auditd / OSSEC / AppArmor policy authoring** — Ubuntu's default AppArmor is fine; don't tune it.
- **fail2ban on a desktop** — fail2ban matters on internet-reachable servers. Your laptop is behind NAT and has no exposed services (after UFW). Skip.
- **Enterprise disk-health monitoring (smartd custom thresholds)** — `smartctl -a /dev/nvmeXnY` once a quarter is enough.

---

[Home](../README.md) · [← Previous: 7.3 Local LLMs](../07-ai-ml-workstation/03-local-llms-ollama-openwebui.md) · **08 — Productivity & Security** · [Next: 8.1 Keyboard →](01-keyboard-automation.md)
