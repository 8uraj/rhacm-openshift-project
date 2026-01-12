
# RHACM HyperShift Auto-Import Learning Repository

This repository demonstrates an **end-to-end automated workflow** to discover and import **OpenShift HyperShift Hosted Control Plane (HCP)** clusters into **Red Hat Advanced Cluster Management for Kubernetes (RHACM)** using the **Multi-Cluster Engine (MCE)** and **policy-based automation**.

The goal is to eliminate manual cluster onboarding and enable **scalable, governed, and repeatable cluster imports**.

---

## 📌 Architecture Overview

![RHACM Architecture](https://developers.redhat.com/sites/default/files/acm_cluster_lp_fig_1.1.png)
### Core Components

- **RHACM Hub Cluster**  
  Central governance cluster where policies are evaluated and managed clusters are controlled.

- **Multi-Cluster Engine (MCE) Hosting Cluster**  
  Runs MCE and HyperShift operators and hosts Hosted Control Plane (HCP) clusters.

- **Hosted (HCP) Clusters**  
  OpenShift clusters whose control plane runs as pods on the MCE cluster.

- **Managed Clusters**  
  Clusters registered and governed by RHACM (including MCE and hosted clusters).

---

## 🔁 High-Level Workflow

```text
HCP Cluster Provisioned
        ↓
HyperShift Add-on Detects HostedCluster
        ↓
DiscoveredCluster Created in RHACM
        ↓
RHACM Policy Evaluation
        ↓
Auto-Import Triggered
        ↓
Cluster Becomes ManagedCluster
````

---

## ✅ Prerequisites

### Required

* Red Hat OpenShift **4.16+** (4.18+ recommended)
* Red Hat Advanced Cluster Management **2.11+** (2.14+ recommended)
* Multi-Cluster Engine operator installed on the hosting cluster
* HyperShift operator configured on the MCE cluster
* `oc` CLI access to both hub and MCE clusters
* Basic understanding of Kubernetes, OpenShift, YAML, and RHACM policies

### Optional (Recommended)

* `clusteradm` CLI installed on the RHACM hub bastion host

---

## 🛠️ 1. Prepare the RHACM Hub Cluster

### 1.1 Install `clusteradm`

```bash
curl -L https://raw.githubusercontent.com/open-cluster-management-io/clusteradm/main/install.sh | bash
clusteradm version
```

---

### 1.2 Isolate Add-ons for the MCE Cluster

Some RHACM add-ons must run in a dedicated namespace to avoid conflicts.

**Add-ons requiring isolation**

* cluster-proxy
* managed-serviceaccount
* work-manager

#### Create `AddOnDeploymentConfig`

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnDeploymentConfig
metadata:
  name: addon-ns-config
  namespace: multicluster-engine
spec:
  agentInstallNamespace: open-cluster-management-agent-addon-discovery
```

Apply:

```bash
oc apply -f addon-deployment-config.yaml
```

---

#### Patch ClusterManagementAddOns

Example for `work-manager` (repeat for others):

```yaml
configs:
- group: addon.open-cluster-management.io
  name: addon-ns-config
  namespace: multicluster-engine
  resource: addondeploymentconfigs
  type: Placements
```

Verify:

```bash
oc get deployment -n open-cluster-management-agent-addon-discovery
```

---

### 1.3 Create Custom KlusterletConfig

```yaml
apiVersion: config.open-cluster-management.io/v1alpha1
kind: KlusterletConfig
metadata:
  name: mce-import-klusterlet-config
spec:
  installMode:
    type: noOperator
  noOperator:
    postfix: mce-import
```

---

### 1.4 (Optional) Label Resources for Backup

```bash
oc label clustermanagementaddon work-manager \
  cluster.open-cluster-management.io/backup=true
```

---

## 🚀 2. Import the MCE Cluster into RHACM

### 2.1 Create ManagedCluster

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: hc-site1-lab
  annotations:
    agent.open-cluster-management.io/klusterlet-config: mce-import-klusterlet-config
spec:
  hubAcceptsClient: true
```

---

### 2.2 Create auto-import-secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: auto-import-secret
  namespace: hc-site1-lab
stringData:
  token: sha256~<YOUR_MCE_TOKEN>
  server: https://api.<MCE_CLUSTER>:6443
type: Opaque
```

> ⚠️ The auto-import-secret is **single-use** and deleted after successful import.

---

### 2.3 Validate Import

```bash
oc get managedcluster
```

Expected:

```text
JOINED: True
AVAILABLE: True
```

---

## 🤖 3. Automate Hosted Cluster Discovery & Import

### 3.1 Create Discovery ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: discovery-config
  namespace: open-cluster-management-global-set
data:
  rosa-filter: ""
```

---

### 3.2 Auto-Import Policy

* Iterates through all DiscoveredCluster objects
* Skips already imported clusters
* Auto-imports **Active MultiClusterEngineHCP** clusters

Apply:

```bash
oc apply -f policy-mce-hcp-autoimport.yaml
```

---

### 3.3 Placement and Binding

Placement (hub-only):

```yaml
matchLabels:
  name: local-cluster
```

Bind using `PlacementBinding`.

---

### 3.4 Validate Policy

```bash
oc get policies -n open-cluster-management-global-set
```

Expected:

```text
COMPLIANCE STATE: Compliant
```

---

## 🔍 4. Validate Discovery and Auto-Import

### 4.1 Check Discovered Clusters

```bash
oc get discoveredcluster -A
```

Key fields:

```yaml
spec:
  importAsManagedCluster: true
  status: Active
  type: MultiClusterEngineHCP
```

Once set, the cluster is automatically imported.

---

### 4.2 HyperShift Add-on Logs

```bash
oc logs -n open-cluster-management-agent-addon-discovery \
  deployment/hypershift-addon-agent -f
```

Expected:

* Local cluster detected
* HyperShift operator management skipped

---

## 🧩 5. Optional: Manual Import of Existing HCP Clusters

Existing clusters can be imported via:

**Infrastructure → Clusters → Import Cluster → YAML View**

Naming pattern:

```text
<hosting-cluster-name>-<agent-nodes>
```

---

## 🏁 Summary

This repository demonstrates:

* Zero-touch HyperShift cluster onboarding
* Policy-driven automation
* Scalable RHACM governance
* Production-ready MCE integration

It supports **fully automated** and **hybrid manual + automated** workflows.

---

## 📚 References

* [https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes)
* [https://docs.redhat.com/en/documentation/openshift_container_platform](https://docs.redhat.com/en/documentation/openshift_container_platform)
* [https://hypershift.openshift.io/](https://hypershift.openshift.io/)

---

## 🤝 Contributions

This repository is intended for **learning and reference purposes**.
Feel free to fork, experiment, and extend.

---
