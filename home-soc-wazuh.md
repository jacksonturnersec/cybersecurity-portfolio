# Home SOC: Wazuh SIEM Deployment

## Overview

Deployed a Wazuh SIEM stack in a home lab environment to gain hands-on experience with centralized log collection, agent-based monitoring, and security event visibility — core building blocks of SOC analyst work.

## Lab Architecture

| Component | Role | OS | IP Address |
|---|---|---|---|
| Ubuntu-Wazuh | Wazuh manager + dashboard | Ubuntu 24.04.4 LTS | 192.168.56.2 |
| Kali Agent | Monitored endpoint | Kali GNU/Linux 2025.2 | 192.168.56.3 |

Both VMs run in VirtualBox on a host-only network, isolating the lab from the host network while allowing manager-to-agent communication.

**Stack:** Wazuh 4.7.5 (manager, indexer, dashboard on a single Ubuntu host)

## Deployment Process

1. Installed Wazuh manager, indexer, and dashboard on the Ubuntu host using the official all-in-one installation script.
2. Verified the dashboard was reachable over HTTPS and confirmed initial state — manager running with zero agents connected (see Figure 1).
3. Deployed the Wazuh agent on the Kali VM, configuring it to register and report to the manager at 192.168.56.2.
4. Confirmed agent registration and active status from the manager's Agents view.

## Verification

**Initial state (manager only):**
Dashboard confirmed the manager was running with no agents added — a clean baseline before deployment.

**Post-deployment state (agent connected):**
Within ~20 minutes of starting the agent deployment, the Kali agent registered successfully:

- Status: **Active**
- Agent coverage: **100%**
- Operating system: Kali GNU/Linux 2025.2
- Agent version: v4.7.5
- Last registered / most active agent: `kali-agent`

This confirms the manager-to-agent communication channel, agent registration process, and the dashboard's real-time visibility into endpoint status are all functioning correctly.

## What This Demonstrates

- **SIEM fundamentals:** understanding the manager/indexer/dashboard architecture and how raw endpoint data becomes centralized, searchable security telemetry.
- **Agent-based monitoring:** deploying and registering an endpoint agent, and validating connectivity end-to-end rather than just trusting a green light.
- **Network segmentation awareness:** running the lab on an isolated host-only network reflects how monitored environments are segmented from management/production networks in real deployments.
- **Baseline before/after verification:** capturing the "before" state (0 agents) and "after" state (1 active agent, 100% coverage) — the same before/after thinking used when validating any change to a monitored environment.

## Next Steps

- Generate test events on the Kali agent (file changes, login attempts) and review how they surface in Wazuh's Security Events module.
- Explore the Integrity Monitoring and Policy Monitoring modules against the Kali endpoint.
- Document a basic investigation using an event triggered on the agent — feeds directly into `investigation-report-01.md`.
