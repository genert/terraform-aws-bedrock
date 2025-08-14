## Deployment

### Alternative #1 - IAM Role Anywhere

| Pros | Cons | Risks |
| ---- | ---- | ----- |
| Minimal infrastructure to stand up; no gateway/service layer required. | No native way to enforce per‑user or per‑team rate limits/quotas; only coarse account‑level service quotas. | Credential/private key compromise can lead to unthrottled spend until certificates or profiles are revoked. |
| Uses standard AWS IAM for authorization; aligns with least‑privilege policies on Bedrock actions. | Certificate lifecycle management is complex (issuance, secure storage, rotation, revocation/CRL) across many developer devices. | Difficult or slow off‑boarding if revocation workflows or device retrieval lag. |
| Direct path from IDE/tools to Bedrock; lowest latency path (no extra hop). | Limited IDE/tooling support for Role Anywhere vs SSO; more custom setup per user and OS trust store differences. | Certificates/private keys may be stored on unmanaged or insecure endpoints, increasing attack surface. |
| Works without building/operating an intermediary service; lower ops burden. | Harder to centrally enforce model allowlists, max token limits, and safety policies across teams. | Inconsistent policy application across teams can cause compliance drift. |
| Clear principal identity in CloudTrail for audit (per certificate/profile). | Usage attribution and cost controls are weaker; no straightforward way to block bursts besides global quotas. | Unexpected cost spikes if a tool loops or a script misbehaves (denial‑of‑wallet). |
| Multi‑account capable via trust anchors and profiles. | Difficult to integrate device posture/MFA checks at the API call; relies on front‑end discipline. | Org‑wide controls (e.g., SCPs) may have to stay permissive, increasing blast radius. |

### Alternative #2 - API GW + Lambda

| Pros | Cons | Risks |
| ---- | ---- | ----- |
| Central control point enables per‑user/per‑team rate limits and quotas (API Gateway usage plans, Redis token buckets). | Additional latency overhead compared to direct calls; potential cold starts in Lambda. | Gateway becomes a single point of failure if not deployed with proper HA and autoscaling. |
| Strong authentication and authorization via OIDC/JWT or IAM Identity Center; easy off‑boarding. | Operational overhead to build/operate gateway, manage deployments, and monitor health. | Misconfiguration of auth/authorization could expose models or data. |
| Consistent policy enforcement: model allowlists, max tokens, safety/guardrail settings across the org. | Additional cost for API Gateway, Lambda, WAF, Redis/ElastiCache, and logging/metrics. | Over‑logging or storing sensitive payloads could create compliance/privacy risks. |
| Robust observability: centralized logging, metrics, per‑user attribution, anomaly detection, and alerts. | Key/secret/JWT distribution to IDEs must be handled (if using API keys or short‑lived tokens). | Rate limiter bugs can either block legitimate usage or fail open and allow costly bursts. |
| Easy, fast revocation: disable a user/team or API key to immediately block access. | Throughput limits and usage plans must be tuned; potential 429s if misconfigured. | WAF/edge configurations may inadvertently block valid traffic if rules are too strict. |
| Security layering: AWS WAF, SCP to deny direct Bedrock, VPC + PrivateLink for private egress. | Requires building or configuring IDE plugins/CLI to target the gateway instead of Bedrock directly. | Vendor/architecture lock‑in to the gateway pattern; migration requires coordination. |
| Future‑proof abstraction: swap models/providers or add caching/retries/circuit breakers without client changes. | Additional CI/CD and testing complexity for the gateway’s policy logic. | Regional/data residency mistakes if the gateway is deployed in the wrong region(s). |

