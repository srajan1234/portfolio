# Kumar Srajan — Engineering Portfolio

**Senior Software Engineer · 5+ years · Noida, India**
Backend & distributed systems — Java, Spring Boot, Microservices, Kafka, AWS, Docker, Kubernetes.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-kumar--srajan-0A66C2?logo=linkedin&logoColor=white)](https://linkedin.com/in/kumar-srajan-9876091a7)
[![GitHub](https://img.shields.io/badge/GitHub-srajan1234-181717?logo=github&logoColor=white)](https://github.com/srajan1234)
[![Email](https://img.shields.io/badge/Email-saju757081%40gmail.com-EA4335?logo=gmail&logoColor=white)](mailto:saju757081@gmail.com)

---

## About

Senior Software Engineer with 5+ years architecting scalable, distributed backend systems. I design and ship
production services in **Java / Spring Boot** with **microservices**, event streaming (**Kafka**), polyglot
persistence (**PostgreSQL, MongoDB, Cassandra, Redis, DynamoDB**), and **AWS / GCP** on **Docker & Kubernetes**.
Track record includes a 60% reduction in operational cost (DynamoDB → PostgreSQL migration) and a Kafka broadcast
redesign that cut campaign failure rates by 60% across millions of messages.

This repository is my **systems-design portfolio**: for each project below it documents the
**High-Level Design (HLD)**, **Low-Level Design (LLD)**, and **key runtime flows** — architecture diagrams, data
models, API surfaces, and design trade-offs. Source code is kept private; these docs show *how* the systems are
built, not a copy-paste of the implementation. Two entries (**IAG Cargo Portal** and **Retaint**) are **sanitized
case studies** of proprietary work — architecture and design only, with internal identifiers, credentials, and
infrastructure details deliberately omitted.

> 🔗 **Live interactive portfolio:** https://claude.ai/code/artifact/c177d054-120e-4ef4-9f44-a3947b71ef0f

---

## Projects

| Project | What it is | Core stack | Design docs |
|---|---|---|---|
| **[IAG Cargo Portal](./projects/iag-cargo-portal/)** 🛩️ | *(case study, current @ Coforge)* Modernizing a legacy airline-cargo portal (Java 8 monolith) into a **serverless AWS microservices** platform with an Angular micro-frontend — same Oracle DB via cross-account private networking. | Angular 21 · AWS Lambda · API Gateway · Java 21 · Python · DynamoDB · EventBridge · SAM/Terraform | [README](./projects/iag-cargo-portal/README.md) · [HLD](./projects/iag-cargo-portal/HLD.md) · [LLD](./projects/iag-cargo-portal/LLD.md) · [Flows](./projects/iag-cargo-portal/FLOWS.md) |
| **[Retaint](./projects/retaint/)** 📣 | *(case study)* Multi-tenant **omnichannel marketing-automation** platform — broadcast + event-triggered automations across email/SMS/WhatsApp/push/RCS/chat, segmentation & revenue analytics, moving millions of messages. | Play Framework · React · Python (Celery) · PostgreSQL · Redis · Cassandra · MongoDB | [README](./projects/retaint/README.md) · [HLD](./projects/retaint/HLD.md) · [LLD](./projects/retaint/LLD.md) · [Flows](./projects/retaint/FLOWS.md) |
| **[Vibe](./projects/vibe/)** | Three-sided emotional-wellness platform with a one-way **privacy wall** — individuals log emotions & get matched; businesses only ever see k-anonymized aggregates. | Java 21 · Spring Boot · React PWA · FastAPI ML · Postgres (2-schema) · Redis event bus | [README](./projects/vibe/README.md) · [HLD](./projects/vibe/HLD.md) · [LLD](./projects/vibe/LLD.md) · [Flows](./projects/vibe/FLOWS.md) |
| **[Linder](./projects/linder/)** | Tinder-style **job matching** — swipe, mutual-match, real-time chat. Mobile app + Spring Cloud microservices. | React Native/Expo · Spring Cloud (Gateway + Eureka + 5 services) · Postgres/Mongo/Redis/RabbitMQ · FCM | [README](./projects/linder/README.md) · [HLD](./projects/linder/HLD.md) · [LLD](./projects/linder/LLD.md) · [Flows](./projects/linder/FLOWS.md) |
| **[InfluenceHub](./projects/influencehub/)** | Influencer-marketing **marketplace** with **watermark-until-paid** escrow and GST invoicing. Microservices. | Java 17 · Spring Boot (3 services) · React · PostgreSQL (DB-per-service) · Redis · Cloudinary | [README](./projects/influencehub/README.md) · [HLD](./projects/influencehub/HLD.md) · [LLD](./projects/influencehub/LLD.md) · [Flows](./projects/influencehub/FLOWS.md) |
| **[CarTag](./projects/cartag/)** | Privacy-first **vehicle QR safety** platform — contact an owner via masked call / WhatsApp **without exposing their phone**, geofenced & time-boxed. | Java 17 · Spring Boot · React 19 · PostgreSQL · Exotel/Fast2SMS | [README](./projects/cartag/README.md) · [HLD](./projects/cartag/HLD.md) · [LLD](./projects/cartag/LLD.md) · [Flows](./projects/cartag/FLOWS.md) |
| **[SnapAds Pro](./projects/snapads-pro/)** | **Snapchat-ads SaaS** — OAuth account connect with an AES-GCM **token vault**, multi-step campaign orchestration, analytics. | Java 17 · Spring Boot · React 19 · PostgreSQL · Snapchat Marketing API | [README](./projects/snapads-pro/README.md) · [HLD](./projects/snapads-pro/HLD.md) · [LLD](./projects/snapads-pro/LLD.md) · [Flows](./projects/snapads-pro/FLOWS.md) |
| **[Party Games](./projects/inhousegames/)** | **Server-authoritative** party-games hub (Tambola / Never-Have-I-Ever / Truth-or-Dare) with **SSE** realtime and AI-generated questions. | Java 17 · Spring Boot · React (TS) · SSE · PostgreSQL · Claude API | [README](./projects/inhousegames/README.md) · [HLD](./projects/inhousegames/HLD.md) · [LLD](./projects/inhousegames/LLD.md) · [Flows](./projects/inhousegames/FLOWS.md) |
| **[Trading Assistant](./projects/trading-assistant/)** | **NSE intraday-options & BTST** dashboard — live option chains, technical-indicator engine, real-time scanners. | Node.js · Express · Vanilla JS · Puppeteer · Upstox/NSE/Yahoo APIs | [README](./projects/trading-assistant/README.md) · [HLD](./projects/trading-assistant/HLD.md) · [LLD](./projects/trading-assistant/LLD.md) · [Flows](./projects/trading-assistant/FLOWS.md) |

---

## Skills → demonstrated in

| Skill area | Technologies | Shown in |
|---|---|---|
| **Backend & APIs** | Java 17/21, Spring Boot, Spring Cloud, Play Framework, REST, Hibernate/JPA | IAG Cargo, Retaint, Vibe, Linder, InfluenceHub, CarTag, SnapAds Pro, Party Games |
| **Serverless & AWS** | AWS Lambda, API Gateway, EventBridge, SQS, DynamoDB, S3, SAM, Terraform | IAG Cargo |
| **Microservices & API Gateway** | Spring Cloud Gateway, Netflix Eureka, BFF pattern, DB-per-service | IAG Cargo, Linder, InfluenceHub, Retaint |
| **Event-driven systems** | Kafka, EventBridge, SQS, Redis pub/sub, RabbitMQ, SSE, Celery | IAG Cargo, Retaint, Vibe, Linder, InfluenceHub, Party Games |
| **Real-time** | WebSocket/STOMP, Server-Sent Events, presence & typing | Vibe, Linder, Party Games |
| **Polyglot persistence** | PostgreSQL, MongoDB, Redis | Linder (Postgres+Mongo+Redis), Vibe, all Spring services |
| **Security & auth** | JWT access + rotating refresh, AES-GCM token vaults, OAuth2 | Vibe, SnapAds Pro, CarTag, Linder |
| **Third-party integration** | Snapchat/Upstox/Exotel/Cloudinary/SendGrid/Firebase/Claude | SnapAds Pro, Trading Assistant, CarTag, InfluenceHub, Party Games |
| **Frontend** | React 18/19, Vite, TypeScript, React Native/Expo, PWA | Vibe, CarTag, SnapAds Pro, Linder, Party Games |
| **DevOps** | Docker, docker-compose, Railway/Vercel, deployment runbooks | Vibe, CarTag, InfluenceHub |

---

## Experience (summary)

| Role | Company | Period | Focus |
|---|---|---|---|
| Senior Software Engineer | Coforge — IAG Cargo (British Airways) | May 2025 – Present | **[IAG Cargo Portal](./projects/iag-cargo-portal/)** — legacy→serverless AWS microservices modernization; booking/tracking/pricing services; Track & Trace with AWS Glue + DynamoDB |
| Software Engineer | Quara AI Tech | Sep 2024 – Mar 2025 | DynamoDB → PostgreSQL migration (−60% cost); hybrid physical/online auction module |
| Software Engineer | Shiprocket | Feb 2022 – Sep 2024 | **[Retaint](./projects/retaint/)** omnichannel marketing-automation platform; Kafka broadcast redesign (−60% failures); Shopify-webhook automation |
| Software Engineer | Daffodil Software | Jan 2021 – Feb 2022 | Spring Boot + Hibernate REST APIs for the PAS digital platform |

**B.Tech, Information Technology — AKTU, Greater Noida (2017–2021)**

---

## How to read this repo

```
portfolio/
├── README.md              ← you are here (index + skills matrix)
├── projects/<name>/
│   ├── README.md          ← overview, features, stack, status
│   ├── HLD.md             ← system architecture, components, trade-offs
│   ├── LLD.md             ← modules, APIs, data model, algorithms
│   └── FLOWS.md           ← sequence diagrams of the key runtime flows
└── resume.pdf
```

All diagrams are authored in **Mermaid** and render natively on GitHub. Each project's design is documented
independently — start with a project's `README.md`, then dive into `HLD` → `LLD` → `FLOWS`.
