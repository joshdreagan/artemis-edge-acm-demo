# Blog outline: Hub-of-hubs GitOps architecture for edge AMQ at fleet scale

**Template:** Technical Deep Dive
**Target audience:** Platform engineers, cluster admins
**Word count target:** 1,300–2,000 words
**SEO keywords:** edge computing, AMQ Broker, Red Hat Advanced Cluster Management, GitOps, hub-of-hubs, ArgoCD, Kustomize, Helm, OpenShift, fleet management

---

## Metadata

- **Proposed title:** Edge computing at fleet scale: Inside the hub-of-hubs GitOps architecture for AMQ Broker on OpenShift
- **Meta title (55 chars):** Hub-of-hubs GitOps architecture for edge AMQ at scale
- **Meta description (158 chars):** Learn how Red Hat Advanced Cluster Management, ArgoCD, and a hub-of-hubs topology deliver AMQ Broker to thousands of edge OpenShift clusters through GitOps.
- **URL slug:** edge-hub-of-hubs-gitops-amq-broker-openshift-fleet-scale

---

## Section-by-section outline

### Introduction
**~150 words**

- **Hook (what is in it for the reader):** Managing messaging infrastructure across two or three edge clusters is straightforward. Managing it across 2,000 single-node OpenShift clusters spanning three geographic regions is a different engineering problem entirely. This post walks through a production-grade GitOps architecture that solves it — showing how Red Hat Advanced Cluster Management (RHACM), ArgoCD, and a hub-of-hubs topology work together to install operators, place workloads, and onboard new clusters without touching each one individually.
- Establish the real-world challenge: edge messaging at retail, manufacturing, or telco scale
- State the three questions the post answers:
  1. How does the three-tier cluster topology work (Global Hub, Regional Hubs, Edge SNOs)?
  2. How does GitOps push operator installations and workload configurations across the fleet?
  3. How does a new OpenShift cluster join the fleet and receive its configuration automatically?
- Brief mention of the open source demo repository readers can explore after reading

---

### H2: The four cluster roles in a hub-of-hubs topology
**~250 words**

Covers the ACM Hub, ACM Spokes, ACM Hub of Hubs (Global Hub), and individual OpenShift clusters.

- **H3: What each tier owns**
  - **Tier 0 — Global Hub:** RHACM with Multicluster Global Hub Operator, fleet ArgoCD (the `hubs` ApplicationSet), fleet-level Thanos Query for aggregated observability, compliance Grafana. No AMQ Broker, no Keycloak — this tier is pure governance and fleet orchestration.
  - **Tier 1 — Regional Hubs (east, central, west):** RHACM managing local edge clusters, ArgoCD (`field-content` Application), AMQ Broker hub instance (`hub-01`), Keycloak for OIDC, edge-broker-policies namespace with Governance Policies, regional Multicluster Observability (MCO) with Thanos/Grafana.
  - **Tier 2 — Edge SNOs (single-node OpenShift):** Lightweight single-node clusters at remote sites. Each runs one AMQ Broker instance per region, configured for AMQP federation back to its regional hub's `hub-01` broker. Provisioned via Hive (cloud) or Assisted Installer (bare-metal).
  - **`local-cluster` — the self-reference:** Every RHACM hub sees itself as `local-cluster`. The Global Hub's `local-cluster` is the Global Hub; a regional hub's `local-cluster` is that regional hub. This avoids confusion when reading `oc get managedclusters` output.

- **[DIAGRAM 1: Three-layer GitOps architecture]**
  - *Description:* A vertical three-layer diagram. **Top layer (Tier 0):** Global Hub cluster with ArgoCD and `hubs` ApplicationSet; arrows labeled "push config overlay" point down to three regional hub icons. **Middle layer (Tier 1):** Three Regional Hub clusters (east, central, west), each with ArgoCD `field-content` and ACM Governance; arrows labeled "ACM Policies (enforce)" point down to groups of SNO icons. **Bottom layer (Tier 2):** Groups of edge SNO clusters per region (NY, NJ under east; IL, OH under central; CA, OR under west), each with a small AMQ Broker icon; dashed arrows labeled "AMQP federation" point up to their regional hub's AMQ `hub-01`.
  - Alt text: "Three-tier hub-of-hubs architecture. Global Hub pushes GitOps config to regional hubs. Regional hubs enforce ACM Policies onto edge SNOs. Edge brokers federate messages back to regional hub brokers."

