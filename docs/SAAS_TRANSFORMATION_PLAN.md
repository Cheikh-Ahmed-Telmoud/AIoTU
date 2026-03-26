# AIoTU SaaS Transformation Plan

This document proposes a practical path to evolve AIoTU into a multi-tenant SaaS platform that is production-ready for:

- IoT integrations (sensor fleets, gateways, device management)
- Satellite integrations (imagery, vegetation/moisture indices)
- ML services for yield prediction and disease detection
- Irrigation automation workflows

## 1) Target SaaS outcomes

### Product goals
- Serve multiple organizations (cooperatives, agribusinesses, ministries) from one platform.
- Support mixed connectivity contexts (online farms + low-bandwidth regions).
- Provide secure, auditable APIs for partner ecosystems.
- Deliver actionable recommendations (yield, disease risk, irrigation actions) with traceability.

### Non-functional goals
- **Availability:** 99.9% or better for core APIs and dashboard.
- **Scalability:** from pilot farms to national-scale deployments.
- **Security:** TLS everywhere, tenant isolation, least-privilege access, key rotation.
- **Observability:** measurable SLOs, centralized logs, per-tenant usage metrics.

## 2) SaaS reference architecture (evolution of current Django MVT)

###! Core control plane
- **API Gateway / Ingress** (rate limits, auth, request tracing).
- **Identity service** (OIDC/OAuth2 + RBAC + optional MFA).
- **Tenant service** (tenant provisioning, plan limits, feature flags).
- **Billing/metering service** (events for device count, API calls, storage, compute).

###! Data plane
- **Ingestion service** for MQTT/HTTP sensor payloads.
- **Stream buffer** (Kafka/Redpanda or RabbitMQ + durable queues).
- **Rules engine** for alerts and automation policies.
- **ML inference services** (yield, disease risk, irrigation recommendation).
- **Scheduler/orchestrator** (Celery Beat today; workflow orchestrator later if needed).

###! Storage plane
- **PostgreSQL + PostGIS** for relational/geospatial data.
- **Timeseries store** (optional in phase 2; e.g., TimescaleDB extension).
- **Object storage** (S3-compatible) for images and model artifacts.
- **Feature store (lightweight)** for reusable model features.

###! Experience layer
- Existing Django UI remains valid for v1 SaaS (fastest path).
- Add external API docs portal + tenant admin console.
- Add webhook subscriptions for partner systems.

## 3) Multi-tenant design decisions

### Tenant isolation models
1. **Shared DB, shared schema + tenant_id** (fastest to ship).
2. **Shared DB, schema-per-tenant** (better blast-radius isolation).
3. **DB-per-tenant** (highest isolation, higher ops cost).

Recommended path:
- Phase 1: shared schema with strict row-level security and scoped query managers.
- Phase 2: premium tenants can move to schema-per-tenant.

### Required implementation elements
- Add `tenant_id` to all tenant-owned entities (farms, plots, sensors, images, predictions, tasks).
- Enforce tenant context in middleware and DRF permission classes.
- Add per-tenant quotas:
  - Max devices
  - Max daily ingest events
  - Max image storage
  - Max monthly model runs

## 4) IoT integration blueprint

### Device onboarding and identity
- Device registry (serial, model, firmware, farm binding, lifecycle state).
- Auth options:
  - mTLS certificates (preferred for gateways)
  - HMAC API keys with rotating secrets (for constrained devices)
- Device shadows/twins for desired vs reported state.

### Protocol and payload standardization
- Support both:
  - MQTT topics: `tenant/{tenant}/farm/{farm}/device/{device}/telemetry`
  - HTTP ingestion endpoint for batch uploads from gateways
- Define versioned payload schemas (JSON Schema/Avro) with validation.
- Include message signatures, timestamp, nonce, and sequence for replay protection.

### Edge/gateway strategy
- Support Raspberry Pi class gateways with:
  - Local buffering (store-and-forward)
  - Compression and retry policies
  - Over-the-air config updates
- Add “offline-first” mode for intermittent networks.

## 5) Satellite integration blueprint

### Use cases
- Vegetation health (NDVI/EVI)
- Moisture stress proxies
- Disease risk augmentation (microclimate + spectral signatures)
- Field zoning for variable-rate irrigation

