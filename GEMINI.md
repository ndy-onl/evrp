# EVRP Project DNA: High-Performance Sovereign Multi-Tenant SaaS Architecture

## Vision
The evrp platform is a sovereign, multi-tenant SaaS ERP based on Frappe Framework (Version 15), designed to serve independent customer databases through a single, shared container application stack on private enterprise hardware.

## Architecture: Multi-Tenancy
- **Strategy:** Frappe native DNS multi-tenancy.
- **Routing:** Coolify-managed Traefik v3 dynamically handles wildcard subdomains (`*.evrp.cloud`) and custom tenant domains.
- **Traffic Flow:** 
  `Internet -> Traefik (SSL) -> Frappe Nginx (Port 8080) -> Gunicorn (Port 8000) / Socket.IO (Port 9000)`

## Core Components
1. **Control Plane (`hub.evrp.cloud`):** 
   - Master site hosting `evrp_core`.
   - Manages onboarding, subscriptions, payment webhooks (Stripe), and monitoring.
2. **Tenant Plane:**
   - Single application stack serving isolated tenant databases.
   - MariaDB (Internal, no external exposure).
   - Redis cluster for caching and queues.

## SaaS Operations
- **Provisioning:** Asynchronous via Celery workers (`bench new-site`).
- **Sandbox/Demo:** Fast-cloning from `template.evrp.cloud` (DB replication + filesystem cloning) to provide 1-click trials.
- **Billing:** Stripe integration with automated maintenance mode toggling based on payment status.

## German Compliance Stack
- **Standard:** EN 16931 (E-Invoicing), GoBD (Record Immutability), DATEV (Financial Export).
- **Modules:** `eu_einvoice`, `erpnext_germany`, `erpnext_datev`.
- **Localization:** Europe/Berlin timezone, SKR03/SKR04 charts, localized formats.

## Development & Deployment
- **Repo 1 (`evrp`):** Infrastructure, Docker Compose, Traefik labels.
- **Repo 2 (`evrp-core`):** Custom Frappe App logic, compliance hooks, SaaS automation.
- **CI/CD:** Coolify triggers builds from `evrp` repository.

## Automated Bank Reconciliation (Postbank/FinTS)
- **Strategy:** Direct FinTS/HBCI integration with asynchronous MFA (PSD2/BestSign) handling.
- **Architecture:** 
  - Overrides core `Bank Account` and `Bank Transaction` classes.
  - Serializes session state (`client.pause()`) when MFA is required.
  - Broadcasts challenge to frontend via WebSockets.
- **Security:** 
  - Single-threaded `banking_sync` queue (Concurrency = 1).
  - Redis locks to prevent IP-based anti-fraud locks from banks.
- **Matching Engine:** Regex-based SEPA reference scanning and IBAN matching with automated Payment Entry submission.

## Automated Invoice Ingestion (AI-OCR Pipeline)
- **Edge Node:** Physical USB scanner automation (SANE/scanbd) on local Linux nodes.
- **Email Ingestion:** IMAP attachment scraper with metadata filtering (size/MIME/pattern).
- **AI OCR Server:** Local VLM (Qwen2-VL) using `instructor` for schema-enforced JSON extraction.
- **Frappe Verification Queue:** 
  - `Invoice Sandbox Queue`: Intermediate verification layer for human-in-the-loop validation.
  - Side-by-side UI rendering PDF and extracted fields.
  - Automated mapping to ERPNext `Purchase Invoice`.
