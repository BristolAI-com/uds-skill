# UDS Deployment Skill

An [Agent Skill](https://agentskills.io/) for deploying applications on [UDS (Unicorn Delivery Service) Core](https://uds.defenseunicorns.com/).

## What it does

Guides AI agents through the full UDS deployment lifecycle:

- Scaffold Helm charts, Zarf packages, and UDS bundles
- Configure Keycloak SSO with Authservice (Istio-level auth)
- Set up network policies and Istio gateway routing
- Deploy to local k3d dev clusters
- Manage test users via Keycloak Admin API
- Avoid common deployment gotchas

## Install

Copy `SKILL.md` into your agent's skills directory. Compatible with any agent that supports the [Agent Skills specification](https://agentskills.io/specification).

## What's UDS Core?

UDS Core is a secure Kubernetes platform that bundles:

| Application | Role |
|-------------|------|
| Istio | Service Mesh |
| Keycloak | Identity & Access Management |
| Authservice | Authorization |
| Pepr | UDS Policy Engine & Operator |
| Prometheus | Monitoring |
| Grafana | Dashboards |
| Loki | Log Aggregation |
| Vector | Log Collection |
| Falco | Runtime Security |
| Velero | Backup & Restore |

Apps deploy as Zarf packages inside UDS bundles, with automatic SSO, network policies, and monitoring integration via the UDS Package CR.
