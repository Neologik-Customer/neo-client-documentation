# Running Costs: neo-secure

This document sets out the expected monthly Azure running cost of a Neologik platform environment deployed with the **neo-secure** infrastructure profile in the UK South region. It explains what drives the cost and how spend is monitored, so that there are no surprises on the bill.

## Summary

neo-secure places every platform service behind private endpoints inside your virtual network, with Azure Bastion and a self hosted deployment runner VM. Inbound traffic passes through Application Gateway WAF_v2 with the web application firewall enabled, and the profile includes a jumpbox VM. It corresponds to Option A (Secure with Application Gateway) in the [Platform Architecture](../architecture/Platform-Architecture.md).

The cost is mostly fixed: compute, search, databases, networking and monitoring make up nearly all of it and vary little from month to month. The only usage based element is AI model consumption, which is capped per agent and per model deployment.

The cluster runs around the clock in both development and production.

| Environment | Monthly cost |
|---|---:|
| Development | £1,284 |
| Production | £1,337 |
| AI model consumption, on top | up to £50 |

All figures are pounds sterling per month at Microsoft pay as you go list prices in UK South, excluding VAT, which is added to your Azure invoice. Azure is priced in US dollars; your GBP billed rate depends on Microsoft's exchange rate and your agreement (Enterprise Agreement, CSP or MCA), and is currently about 16% above these GBP list figures.

## Contents

1. [Cost breakdown](#cost-breakdown)
2. [Assumptions](#assumptions)
3. [What makes the cost vary](#what-makes-the-cost-vary)
4. [How spend is monitored](#how-spend-is-monitored)
5. [Ways to reduce the cost](#ways-to-reduce-the-cost)
6. [Price source](#price-source)

## Cost breakdown

| Item | Development | Production |
|---|---:|---:|
| AKS nodes (Standard_D2s_v6) | £289 | £289 |
| AKS control plane | Free | £54 |
| AKS node disks (Premium SSD P10) | £81 | £81 |
| NAT gateway (400 GB), load balancer, public IP addresses | £59 | £59 |
| Application Gateway WAF_v2 (minimum 1 instance) | £319 | £319 |
| Azure Bastion (Basic) | £102 | £102 |
| Runner VM (D2as_v7, business hours, 2 x P10 disks) | £52 | £52 |
| Jumpbox VM (D2as_v7, business hours, P10 disk) | £34 | £34 |
| Private endpoints (12) and private DNS zones (14) | £70 | £70 |
| Azure AI Search (Standard S1) | £181 | £181 |
| Azure SQL Database (Standard S0) | £14 | £14 |
| Container Registry (Premium) | £37 | £37 |
| Azure Monitor and Log Analytics | £41 | £41 |
| Cosmos DB, storage accounts, Key Vault, Bot Service, start/stop automation | £5 | £5 |
| **Fixed infrastructure total** | **£1,284** | **£1,337** |

Development and production differ only in the AKS control plane tier: Free in development, Standard with the uptime SLA in production.

## Assumptions

| Area | Assumption |
|---|---|
| AKS nodes | Standard_D2s_v6 (2 vCPU, 8 GB), always on. The autoscaler minimum is 2 system and 2 user nodes; the measured average on always on environments is about 4.6 nodes |
| Secure access | Azure Bastion Basic, always on. The runner VM (Standard_D2as_v7) and, the jumpbox VM shut down automatically each evening and are costed at business hours |
| Private networking | 12 private endpoints and 14 private DNS zones |
| Networking | NAT gateway for outbound traffic (400 GB processed a month, as measured), Standard load balancer, 3 static public IP addresses |
| Azure AI Search | Standard S1, 1 replica, 1 partition |
| Azure SQL Database | Standard S0 (10 DTU) |
| Container Registry | Premium (required for private endpoints) |
| Monitoring | Alert rules, managed Prometheus and 5 GB of Log Analytics ingestion a month with 30 day retention |
| Small items | Cosmos DB serverless chat history, 4 storage accounts, Key Vault, Bot Service on Teams and Web Chat, start/stop automation: about £5 in total |
| AI model consumption | Not in the fixed total. Budget up to £50 a month; actual use depends on the model chosen and on how much the agents are used |

## What makes the cost vary

* **Node count.** Each additional Standard_D2s_v6 node adds about £63 a month. The autoscaler adds nodes only under sustained load and removes them when load falls.
* **AI model consumption.** Billed per token by Azure AI Foundry. Each agent carries token limits and each model deployment has a throughput cap, so a single agent cannot consume unbounded capacity.
* **Document volume.** More documents mean more storage, more index capacity and more ingestion tokens. The search tier changes only if the index outgrows it, and only by agreement.
* **Log volume.** Log Analytics bills per GB ingested. The platform keeps ingestion low by default.

## How spend is monitored

* **Azure budget.** The deployment creates a monthly Azure Cost Management budget on your subscription, with notifications to Neologik support at 80% of forecast spend, 100% of actual spend and 120% of actual spend.
* **Neologik spend monitoring.** Our centralised analytics service reads your subscription's actual daily cost several times a day and alerts our support team automatically when a day's spend reaches three times the trailing 30 day average, or when the subscription's state changes.
* **Your own visibility.** The platform runs in your subscription, so you have full Azure Cost Management access at all times, and you can add budgets of your own with notifications to your finance or IT contacts.

Budgets are alerts, not caps: Azure never switches resources off. The budget amount is agreed with you at onboarding.

## Ways to reduce the cost

| Option | Effect | Trade off |
|---|---|---|
| Reserved instances for AKS nodes | 38% (1 year) or 61% (3 years) off node compute | Commitment term |
| AKS start/stop outside business hours | Saves around two thirds of node compute | Platform unavailable outside the schedule; suited to development only |
| Discounts under your Azure agreement | Varies | None |

## Price source

Unit prices come from the [Azure Retail Prices API](https://prices.azure.com/api/retail/prices) in GBP for UK South, retrieved on 24 September 2026. Microsoft publishes these GBP prices as a reference conversion of its US dollar prices, and they exclude taxes. Quantities are measured from running environments where stated. Microsoft revises prices from time to time; validate against the [Azure pricing calculator](https://azure.microsoft.com/en-gb/pricing/calculator/) and your own agreement before relying on a figure. Neologik licensing and professional services are priced separately in your Statement of Work and are not included here.
