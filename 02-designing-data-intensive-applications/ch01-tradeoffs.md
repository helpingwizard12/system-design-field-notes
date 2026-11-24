# Chapter 1 — Trade-Offs in Data Systems Architecture

*DDIA 2nd ed. (Kleppmann & Riccomini) · Week 1 · Study notes*

> **Big idea:** In data systems there is rarely one "right" answer, only trade-offs. Learn the options, what each costs, and when each one wins.

---

## 1. Operational vs Analytical Systems

| | **Operational (OLTP)** | **Analytical (OLAP)** |
|---|---|---|
| Purpose | Run the business now | Understand the business |
| Reads | Point queries: a few records by key | Scan millions of rows → aggregates |
| Writes | Create/update/delete single records | Bulk load (ETL) or event streams |
| Users | App end users | Analysts, data scientists |
| Queries | Fixed, in app code | Ad-hoc, exploratory SQL / BI tools |
| Data represents | Current state | History of events |
| Size | GB–TB | TB–PB |

- **People:** backend engineers (operational side); business analysts (BI) and data scientists (ML) (analytical side); data engineers and analytics engineers connect the two.
- **Real-time / product analytics** (Pinot, Druid, ClickHouse): analytical queries served to *end users* with low latency.
- **Data warehouse:** a separate, read-only copy of data from all OLTP systems. It exists because of **data silos**, schemas that suit OLTP but not analytics, performance isolation, and security.
- **ETL / ELT:** extract → transform → load (or load first, then transform inside the warehouse). SaaS data comes in through connectors (Fivetran, Airbyte).
- **HTAP:** OLTP and OLAP in one system. Useful for cases like fraud detection, but it doesn't replace the warehouse, because one service's DB can't combine data from hundreds of others.
- **Data lake:** cheap object storage holding *any* files (Parquet, text, images) for data scientists (Pandas, Spark). The **sushi principle**: raw data is better.
- **Beyond the lake:** DataOps and governance (GDPR/CCPA), **stream processing** for responses within seconds, and **reverse ETL** (analytical outputs such as ML models fed back into the app).
- **Trend:** the bigger the scale, the more specialized the systems.

## 2. Systems of Record vs Derived Data
- **System of record (source of truth):** written to first, each fact stored once (normalized), and it wins any disagreement.
- **Derived data:** computed from another system and *rebuildable* if lost. Caches, indexes, materialized views, warehouses and ML models are all derived. It's redundant, but needed for fast reads and different views of the data.
- Whether a system is "record" or "derived" depends on **how you use it**, not on the tool.
- Hard part: **keeping derived data in sync** when the source changes (Ch 11–12).

## 3. Cloud vs Self-Hosting
- **Build vs buy:** do core competencies in-house and outsource commodity work.
- Spectrum: bespoke in-house → off-the-shelf **self-hosted** (on-prem or IaaS VM) → **cloud service / SaaS**.
- **Cloud wins** when you lack the expertise, or when **load varies** (elasticity; bursty analytics).
- **Self-hosting wins** when you have the expertise and a **predictable load**, or need deep tuning.
- **Cloud downsides:** no control (features, outages, debugging), **vendor lock-in**, price and product changes, sanctions, trusting the provider with data.

## 4. Cloud-Native Architecture
- Built *for* the cloud (Aurora, Spanner, Snowflake, BigQuery), not just moved onto a VM.
- **Layering:** higher-level services built on lower ones (Snowflake → S3 → disks).
- **Storage:** local VM disks are ephemeral. **Virtual disks (EBS)** run old software unchanged but add overhead and turn every I/O into a network call. Cloud-native databases keep small values in a fast tier and big blocks in **object storage**.
- **Separation of storage and compute** (disaggregation): scale each independently, at the cost of network transfer.
- **Multitenancy:** customers share hardware. Better utilization, but needs isolation (noisy neighbors, security).
- **Ops in the cloud era:** DBAs/sysadmins → DevOps/SRE (automation, ephemeral servers, frequent deploys, blameless postmortems). Capacity planning becomes **financial planning**, and performance optimization becomes **cost optimization**. Watch quotas. Integration, security, monitoring and root-cause analysis can't be outsourced.

## 5. Distributed vs Single-Node
- **Why distribute:** inherent distribution, cloud-service calls, fault tolerance, scalability, latency (being near users), elasticity, specialized hardware.
- **Costs:** a timeout means you don't know whether the request happened, so retries must be **idempotent**. Network calls are much slower than local calls. Debugging needs **observability** and tracing. Cross-service **consistency** becomes the app's problem.
- A single machine is often simpler, cheaper and even faster (DuckDB, SQLite). **Don't distribute until you have to.**
- **Microservices:** one purpose, one API, one team, one private DB. Mainly a solution to a *people* problem, so it's overhead for small teams. Costs: infrastructure per service, hard testing, API evolution (OpenAPI/gRPC).
- **Serverless / FaaS:** pay per execution, automatic scaling. Limits: time limits, cold starts, restricted runtimes. "Serverless" now also means "autoscaling, billed by usage."
- **Cloud vs supercomputing (HPC):** HPC runs trusted batch jobs that checkpoint and restart the whole job on a failure, with nodes physically close together. The cloud serves untrusted tenants, stays always available, and spans regions.

## 6. Data Systems, Law, and Society
- **GDPR, CCPA and the EU AI Act** constrain how personal data is used. Automated decisions (loans, hiring, policing) affect real people.
- **Right to be forgotten** vs immutable logs and derived data (ML training sets). This creates real engineering problems.
- Stored data carries **risk**: leaks, fines, government demands, and user safety.
- **Data minimization** (*Datensparsamkeit*): collect only what you need for a stated purpose, and delete it when it's no longer needed. Industry standards: PCI, SOC 2.

---

## Review Questions
1. A supermarket wants (a) customers to see their basket total at checkout and (b) the CFO to see revenue per store per month. Classify each as OLTP or OLAP, and give two properties from the comparison table that distinguish them.
2. Why do companies copy data into a separate warehouse instead of letting analysts query production databases? Give three reasons, and explain why HTAP doesn't remove the need.
3. A Redis cache and the PostgreSQL orders table disagree on an order's price. Which is right, and why? What general problem does this illustrate?
4. Your app's load is steady all year and your team knows Postgres well. Argue for self-hosting, then name two risks you'd accept by choosing a cloud service instead.
5. A payment call times out and the app retries, charging the user twice. Explain why this happens in distributed systems, what makes a retry safe, and why the book advises against going distributed unless necessary.
