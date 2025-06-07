## Mastering Cluster Lifecycle with Precision: The `cluster-curator-controller` for RHACM and Hosted Control Planes

In the dynamic landscape of multi-cluster Kubernetes management, simply deploying clusters isn't enough. True mastery lies in ensuring each cluster is provisioned, configured, and maintained with pinpoint accuracy, often involving intricate pre- and post-deployment steps. Red Hat Advanced Cluster Management (RHACM) provides the central command; however, for granular, job-driven orchestration, especially when using **hosted control planes**, we turn to the powerful, open-source **`cluster-curator-controller`**.

This controller is a core component of the Open Cluster Management project, meticulously designed to extend the capabilities of Hive's `ClusterDeployment` and RHACM's `ManagedCluster` kinds. Crucially, it brings robust, automated lifecycle management to **hosted cluster and node pool resources**, allowing for precise, job-based execution of tasks.

The `cluster-curator-controller` orchestrates these operations by reacting to the `ClusterCurator` Custom Resource Definition (CRD). Unlike RHACM's policy engine which monitors for compliance, the `cluster-curator-controller` focuses on executing discrete, ordered Kubernetes Jobs to drive cluster state transitions.

### The Orchestration Flow of `cluster-curator-controller`

When a `ClusterCurator` CR is detected, the controller launches a series of Kubernetes Jobs on the hub cluster:

1.  **Pre-hook Ansible:** Executes automation *before* the core cluster provisioning begins.
2.  **Activate and Monitor:** Manages the activation (e.g., unpausing) and oversees the cluster deployment process.
3.  **Post-hook Ansible:** Runs critical configuration and integration tasks *after* the cluster is provisioned and ready.

This job-centric approach ensures each step is auditable, repeatable, and deeply integrated into your Kubernetes environment.

---

### Precision Curation: Technical Examples

Let's explore key use cases for the `cluster-curator-controller`, emphasizing its role in hosted cluster deployments and fine-grained control.

#### 1. Curating Hosted Control Plane Provisioning with Pause Control

One of the most powerful integrations of the `cluster-curator-controller` is with hosted control planes. This allows you to **pause the provisioning of a hosted cluster until specific pre-configuration steps are complete**, giving you precise control over the initial build process.

**Use Case:** Provision a hosted cluster and its node pool in a paused state, then unpause it and trigger post-provisioning Ansible playbooks via the `ClusterCurator` controller. This is ideal for ensuring external dependencies or specific network setups are ready before the cluster fully initializes.

**Steps & Configuration:**

1.  **Define `HostedCluster` and `NodePool` with `spec.pausedUntil: 'true'`:**
    This tells the hosted cluster provider to create the resources but not proceed with full provisioning until this field is removed.

    ```yaml
    apiVersion: hypershift.openshift.io/v1beta1
    kind: HostedCluster
    metadata:
      name: my-hosted-cluster
      namespace: clusters # Common convention for HostedClusters
    spec:
      pausedUntil: 'true' # Crucially pauses creation
      # ... other HostedCluster specifications (platform, release, network, etc.)
    ---
    apiVersion: hypershift.openshift.io/v1beta1
    kind: NodePool
    metadata:
      name: my-hosted-cluster-workers
      namespace: clusters
    spec:
      clusterName: my-hosted-cluster
      pausedUntil: 'true' # Also pause NodePool creation
      # ... other NodePool specifications (replicas, platform, etc.)
    ```
    The cluster will show as "Creating" in the console, awaiting curation.