### Integration pattern
- Build a **satellite provider adapter layer** to avoid vendor lock-in.
- Per-plot AOI geometry stored in PostGIS.
- Scheduled jobs fetch scenes/tiles by AOI + date window.
- Run cloud masking + index extraction pipeline.
- Persist derived features by `tenant_id`, `plot_id`, `observation_date`.

### Data quality controls
- Keep scene metadata (cloud cover, spatial resolution, source, processing level).
- Mark missing/low-quality observations explicitly for model robustness.

## 6) ML services roadmap

### Yield prediction service
- Keep Random Forest as baseline for explainability.
- Introduce model registry (version, metrics, training window, feature list).
- Add confidence intervals and drift monitoring.

### Disease detection service
- Inputs: field images + weather + sensor context + satellite features.
- Start with “risk scoring” before full disease classification.
- Human-in-the-loop workflow for agronomist validation and label curation.

### Irrigation automation service
- Policy engine combining:
  - Soil moisture thresholds
  - ET/weather forecast
  - Crop growth stage
  - Water availability constraints
- Output actions:
  - Recommendation only (phase 1)
  - Semi-automated approval workflow (phase 2)
  - Full closed-loop control where regulation allows (phase 3)

## 7) Security and compliance baseline

- TLS/DTLS for all links; enforce modern cipher suites.
- Signed command channel for actuator control; idempotent commands.
- RBAC + audit logs for every sensitive operation.
- Secrets managed centrally; periodic key/cert rotation.
- Backup, restore, and disaster recovery runbooks tested quarterly.
- Security alignment targets:
  - ETSI EN 303 645
  - NISTIR 8259A

## 8) API productization for SaaS

Expose versioned APIs:
- `/v1/ingest/*` for telemetry
- `/v1/devices/*` for provisioning/state
- `/v1/plots/*` and `/v1/geodata/*`
- `/v1/predictions/yield`
- `/v1/predictions/disease-risk`
- `/v1/automation/irrigation/*`

Add:
- API keys/service accounts per tenant
- Webhooks for alerts and prediction-ready events
- OpenAPI specs + SDK generation

## 9) Delivery plan (90/180/360 days)

### 0–90 days (Foundation)
- Multi-tenant core (`tenant_id`, middleware, permissions).
- API hardening and versioning.
- IoT device registry + MQTT/HTTP ingestion contracts.
- Object storage for images and model artifacts.
- Baseline observability (metrics, logs, traces).

### 91–180 days (Intelligence)
- Satellite adapter MVP + AOI pipeline.
- Yield model registry + drift monitoring.
- Disease risk scoring pipeline (image + climate features).
- Alerting/rules engine with tenant-level policies.

### 181–360 days (Automation & Scale)
- Irrigation recommendation engine with approvals.
- Optional actuator integrations for closed-loop pilots.
- Billing and quota enforcement per plan tier.
- SRE hardening (autoscaling, chaos drills, DR tests).

## 10) Suggested Django implementation mapping

- New apps:
  - `tenancy`
  - `device_registry`
  - `ingestion`
  - `satellite`
  - `mlops`
  - `automation`
  - `billing`
- Keep existing modules but move cross-cutting logic (auth, tenancy, auditing) into shared services.
- Introduce domain events for loose coupling between apps.

## 11) KPIs to track after launch

- Time to onboard a tenant and first device
- Telemetry ingestion success rate
- Prediction latency p95
- Forecast accuracy by crop/region
- Alert precision/recall (especially disease risk)
- Water-use efficiency improvement
- Churn and feature adoption by tenant tier

## 12) Immediate next actions for this repository

1. Add tenancy primitives (Tenant model + tenant-aware query mixins).
2. Create ingestion contracts and validation tests.
3. Externalize image/model storage to S3-compatible backend.
4. Add satellite adapter interface and one concrete provider plugin.
5. Split ML inference into background task endpoints with explicit model versioning.
6. Add audit logging for commands/recommendations and user actions.

---

This roadmap intentionally preserves the current Django foundation while introducing SaaS-grade tenancy, security, interoperability, and automation in incremental phases.
