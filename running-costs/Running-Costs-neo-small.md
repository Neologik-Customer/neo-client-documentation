# Running Costs: neo-small

This document sets out the expected monthly Azure running cost of a Neologik platform environment deployed with the **neo-small** infrastructure profile in the UK South region. It explains what drives the cost and how spend is monitored, so that there are no surprises on the bill.

## Summary

neo-small uses public service endpoints protected by firewall rules and Microsoft Entra ID, NGINX ingress and a GitHub hosted deployment runner. It is the lowest cost profile and is typically used for development, proof of value and cost sensitive production. It corresponds to Option C (Small) in the [Platform Architecture](../architecture/Platform-Architecture.md).

The cost is mostly fixed: compute, search, databases, networking and monitoring make up nearly all of it and vary little from month to month. The only usage based element is AI model consumption, which is capped per agent and per model deployment.

In development the AKS cluster is stopped outside business hours (08:00 to 18:00, Monday to Friday); Neologik activates the schedule after the deployment and confirms that it runs. Production runs around the clock.

| Environment | Monthly cost |
|---|---:|
| Development, start/stop on (proof of value) | £458 |
| Development, start/stop off | £673 |
| Production (always on) | £727 |
| AI model consumption, on top | about £50 on average for one use case with its proof of value users |

All figures are pounds sterling per month at Microsoft pay as you go list prices in UK South, excluding VAT, which is added to your Azure invoice. Azure is priced in US dollars; your GBP billed rate depends on Microsoft's exchange rate and your agreement (Enterprise Agreement, CSP or MCA), and is currently about 16% above these GBP list figures.

## Contents

1. [Cost breakdown](#cost-breakdown)
2. [Assumptions](#assumptions)
3. [What makes the cost vary](#what-makes-the-cost-vary)
4. [How spend is monitored](#how-spend-is-monitored)
5. [Ways to reduce the cost](#ways-to-reduce-the-cost)
6. [Price source](#price-source)

## Cost breakdown

| Item | Development, start/stop on | Development, start/stop off | Production |
|---|---:|---:|---:|
| AKS nodes (Standard_D2s_v6) | £121 | £289 | £289 |
| AKS control plane | Free | Free | £54 |
| AKS node disks (Premium SSD P10) | £34 | £81 | £81 |
| NAT gateway (400 GB), load balancer, public IP addresses | £59 | £59 | £59 |
| Azure AI Search (Standard S1) | £181 | £181 | £181 |
| Azure SQL Database (Standard S0) | £14 | £14 | £14 |
| Container Registry (Standard) | £15 | £15 | £15 |
| Azure Monitor and Log Analytics | £30 | £30 | £30 |
| Cosmos DB, storage accounts, Key Vault, Bot Service, start/stop automation | £5 | £5 | £5 |
| **Fixed infrastructure total** | **£458** | **£673** | **£727** |

Production runs the cluster around the clock and uses the Standard AKS control plane tier with the uptime SLA; development stops outside business hours and uses the Free tier.

### Validation against actual spend

A neo-small development environment in UK South, with start/stop in operation, was billed about £532 a month in September 2026, which is £458 at list prices. The quantities in this document (node hours, disks, network traffic, log volume) are taken from that environment. Development environments of this profile in UK South whose clusters ran around the clock were billed £759 to £766 a month in August 2026, consistent with the start/stop off figure of £673 at list prices plus the same billing uplift. Resources a customer adds outside the platform, such as Microsoft Fabric, Azure Maps or Microsoft Defender for Cloud, are billed on top.

## Assumptions

| Area | Assumption |
|---|---|
| AKS nodes | Standard_D2s_v6 (2 vCPU, 8 GB). The autoscaler minimum is 2 system and 2 user nodes and it adds nodes under load. Measured averages: about 6.5 nodes while a development cluster runs, about 4.6 nodes on an always on cluster |
| AKS running hours | Development: 08:00 to 18:00, Monday to Friday (about 217 hours a month); the node disks exist only while the cluster runs. Production: around the clock (730 hours) |
| Networking | NAT gateway for outbound traffic (400 GB processed a month, as measured), Standard load balancer, 3 static public IP addresses |
| Azure AI Search | Standard S1, 1 replica, 1 partition |
| Azure SQL Database | Standard S0 (10 DTU) |
| Container Registry | Standard |
| Monitoring | Alert rules, managed Prometheus and 2 GB of Log Analytics ingestion a month with 30 day retention |
| Small items | Cosmos DB serverless chat history, 4 storage accounts, Key Vault, Bot Service on Teams and Web Chat, start/stop automation: about £5 in total |
| AI model consumption | Not in the fixed total. A proof of value runs as a development environment with start/stop on; one use case with its proof of value users averages about £50 a month. Beyond a proof of value, use depends on the model chosen and on how much the agents are used |

## What makes the cost vary

* **Cluster hours.** Keeping a development cluster running outside business hours is the largest single change, shown in the validation section above.
* **Node count.** Each additional Standard_D2s_v6 node adds about £19 a month in development (business hours) and about £63 in production. The autoscaler adds nodes only under sustained load and removes them when load falls.
* **AI model consumption.** Billed per token by Azure AI Foundry. Each agent carries token limits and each model deployment has a throughput cap, so a single agent cannot consume unbounded capacity.
* **Document volume.** More documents mean more storage, more index capacity and more ingestion tokens. The search tier changes only if the index outgrows it, and only by agreement.
* **Log volume.** Log Analytics bills per GB ingested. The platform keeps ingestion low by default.

## How spend is monitored

* **Azure budget.** The deployment creates a monthly Azure Cost Management budget on your subscription, with notifications to Neologik support at 80% of forecast spend, 100% of actual spend and 120% of actual spend.
* **Neologik spend monitoring.** Our centralised analytics service reads your subscription's actual daily cost several times a day and alerts our support team automatically when a day's spend reaches three times the trailing 30 day average, or when the subscription's state changes.
* **Your own visibility.** The platform runs in your subscription, so you have full Azure Cost Management access at all times, and you can add budgets of your own with notifications to your finance or IT contacts.

Budgets are alerts, not caps: Azure never switches resources off. The budget amount depends on whether start/stop is on:

| Environment | Budget | 80% of forecast | 100% of actual | 120% of actual | Expected spend |
|---|---:|---:|---:|---:|---|
| Development, start/stop on (proof of value) | £600 | £480 | £600 | £720 | about £508, including about £50 of AI use for a proof of value |
| Development, start/stop off | £800 | £640 | £800 | £960 | £673 plus AI use |
| Production (always on) | £800 | £640 | £800 | £960 | £727 plus AI use |

## Ways to reduce the cost

| Option | Effect | Trade off |
|---|---|---|
| Reserved instances for production AKS nodes | 38% (1 year) or 61% (3 years) off node compute | Commitment term. Not suited to development: a reservation bills every hour, so a node that runs only in business hours costs less on pay as you go |
| Discounts under your Azure agreement | Varies | None |

## Price source

Unit prices come from the [Azure Retail Prices API](https://prices.azure.com/api/retail/prices) in GBP for UK South, retrieved on 24 September 2026. Microsoft publishes these GBP prices as a reference conversion of its US dollar prices, and they exclude taxes. Quantities are measured from running environments where stated. Microsoft revises prices from time to time; validate against the [Azure pricing calculator](https://azure.microsoft.com/en-gb/pricing/calculator/) and your own agreement before relying on a figure. Neologik licensing and professional services are priced separately in your Statement of Work and are not included here.
