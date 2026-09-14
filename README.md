![Microsoft Certified: Azure AI Cloud Developer Associate — Exam AI-200](assets/AI-200%20header.png)

# AI-200 study notes

Personal notes for **Exam AI-200: Developing AI Cloud Solutions on Azure**, which earns [Microsoft Certified: Azure AI Cloud Developer Associate](https://learn.microsoft.com/en-us/credentials/certifications/azure-ai-cloud-developer-associate/).

The exam tests whether you can design, build, deploy, secure, monitor, and troubleshoot AI solutions on Azure, with the weight on back-end services rather than model training.

This cert replaces [AZ-204](https://learn.microsoft.com/en-us/credentials/certifications/azure-developer/) (retiring 31 July 2026).

## Exam snapshot

| | |
| --- | --- |
| Exam | AI-200 |
| Credential | Azure AI Cloud Developer Associate |
| Level | Associate / Developer |
| Questions | ~40–60 |
| Duration | 120 minutes |
| Pass score | 700 / 1000 |
| Format | Multiple choice, case studies, interactive items |
| Cost | ~USD 165 (varies by region) |
| Renewal | Free annual assessment on Microsoft Learn |

Source: [exam-focus/exam-tips.md](exam-focus/exam-tips.md) and the [official AI-200 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200).

## What you need to be good at

- Azure SDKs (and third-party SDKs used on Azure)
- Azure data services, including vector databases
- Messaging and eventing
- Containerized apps on Azure
- Monitoring and troubleshooting
- Python

## Skills measured

| Domain | Weight |
| --- | --- |
| [Develop containerized solutions on Azure](#1-develop-containerized-solutions-on-azure-20–25) | 20–25% |
| [Develop AI solutions with Azure data services](#2-develop-ai-solutions-with-azure-data-services-25–30) | 25–30% |
| [Connect to and consume Azure services](#3-connect-to-and-consume-azure-services-20–25) | 20–25% |
| [Secure, monitor, and troubleshoot Azure solutions](#4-secure-monitor-and-troubleshoot-azure-solutions-20–25) | 20–25% |

### 1. Develop containerized solutions on Azure (20–25%)

- Build, store, version, and manage images in **Azure Container Registry** (including ACR Tasks)
- Deploy containers to **App Service**, including env vars and secrets
- Deploy to **Azure Container Apps**: environments, revisions, **KEDA** scaling
- Deploy and manage apps on **AKS** with manifests
- Inspect logs, events, and connectivity on AKS and Container Apps

### 2. Develop AI solutions with Azure data services (25–30%)

**Cosmos DB for NoSQL**

- Connect with the SDK, run queries
- Indexing, consistency, and RU cost
- Store embeddings and run vector similarity search
- Change feed processors

**Azure Database for PostgreSQL**

- Connect and query with SDKs
- Schema, indexes, data types
- **pgvector**: embeddings, semantic retrieval, RAG with metadata filters
- Size compute/memory/storage for vector workloads
- Connection tuning for throughput and latency

**Azure Managed Redis**

- Cache, expire, and invalidate data
- Vector indexing for similarity search

### 3. Connect to and consume Azure services (20–25%)

- **Service Bus**: queues, topics, subscriptions, dead-letter handling
- **Event Grid**: filters, custom events, retries
- **Azure Functions**: triggers, bindings, serverless APIs, deploy function apps

### 4. Secure, monitor, and troubleshoot Azure solutions (20–25%)

- **Key Vault**: store, retrieve, and rotate secrets
- **App Configuration**: store and retrieve app settings
- Distributed tracing with **OpenTelemetry**
- **KQL** queries over logs and metrics

## Repo layout

```
AI-200/
├── README.md                 # this file
└── exam-focus/
    └── exam-tips.md          # exam format and logistics
```

Notes will land under folders that match the four domains as they get written.

## Official links

- [Certification overview](https://learn.microsoft.com/en-us/credentials/certifications/azure-ai-cloud-developer-associate/)
- [AI-200 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200)
- [Exam sandbox](https://aka.ms/examdemo)
- [Schedule the exam (Pearson VUE)](https://learn.microsoft.com/en-us/credentials/certifications/azure-ai-cloud-developer-associate/)

## Disclaimer

These are personal notes for preparing for the exam. Community contributions are welcome.