---

### H2: GitOps layer 1 — the Global Hub pushes configuration to regional hubs
**~200 words**

- The `hubs` ApplicationSet on the Global Hub uses a `clusters` generator with label selector `hub-tier: regional`
- When a regional hub is imported as a ManagedCluster, the ApplicationSet automatically creates a `hub-config-<name>` ArgoCD Application targeting that hub
- Each Application pushes a Kustomize overlay from `fleet-gitops/hubs/overlays/<region>/` containing:
  - AMQ fleet ApplicationSet definition
  - AMQ Broker operator subscription policy
  - GitOps operator policy
  - User-workload monitoring policy
  - Observability metrics allowlist
  - GitOpsCluster registration
- Key point: regional hubs receive their baseline configuration from Git, not from manual `oc apply` commands — platform engineers commit to Git, ArgoCD syncs
- Mention that `field-content` on each regional hub is a Helm chart (39 templates, 7 sync waves) handling the local stack

---

### H2: GitOps layer 2 — ACM Governance installs operators across all spokes
**~250 words**

Shows how the ACM Hub installs the AMQ Broker operator across all spoke clusters using GitOps-driven policies.

- **H3: How Helm generates ACM policies**
  - The hub's Helm chart (`templates/edge-broker-policies.yaml`), synced by ArgoCD `field-content`, renders one Policy + Placement per entry in `values.yaml` `edgeBrokers[]`
  - `policy-amq-broker-operator-sno` — a single Policy that targets every cluster with label `sites: edge-sno`. It enforces three resources: a Namespace, an OperatorGroup, and an AMQ Broker Subscription. Remediation action is `enforce` (not `inform`), so RHACM creates these resources on each matching cluster automatically.
  - Compliance feedback: the hub shows `Compliant` status per cluster without needing `oc login` to each SNO

- **H3: Placement selectors target the right clusters**
  - Placements in namespace `edge-broker-policies` use label-based selection (`edge-region: NY`, `edge-region: NJ`, etc.)
  - Each spoke cluster carries labels assigned at provisioning time: `sites: edge-sno`, `fleet: amq`, `edge-region: <region>`
  - `spoke-03` (CT) Placement stays `NoManagedClusterMatched` until a CT cluster joins — this is expected, not an error
  - Key architectural insight: the `ztp-policies` ApplicationSet is intentionally unbound. Binding it would sync the full hub Helm chart to every SNO. Edge broker config goes through ACM Policies, not ArgoCD apps on the spokes.

---

### H2: GitOps layer 3 — placing workloads on selected clusters
**~200 words**

Shows how workloads (broker instances) land on specific clusters, not all clusters.

- Per-region broker Policies: `policy-amq-broker-spoke-01` targets only `sno-edge-01` (NY); `policy-amq-broker-spoke-02` targets only `sno-edge-02` (NJ)
- Each Policy contains a ConfigurationPolicy with the `ActiveMQArtemis` CR for that region, including AMQP federation configuration pointing back to the regional hub's `hub-01` broker
- The Placement + Policy combination achieves workload targeting: "this specific broker configuration goes to this specific cluster"
- Federation topology: each SNO broker federates messages to its regional hub over AMQPS, creating a regional message aggregation point
- At scale, this per-entry approach has limits — introduce PolicyGenerator (a Kustomize plugin) with `fromClusterClaim` for fleet parameterization at hundreds or thousands of sites (link forward to the scaling section)

---

### H2: Adding a new OpenShift cluster to the fleet
**~250 words**

Shows the full lifecycle of a new cluster joining the hub.

- **H3: Provisioning with Hive (cloud) or Assisted Installer (bare-metal)**
  - The `deploy-spokes.sh` script automates: pre-flight checks → credential extraction from ACM secret → per-spoke namespace and secrets → ArgoCD Helm values patch enabling `spokeProvisioning`
  - ArgoCD sync creates: `ClusterDeployment` (triggers Hive), `ManagedCluster` (auto-imports into RHACM), `KlusterletAddonConfig` (enables ACM add-ons)
  - For on-premises: Assisted Installer replaces Hive; a BMC with Redfish support enables virtual media boot

