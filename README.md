# ruleset

My ruleset for LLM apps.

# AGENTS.MD

This repository provides reusable agent instructions organized by profile. Each machine and project can select a different rule set without duplicating its source.

- **coding_basic**: [Deployment and synchronization guide](agentsmd/coding_basic/README.md) · [Rule file](agentsmd/coding_basic/AGENTS.md)

Clone the repository once, then link each agent or project to its selected `agentsmd/<profile>/AGENTS.md`. Profiles remain independent and share the same central Git checkout.

See the **coding_basic** guide for Linux, macOS, Windows, project linking, periodic synchronization, and agent-triggered updates.
