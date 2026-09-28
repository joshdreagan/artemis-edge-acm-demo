# Edge computing at fleet scale: Inside the hub-of-hubs GitOps architecture for AMQ Broker on OpenShift

Managing messaging infrastructure across two or three edge clusters is straightforward. Managing it across 2,000 single-node OpenShift clusters spanning three geographic regions is a different engineering problem entirely. The cluster topology changes, the deployment model changes, and the observability strategy changes.

In this post, I walk through a production-grade GitOps architecture that solves this problem. I show how Red Hat Advanced Cluster Management for Kubernetes (RHACM), Argo CD, and a hub-of-hubs topology work together to install operators, place workloads, and onboard new clusters — without touching each one individually. By the end, you will understand the three-tier cluster topology, how GitOps pushes configuration across the fleet, and how a new Red Hat OpenShift Container Platform cluster joins the fleet and receives its full stack automatically.

This architecture applies anywhere you need reliable messaging at remote sites: retail stores, manufacturing floors, telecommunications edge nodes, or energy substations. An [open source demo repository](https://github.com/tosin2013/artemis-edge-acm-demo) is available if you want to explore the implementation after reading.

## The three cluster tiers in a hub-of-hubs topology

The architecture uses three tiers of clusters, each with a distinct responsibility. Understanding these tiers is essential before diving into the GitOps layers that connect them.

### What each tier owns

**Tier 0 — Global Hub.** This cluster runs RHACM with the Multicluster Global Hub Operator, the fleet Argo CD instance (specifically the `hubs` ApplicationSet), and a fleet-level Thanos Query for aggregated observability. It has no Red Hat AMQ Broker instance, no Keycloak — its job is pure governance and fleet orchestration.

**Tier 1 — Regional hubs (east, central, west).** Each regional hub runs its own RHACM instance managing local edge clusters, an Argo CD `field-content` Application, an AMQ Broker hub instance (`hub-01`), Keycloak for OpenID Connect (OIDC) authentication, and ACM Governance Policies in the `edge-broker-policies` namespace. Each hub also runs Multicluster Observability (MCO) with Thanos and Grafana for regional metrics.

**Tier 2 — Edge single-node OpenShift (SNO) clusters.** These are lightweight single-node clusters deployed at remote sites. Each runs one AMQ Broker instance configured for Advanced Message Queuing Protocol Secure (AMQPS) federation back to its regional hub's `hub-01` broker. Provisioning uses Hive for cloud environments or the Assisted Installer for bare-metal sites.

One detail worth calling out: every RHACM hub sees itself as `local-cluster`. The Global Hub's `local-cluster` is the Global Hub. A regional hub's `local-cluster` is that regional hub. This avoids confusion when you run `oc get managedclusters` and see `local-cluster` in the output.

*[Diagram 1: Three-layer GitOps architecture showing Global Hub pushing config to regional hubs, regional hubs enforcing ACM Policies onto edge SNOs, and edge brokers federating messages back to regional hub brokers.]*

## GitOps layer 1 — the Global Hub pushes configuration to regional hubs

The first GitOps layer connects Tier 0 to Tier 1. On the Global Hub, the `hubs` ApplicationSet uses a `clusters` generator with the label selector `hub-tier: regional`. When I import a regional hub as a ManagedCluster and apply that label, the ApplicationSet automatically creates a `hub-config-<name>` Argo CD Application targeting that hub.

Each Application pushes a Kustomize overlay from `fleet-gitops/hubs/overlays/<region>/` containing:

- AMQ fleet ApplicationSet definition
- AMQ Broker operator subscription policy
- GitOps operator policy
- User-workload monitoring policy
- Observability metrics allowlist
- GitOpsCluster registration

The key principle here is that regional hubs receive their baseline configuration from Git, not from manual `oc apply` commands. I commit to Git, Argo CD syncs, and the regional hub converges to the desired state. The `field-content` Application on each regional hub is itself a Helm chart with 39 templates across 7 sync waves, handling the full local stack — operators, messaging, authentication, observability, and workshop resources.

## GitOps layer 2 — ACM Governance installs operators across all spokes

The second layer handles operator installation on every edge cluster. This is where RHACM Governance does the heavy lifting, replacing what would otherwise be hundreds of individual `oc apply` sessions.

### How Helm generates ACM policies

The hub's Helm chart includes a template (`templates/edge-broker-policies.yaml`) that Argo CD's `field-content` Application syncs. This template renders one Policy and one Placement per entry in the `edgeBrokers[]` array in `values.yaml`.

The operator policy — `policy-amq-broker-operator-sno` — targets every cluster carrying the label `sites: edge-sno`. It enforces three resources: a Namespace, an OperatorGroup, and an AMQ Broker Subscription. The remediation action is `enforce`, not `inform`, so RHACM creates these resources on each matching cluster automatically. I can verify compliance from the hub without needing `oc login` access to each SNO:

```bash
oc get policy -n edge-broker-policies
```

```text
NAME                             REMEDIATION ACTION   COMPLIANCE STATE
policy-amq-broker-operator-sno   enforce              Compliant
policy-amq-broker-spoke-01       enforce              Compliant
policy-amq-broker-spoke-02       enforce              Compliant
policy-amq-broker-spoke-03       enforce
```

### Placement selectors target the right clusters

Placements in the `edge-broker-policies` namespace use label-based selection. Each spoke cluster carries labels assigned at provisioning time: `sites: edge-sno`, `fleet: amq`, and `edge-region: <region>`. A Placement for spoke-03 (CT) stays `NoManagedClusterMatched` until a CT cluster actually joins — this is expected, not an error.

One architectural insight worth highlighting: the `ztp-policies` ApplicationSet is intentionally unbound. Binding it would sync the full hub Helm chart to every SNO. Edge broker configuration flows through ACM Policies, not Argo CD Applications on the spokes.

## GitOps layer 3 — placing workloads on selected clusters

The third layer controls which specific broker configurations land on which specific clusters. Per-region broker Policies — such as `policy-amq-broker-spoke-01` targeting only `sno-edge-01` (NY) and `policy-amq-broker-spoke-02` targeting only `sno-edge-02` (NJ) — each contain a ConfigurationPolicy with the `ActiveMQArtemis` custom resource (CR) for that region.

Each CR includes AMQP federation configuration pointing back to the regional hub's `hub-01` broker. The Placement plus Policy combination achieves precise workload targeting: this specific broker configuration goes to this specific cluster. The result is a regional message aggregation topology where every SNO broker federates messages to its regional hub over AMQPS.

At scale, this per-entry approach has limits. For hundreds or thousands of sites, PolicyGenerator — a Kustomize plugin — replaces explicit Helm entries with template-driven policy generation. RHACM policy templates support `fromClusterClaim` functions that resolve per-cluster values at policy evaluation time, so a single policy template renders site-specific broker configurations without maintaining N separate policy objects.

## Adding a new OpenShift cluster to the fleet

This is where the architecture pays off. Adding a new cluster requires a Git commit, not a runbook.

### Provisioning with Hive or Assisted Installer

The `deploy-spokes.sh` script automates the full pipeline: pre-flight checks, credential extraction from the ACM secret, per-spoke namespace and secret creation, and an Argo CD Helm values patch enabling `spokeProvisioning`. When Argo CD syncs, it creates `ClusterDeployment` CRs (triggering Hive), `ManagedCluster` CRs (auto-importing into RHACM), and `KlusterletAddonConfig` CRs (enabling ACM add-ons). For on-premises bare-metal sites, the Assisted Installer replaces Hive — a Baseboard Management Controller (BMC) with Redfish support enables virtual media boot.

### The automatic configuration cascade

Once the new cluster joins with its labels (`sites: edge-sno`, `fleet: amq`, `edge-region: <region>`), three things happen without manual intervention:

1. **ACM Governance Placement re-evaluates.** The operator policy immediately matches the new cluster, and RHACM enforces the AMQ Broker operator installation.
2. **Per-region broker Policy matches.** If a broker Policy's Placement selector matches the new cluster's `edge-region` label, the broker CR is enforced automatically.
3. **MCO metrics collection begins.** The endpoint-observability-operator deploys a metrics-collector on the new cluster, and its AMQ metrics appear in regional Grafana.

The net result: I commit the cluster definition to Git, Hive provisions the cluster, the cluster auto-imports into RHACM, labels trigger policy evaluation, the operator and broker deploy, federation activates, and metrics flow — zero manual configuration on the edge cluster.

## Observability across the fleet

Monitoring thousands of broker instances requires a tiered approach. Each regional hub runs MCO, which aggregates Prometheus metrics from its managed clusters into Thanos with S3-compatible object storage for long-term retention.

A custom `observability-metrics-custom-allowlist` ConfigMap adds AMQ Broker metrics — `artemis_message_count`, `artemis_messages_added`, `artemis_connection_count`, and others — to the collection pipeline. On the SNO side, Policy `policy-sno-uwl-amq-metrics` enables user-workload monitoring on each SNO and places a spoke-side allowlist so the `uwl-metrics-collector` federates `artemis_*` series into Thanos.

A custom Grafana dashboard ("AMQ Broker - Artemis Edge") with a cluster dropdown filter lets me distinguish hub metrics from SNO metrics at a glance. At the fleet level, the Global Hub runs a Thanos Query that federates to regional Thanos gRPC endpoints, aggregating AMQ metrics across all regions without duplicating scrape load.

*[Diagram 2: RHACM governance boundary diagram showing Global Hub Argo CD managing fleet-level apps and policies, Regional Hub Argo CD managing five domain-specific apps, and ACM Policies flowing from regional hubs to edge SNOs.]*

## Scaling beyond the lab

RHACM 2.17 has been tested managing up to 3,500 SNO clusters from a three-node bare-metal hub (112 CPU cores, 512 GiB RAM per node), according to the [RHACM sizing guidelines](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html/install/sizing-your-cluster). Below roughly 2,000 clusters, a single hub (Mode 1) is typically sufficient. Above 2,000 clusters or across multiple geographic regions, evaluate hub-of-hubs (Mode 2) to distribute the management load.

For fleet parameterization, PolicyGenerator replaces per-site Helm entries with Kustomize-based policy generation and `fromClusterClaim` for per-cluster variable injection at evaluation time. The App of Apps pattern decomposes the monolithic Helm chart into domain-separated Argo CD child Applications, solving custom resource definition (CRD) ordering, independent lifecycles, and PolicyGenerator compatibility. For observability, at 1,800 clusters scraping 6 AMQ metrics every 30 seconds, tiered observability — regional scrape plus fleet-level federation — prevents bottlenecks.

## Wrap up

The three-layer GitOps model — Global Hub to regional hubs to edge SNOs — transforms fleet-scale messaging from a manual, per-cluster task into a declarative, Git-driven pipeline. New clusters receive their full stack through label-driven policy evaluation. ACM Policies handle enforcement (not Argo CD on spokes), Placements handle targeting, MCO handles observability, and Git remains the single source of truth.

As the fleet grows, the same architecture extends with PolicyGenerator for template-driven policy generation and the App of Apps pattern for domain separation. The patterns work at lab scale and at 3,500-cluster scale.

## Get started

Ready to deploy AMQ Broker at the edge with Red Hat Advanced Cluster Management for Kubernetes? Explore the full documentation:

- [Red Hat Advanced Cluster Management for Kubernetes 2.17 documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17)
- [Red Hat AMQ Broker 7.12 documentation](https://docs.redhat.com/en/documentation/red_hat_amq_broker/7.12)
- [Red Hat OpenShift GitOps (Argo CD) documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/)

## Learn more

- [Multicluster Global Hub (ACM 2.15)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.15/html/multicluster_global_hub/multicluster-global-hub) — fleet-of-fleets architecture
- [PolicyGenerator (ACM 2.16 Governance)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html/governance/policy-deployment) — Kustomize-based policy generation at scale
- [MCO Observability (ACM 2.17)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html/observability/observing-environments-intro) — custom metrics allowlist and Thanos storage
- [Preparing to install SNO (OCP 4.22)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_a_single_node/preparing-to-install-sno) — bare-metal hardware requirements

---

## Metadata

- **Meta title (56 chars):** Hub-of-hubs GitOps architecture for edge AMQ at scale
- **Meta description (158 chars):** Learn how Red Hat Advanced Cluster Management, Argo CD, and a hub-of-hubs topology deliver AMQ Broker to thousands of edge OpenShift clusters through GitOps.
- **URL slug:** edge-hub-of-hubs-gitops-amq-broker-openshift-fleet-scale
- **Diagram 1 alt text:** Three-tier hub-of-hubs architecture. Global Hub pushes GitOps config to regional hubs. Regional hubs enforce ACM Policies onto edge SNOs. Edge brokers federate messages back to regional hub brokers.
- **Diagram 2 alt text:** RHACM governance boundaries. Global Hub Argo CD manages fleet-level apps and policies. Regional Hub Argo CD manages five domain-specific apps. ACM Policies flow from regional hubs to edge SNOs.
- **Word count:** ~1,830