- **H3: The automatic configuration cascade**
  - Once the new cluster joins with its labels (`sites: edge-sno`, `fleet: amq`, `edge-region: <region>`), three things happen without manual intervention:
    1. **ACM Governance Placement re-evaluates** — the operator policy immediately matches the new cluster, and RHACM enforces the AMQ Broker operator installation
    2. **Per-region broker Policy matches** — if a broker Policy's Placement selector matches the new cluster's `edge-region` label, the broker CR is enforced
    3. **MCO metrics collection begins** — the endpoint-observability-operator deploys a metrics-collector on the new cluster; its AMQ metrics appear in regional Grafana
  - Net result: commit the cluster definition to Git → Hive provisions → cluster auto-imports → labels trigger policy evaluation → operator and broker deploy → federation activates → metrics flow — zero manual configuration on the edge cluster

---

### H2: Observability across the fleet
**~200 words**

- Regional MCO (Multicluster Observability) aggregates Prometheus metrics from each regional hub's managed clusters into Thanos with GCS-backed long-term storage
- Custom `observability-metrics-custom-allowlist` ConfigMap adds AMQ Broker metrics (`artemis_message_count`, `artemis_messages_added`, `artemis_connection_count`, etc.) to the collection
- User-workload monitoring on SNOs: Policy `policy-sno-uwl-amq-metrics` enables `enableUserWorkload: true` on each SNO and places a spoke-side allowlist so `uwl-metrics-collector` federates `artemis_*` into Thanos
- Custom Grafana dashboard ("AMQ Broker - Artemis Edge") with cluster dropdown filter distinguishes hub vs SNO metrics
- Fleet-tier observability: Global Hub runs a Thanos Query that federates to regional Thanos gRPC endpoints — aggregated AMQ metrics across all regions without duplicating scrape load

- **[DIAGRAM 2: RHACM governance boundary diagram]**
  - *Description:* A side-by-side comparison. **Left box (Global Hub):** ArgoCD root app with `apps/overlays/global-hub/` containing `app-operators` (subset) and `app-observability` (fleet-tier); below it, ACM scope showing `fleet-gitops/hubs/` pushing to regional hubs and `fleet-gitops/platform/` with policies for managed hubs. **Right box (Regional Hub):** ArgoCD root app with `apps/overlays/regional-hub/` containing five child apps (operators, messaging, auth, observability regional-tier, workshop); below it, ACM scope showing `fleet-gitops/amq/` pulling to SNOs and ACM Policies enforcing operator + broker config on managed SNOs. Arrows between the boxes show the `hubs` ApplicationSet push direction.
  - Alt text: "RHACM governance boundaries. Global Hub ArgoCD manages fleet-level apps and policies. Regional Hub ArgoCD manages domain-specific apps. ACM Policies flow from regional hubs to edge SNOs."

---

### H2: Scaling beyond the lab
**~150 words**

- RHACM 2.17 tested with 3,500 managed SNOs from a three-node bare-metal hub (112 CPU cores, 512 GiB RAM per node)
- Below ~2,000 clusters: single hub (Mode 1) is typically sufficient
- Above ~2,000 clusters or multi-region: evaluate hub-of-hubs (Mode 2) to distribute management load
- PolicyGenerator replaces per-site Helm entries with Kustomize-based policy generation and `fromClusterClaim` for per-cluster variable injection at evaluation time
- App of Apps pattern decomposes the monolithic Helm chart into domain-separated ArgoCD child Applications, solving CRD ordering, independent lifecycles, and PolicyGenerator compatibility
- Metric cardinality: at 1,800 clusters scraping 6 AMQ metrics every 30 seconds, tiered observability (regional scrape + fleet-level federation) prevents bottlenecks

---

### Wrap up
**~100 words**

- Restate the three-layer GitOps model: Global Hub → Regional Hubs → Edge SNOs
- Emphasize the zero-touch onboarding: new clusters receive their full stack through label-driven policy evaluation
- Reinforce the key architectural decisions: ACM Policies for enforcement (not ArgoCD on spokes), Placement for targeting, MCO for observability, Git as the single source of truth
- Forward-looking: as the fleet grows, the same architecture extends with PolicyGenerator and App of Apps decomposition

---

### Call to action
**~50 words**

Ready to deploy AMQ Broker at the edge with Red Hat Advanced Cluster Management? Explore the full documentation:

- [Red Hat Advanced Cluster Management for Kubernetes 2.17 documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17)
- [Red Hat AMQ Broker 7.12 documentation](https://docs.redhat.com/en/documentation/red_hat_amq_broker/7.12)
- [OpenShift GitOps (ArgoCD) documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/)

---

### Learn more (optional)
**~50 words**

- [Multicluster Global Hub (ACM 2.15)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.15/html/multicluster_global_hub/multicluster-global-hub) — fleet-of-fleets architecture
- [PolicyGenerator (ACM 2.16 Governance)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html/governance/policy-deployment) — Kustomize-based policy generation at scale
- [MCO Observability (ACM 2.17)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html/observability/observing-environments-intro) — custom metrics allowlist and Thanos storage
- [Preparing to install SNO (OCP 4.22)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_a_single_node/preparing-to-install-sno) — bare-metal hardware requirements

---

## Estimated word counts

| Section | Words |
|---|---|
| Introduction | ~150 |
| The four cluster roles in a hub-of-hubs topology | ~250 |
| GitOps layer 1 — the Global Hub pushes configuration to regional hubs | ~200 |
| GitOps layer 2 — ACM Governance installs operators across all spokes | ~250 |
| GitOps layer 3 — placing workloads on selected clusters | ~200 |
| Adding a new OpenShift cluster to the fleet | ~250 |
| Observability across the fleet | ~200 |
| Scaling beyond the lab | ~150 |
| Wrap up | ~100 |
| Call to action + Learn more | ~100 |
| **Total** | **~1,850** |

---

## Proposed diagrams

### Diagram 1: Three-layer GitOps architecture
- **Format:** SVG or high-resolution PNG
- **Layout:** Three horizontal tiers, top-to-bottom
- **Top (Tier 0):** Single Global Hub icon with ArgoCD logo and `hubs` ApplicationSet label. Three downward arrows labeled "push config overlay" to the middle tier.
- **Middle (Tier 1):** Three Regional Hub icons (east, central, west) each showing ArgoCD `field-content`, ACM Governance shield, and AMQ `hub-01` broker. Downward arrows labeled "ACM Policies (enforce)" to the bottom tier.
- **Bottom (Tier 2):** Grouped edge SNO icons under each regional hub (e.g., NY+NJ under east, IL+OH under central, CA+OR under west). Each SNO has a small AMQ Broker icon. Dashed upward arrows labeled "AMQP federation" connect back to regional hub brokers.
- **Color coding:** Red Hat red for RHACM components, ArgoCD blue for GitOps, orange for AMQ Broker.
- **Alt text:** Three-tier hub-of-hubs architecture. Global Hub pushes GitOps config to regional hubs. Regional hubs enforce ACM Policies onto edge SNOs. Edge brokers federate messages back to regional hub brokers.

### Diagram 2: RHACM governance boundary diagram
- **Format:** SVG or high-resolution PNG
- **Layout:** Two large side-by-side boxes with connecting arrows
- **Left box — Global Hub:** ArgoCD root app (`apps/overlays/global-hub/`) listing `app-operators (subset)` and `app-observability (fleet-tier)`. Below: ACM scope listing `fleet-gitops/hubs/` and `fleet-gitops/platform/`. A horizontal arrow labeled "`hubs` ApplicationSet" points right.
- **Right box — Regional Hub:** ArgoCD root app (`apps/overlays/regional-hub/`) listing five child apps: operators, messaging, auth, observability (regional-tier), workshop. Below: ACM scope listing `fleet-gitops/amq/` and `edge-broker-policies`. Downward arrows labeled "ACM Policies" point to small SNO icons below.
- **Boundary lines:** Dashed borders around each box emphasize that each tier has its own ArgoCD instance and its own ACM governance scope.
- **Alt text:** RHACM governance boundaries. Global Hub ArgoCD manages fleet-level apps and policies. Regional Hub ArgoCD manages five domain-specific apps. ACM Policies flow from regional hubs to edge SNOs.

---

## Pre-submission checklist (from Red Hat editorial handbook)

- [ ] Sentence-case headings throughout (no title-case)
- [ ] First-person voice where appropriate
- [ ] All product names match Official Red Hat product names list (Red Hat Advanced Cluster Management for Kubernetes, Red Hat AMQ Broker, Red Hat OpenShift, ArgoCD)
- [ ] Acronyms spelled out on first use: RHACM, SNO, MCO, AMQ, CRD, AMQP
- [ ] Code formatted with monospace (no backticks in the final published version)
- [ ] Digital call to action linking to docs.redhat.com
- [ ] Alt text provided for all diagrams
- [ ] No jargon, hyperbole, or unsupported claims
- [ ] Sources cited are current (ACM 2.17, OCP 4.22)