2.  **Prepare Ansible Automation (e.g., with Ansible Automation Platform):**
    For `prehook` or `posthook` Ansible jobs, you'll likely use an external Ansible Automation Platform (AAP) instance (formerly Ansible Tower/AWX). The `cluster-curator-controller` integrates with this via a Secret containing connection details.

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: tower-access-my-cluster # Unique name, specific to this cluster's context
      namespace: clusters # Must be in the HostedCluster's namespace
    stringData:
      host: [https://my-tower-domain.io](https://my-tower-domain.io) # Your AAP instance URL
      token: <YOUR_ANSIBLE_AUTOMATION_PLATFORM_TOKEN> # Token for an admin user
    ```

3.  **Define the `ClusterCurator` Resource:**
    This `ClusterCurator` resource now orchestrates the unpausing and subsequent post-provisioning tasks. The `cluster-curator-controller` automatically detects the cluster type and removes `spec.pausedUntil: 'true'` from *both* the `HostedCluster` and `NodePool` resources, allowing provisioning to proceed.

    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1beta1
    kind: ClusterCurator
    metadata:
      name: my-hosted-cluster # MUST match the HostedCluster's name
      namespace: clusters # MUST match the HostedCluster's namespace
      labels:
        open-cluster-management: curator # Useful for filtering/management
    spec:
      # 'install' is a desiredCuration command that includes unpausing
      desiredCuration: install
      install:
        prehook:
          # Optional: Run Ansible Job Templates *before* unpausing.
          # For example, to prepare networking or external services.
          - name: PreInstallNetworkSetup # An Ansible Job Template name in AAP
            extra_vars:
              vpc_id: vpc-abcdef123
        posthook:
          # Run Ansible Job Templates *after* the cluster is fully provisioned and imported.
          # E.g., to install cluster operators (Quay, Cert-manager, ODF), configure Argo CD,
          # or set up logging/monitoring agents.
          - name: InstallCoreOperators # An Ansible Job Template name in AAP
            extra_vars:
              cluster_name: my-hosted-cluster
              operator_channel: stable
          - name: ConfigureQuayIntegration # Another AAP Job Template
        towerAuthSecret: tower-access-my-cluster # References the Secret created above
    ```

This flow empowers you to inject custom automation at critical points, ensuring your hosted clusters are perfectly tailored from the moment they come online.

---

#### 2. Fine-Grained RBAC for `ClusterCurator` Resources

Controlling who can trigger and view these curation jobs is paramount for operational security. The `cluster-curator-controller` respects standard Kubernetes RBAC.

**Use Case:** Grant specific teams or automation accounts the ability to *create and modify* `ClusterCurator` resources (e.g., Platform Engineers), while others can only *view* their status (e.g., Application Developers).

**RBAC Configuration:**

```yaml
# Permissions for end users to edit ClusterCurators
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: clustercurator-editor-role
rules:
- apiGroups:
  - cluster.open-cluster-management.io # Corrected API group
  resources:
  - clustercurators
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - cluster.open-cluster-management.io # Corrected API group
  resources:
  - clustercurators/status
  verbs:
  - get # Editors need to check status after modifying
# Permissions for end users to view ClusterCurators
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: clustercurator-viewer-role
rules:
- apiGroups:
  - cluster.open-cluster-management.io # Corrected API group
  resources:
  - clustercurators
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - cluster.open-cluster-management.io # Corrected API group
  resources:
  - clustercurators/status
  verbs:
  - get # Viewers need to check status
These ClusterRole definitions, when bound to ServiceAccounts, Users, or Groups, allow for precise control over who can initiate or monitor cluster curation workflows.
---

3. Monitoring and Diagnostics
Understanding the state of your curation jobs is critical. The cluster-curator-controller makes this transparent through standard Kubernetes Job logs.

Diagnostic Steps:

To monitor the status of the provisioning, you can look at the ClusterCurator status itself, but for detailed execution, directly inspect the logs of the underlying Kubernetes Jobs created by the controller within the hosted cluster's namespace.

Bash

# To list the curator jobs:
oc get jobs -n <HOSTED_CLUSTER_NAMESPACE> -l open-cluster-management.io/cluster-curator

# To view logs for a specific phase of a job:
# Replace <CURATOR_JOB_NAME> (e.g., 'curator-job-d9pwh') with the actual job name
# Replace <CONTAINER_NAME> with 'prehook-ansiblejob', 'activate-and-monitor', or 'posthook-ansiblejob'

oc logs job/<CURATOR_JOB_NAME> -c prehook-ansiblejob -n <HOSTED_CLUSTER_NAMESPACE>
oc logs job/<CURATOR_JOB_NAME> -c activate-and-monitor -n <HOSTED_CLUSTER_NAMESPACE>
oc logs job/<CURATOR_JOB_NAME> -c posthook-ansiblejob -n <HOSTED_CLUSTER_NAMESPACE>

# Add "-f" to tail the output for real-time monitoring:
oc logs -f job/<CURATOR_JOB_NAME> -c activate-and-monitor -n <HOSTED_CLUSTER_NAMESPACE>
---


If a failure occurs, the Job's status will reflect this. Inspecting the curator-job-container value within the job logs can pinpoint the exact step where the failure occurred. If monitor is the container, look for additional provisioning jobs for more detail.

"Improving Your Cluster Curation Workflow
Based on these technical examples, here are two suggestions to further enhance your use of cluster-curator-controller:

Suggestion 1: Implement GitOps for ClusterCurator Definitions:

How to improve: Treat your ClusterCurator Custom Resources themselves as GitOps artifacts. Store them in a Git repository alongside your HostedCluster and NodePool definitions.
Technical Implementation: Use a GitOps operator like OpenShift GitOps (Argo CD) on your RHACM hub cluster. Configure Argo CD to synchronize your Git repository containing these YAML definitions to the hub.
Benefit: This establishes a single source of truth for your cluster's desired state and its associated curation workflows. All changes are version-controlled, auditable, and can be managed through pull requests, streamlining collaboration and ensuring consistency. When you want to provision a new cluster with predefined curation, you simply commit the relevant YAML files to Git.
Suggestion 2: Dynamic towerAuthSecret Management and Reuse:

How to improve: Instead of manually creating a towerAuthSecret for each HostedCluster and ClusterCurator pair, centralize and automate its creation and injection.
Technical Implementation:
Centralized Secret: Create a single towerAuthSecret in a dedicated management namespace (e.g., open-cluster-management-system) on your hub cluster with permissions restricted to the cluster-curator-controller's service account.
Dynamic Injection (Kustomize/Controller): Use Kustomize overlays in your GitOps repository to patch the towerAuthSecret name into multiple ClusterCurator resources, or develop a small, custom controller that automatically copies or references a centralized towerAuthSecret into the hosted cluster's namespace upon creation.
Benefit: This approach reduces manual overhead, decreases the risk of inconsistent or expired tokens, and improves security by centralizing sensitive credentials away from individual cluster definition files.
Conclusion
The cluster-curator-controller fills a critical gap in sophisticated multi-cluster environments, particularly for hosted control plane deployments. By providing a job-driven, highly configurable mechanism for pre- and post-provisioning automation, it allows platform teams to achieve unparalleled precision in shaping their Kubernetes clusters. Leveraging it alongside RHACM, driven by GitOps principles, empowers you to automate virtually every aspect of your cluster's lifecycle, from initial deployment to critical day-2 operations. This level of curation is what transforms a collection of clusters into a truly managed and standardized fleet.
