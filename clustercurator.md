## Mastering Cluster Lifecycle with Precision: The `cluster-curator-controller` for RHACM and Hosted Control Planes

In the dynamic landscape of multi-cluster Kubernetes management, simply deploying clusters isn't enough. True mastery lies in ensuring each cluster is provisioned, configured, and maintained with pinpoint accuracy, often involving intricate pre- and post-deployment steps. Red Hat Advanced Cluster Management (RHACM) provides the central command; however, for granular, job-driven orchestration, especially when using **hosted control planes**, we turn to the powerful, open-source **`cluster-curator-controller`**.

This controller is a core component of the Open Cluster Management project, meticulously designed to extend the capabilities of Hive's `ClusterDeployment` and RHACM's `ManagedCluster` kinds. Crucially, it brings robust, automated lifecycle management to **hosted cluster and node pool resources**, allowing for precise, job-based execution of tasks.

The `cluster-curator-controller` orchestrates these operations by reacting to the `ClusterCurator` Custom Resource Definition (CRD). Unlike RHACM's policy engine which monitors for compliance, the `cluster-curator-controller` focuses on executing discrete, ordered Kubernetes Jobs to drive cluster state transitions.

### Supported Operations

The `ClusterCurator` CRD supports the following `desiredCuration` values, each with its own pre/post hook capabilities:

| Operation | Description |
|---|---|
| `install` | Provision a new cluster (Hive or Hypershift) with pre/post hooks |
| `upgrade` | Upgrade cluster version, channel, or node pools with pre/post hooks |
| `scale` | Scale cluster workers with pre/post hooks |
| `destroy` | Destroy a cluster with pre/post hooks |
| `delete-cluster-namespace` | Clean up the cluster namespace |

### The Orchestration Flow of `cluster-curator-controller`

When a `ClusterCurator` CR is detected, the controller launches a Kubernetes Job on the hub cluster containing a pipeline of init containers that execute in sequence:

1.  **Cloud Provider Setup:** Apply cloud provider secrets (AWS, GCP, Azure, VMware).
2.  **Ansible Provider Setup:** Create Ansible Tower/AAP secrets.
3.  **Pre-hook Ansible:** Execute automation *before* the core cluster operation begins.
4.  **Activate and Monitor:** Manage activation (e.g., unpausing), start the operation, and monitor to completion.
5.  **Monitor Import:** Watch ManagedCluster import status.
6.  **Post-hook Ansible:** Run configuration and integration tasks *after* the cluster operation completes.
7.  **Complete:** Mark curation as done.

The controller auto-detects the cluster type (Hive vs Hypershift) by comparing the ClusterCurator name and namespace, and adjusts its behavior accordingly. This job-centric approach ensures each step is auditable, repeatable, and deeply integrated into your Kubernetes environment.

### Hook Types

Each operation (install, upgrade, scale, destroy) supports the following hook configuration:

- **prehook** -- Array of Ansible jobs executed before the cluster operation
- **posthook** -- Array of Ansible jobs executed after the cluster operation
- **overrideJob** -- Completely replace the default Job spec with a custom Kubernetes Job definition
- **towerAuthSecret** -- Reference to the Secret with Ansible Tower/AAP credentials
- **jobMonitorTimeout** -- How long to wait for AnsibleJob discovery (default: 5 minutes)

Each hook entry supports:
- `name` (required) -- Ansible Tower/AWX job template name
- `type` -- `Job` (default) or `Workflow` (for Ansible workflow templates)
- `extra_vars` -- Arbitrary key-value parameters passed to Ansible
- `job_tags` / `skip_tags` -- Ansible task filtering (Job type only)

The controller automatically injects useful `extra_vars` into Ansible jobs, including `cluster_deployment` (full ClusterDeployment spec), `install_config` (networking, compute, platform config), and `cluster_info` (ManagedClusterInfo data for upgrades).

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


#### 2. Fine-Grained RBAC for `ClusterCurator` Resources

Controlling who can trigger and view these curation jobs is paramount for operational security. The `cluster-curator-controller` respects standard Kubernetes RBAC.

**Use Case:** Grant specific teams or automation accounts the ability to *create and modify* `ClusterCurator` resources (e.g., Platform Engineers), while others can only *view* their status (e.g., Application Developers).

These ClusterRole definitions, when bound to ServiceAccounts, Users, or Groups, allow for precise control over who can initiate or monitor cluster curation workflows.


### Hosted Cluster Upgrade Strategies

The `cluster-curator-controller` supports three distinct upgrade modes for hosted clusters, giving you fine-grained control over what gets upgraded and when.

#### Upgrade Both Control Plane and NodePools (Default)

By default, an upgrade updates the HostedCluster release image and then rolls through all NodePools sequentially:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-hosted-cluster
  namespace: clusters
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate: "4.16.5"
    monitorTimeout: 180 # Minutes to wait for upgrade completion (default: 120)
```

#### Upgrade NodePools Only

Use `upgradeType: NodePools` to update only the worker NodePools without touching the control plane. You can also target specific NodePools with `nodePoolNames`:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-hosted-cluster
  namespace: clusters
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate: "4.16.5"
    upgradeType: NodePools
    nodePoolNames:
      - my-hosted-cluster-workers-az1
      - my-hosted-cluster-workers-az2
```

The controller validates that the NodePool version does not exceed the control plane version.

#### Update Channel Only

Change the update channel independently of version, for example to switch to fast or EUS channels:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-hosted-cluster
  namespace: clusters
spec:
  desiredCuration: upgrade
  upgrade:
    channel: "fast-4.16"
```

### EUS-to-EUS Upgrades for Hive Clusters

The controller provides special handling for Extended Update Support (EUS) transitions, which require upgrading through an intermediate version. The CRD includes an `intermediateUpdate` field that manages this multi-step process automatically.

**Use Case:** Upgrade from OCP 4.14 (EUS) directly to 4.16 (EUS), with the controller automatically handling the required 4.15 intermediate hop and API deprecation acknowledgments.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-managed-cluster
  namespace: my-managed-cluster
spec:
  desiredCuration: upgrade
  upgrade:
    intermediateUpdate: "4.15.30"
    desiredUpdate: "4.16.5"
    monitorTimeout: 180
```

The controller will:
1. Create the admin acknowledgment ConfigMap entry (`ack-4.15-kube-1.28-api-removals-in-4.16`) on the managed cluster
2. Upgrade to the intermediate version (4.15.30) first
3. Wait for the intermediate upgrade to complete
4. Proceed with the final upgrade to 4.16.5

**Note:** The `intermediateUpdate` field has validation rules -- once set, it cannot be modified and requires `desiredUpdate` to also be present.

### Empowering Flexible Upgrades: Bypassing Recommended Versions with ClusterCurator

Beyond initial provisioning, cluster-curator-controller also offers advanced capabilities for managing OpenShift cluster upgrades, including the crucial ability to specify and upgrade to non-recommended OpenShift versions. This provides significant value to customers who need fine-grained control over their cluster's lifecycle.

Use Case: Upgrade an OpenShift managed cluster to a specific patch version or a release not currently marked as "recommended" by the OpenShift update graph. This is essential for scenarios like applying specific bug fixes, testing release candidates, or adhering to strict internal versioning policies.

### Technical Example:

To allow an upgrade to a `non-recommended` version, you add a specific annotation to your ClusterCurator resource. The ClusterCurator will then set the ClusterVersion resource on the managed cluster with the force update strategy.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-managed-cluster # Name of the ManagedCluster to upgrade
  namespace: my-managed-cluster-namespace
  annotations:
    # This annotation instructs the ClusterCurator to allow non-recommended versions
    # and use the 'force' update strategy on the ClusterVersion resource.
    clustercurator.open-cluster-management.io/upgrade-allow-not-recommended-versions: "true"
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate:
      version: "4.15.39" # The specific, potentially non-recommended version
      # For multi-architecture images, the controller often appends '-multi' tag
      # e.g., image: quay.io/openshift-release-dev/ocp-release@sha256:<digest>
```

When this `ClusterCurator` resource is applied, the controller will proceed with the upgrade to '4.15.39', overriding any recommendations from the OpenShift update service. This mechanism offers critical flexibility for managing diverse cluster environments.

### Scaling Clusters with Hooks

The `scale` operation allows you to wrap cluster scaling with automation:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-cluster
  namespace: my-cluster
spec:
  desiredCuration: scale
  scale:
    towerAuthSecret: tower-access
    prehook:
      - name: ValidateCapacity
        extra_vars:
          target_replicas: 5
    posthook:
      - name: PostScaleValidation
```

### Destroying Clusters with Hooks

The `destroy` operation ensures cleanup automation runs around cluster deletion:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-cluster
  namespace: my-cluster
spec:
  desiredCuration: destroy
  destroy:
    towerAuthSecret: tower-access
    prehook:
      - name: BackupClusterData
      - name: DeregisterFromDNS
    posthook:
      - name: CleanupExternalResources
```

### Custom Job Pipelines with overrideJob

For scenarios where Ansible hooks aren't sufficient, you can completely replace the default Job pipeline with a custom Kubernetes Job definition using `overrideJob`. This gives you full control over the containers, volumes, and execution logic:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-cluster
  namespace: my-cluster
spec:
  desiredCuration: install
  install:
    overrideJob:
      apiVersion: batch/v1
      kind: Job
      metadata:
        name: custom-install-pipeline
      spec:
        template:
          spec:
            containers:
              - name: my-custom-curator
                image: my-registry/my-curator:latest
                command: ["./run-custom-pipeline.sh"]
                env:
                  - name: CLUSTER_NAME
                    value: my-cluster
            restartPolicy: Never
```

### Improving Your Cluster Curation Workflow

Based on these technical examples and recent insights from users, here are two suggestions to further enhance your use of cluster-curator-controller:

### Suggestion 1: Implement GitOps for ClusterCurator Definitions:

How to improve: Treat your ClusterCurator Custom Resources themselves as GitOps artifacts. Store them in a Git repository alongside your HostedCluster and NodePool definitions.
Technical Implementation: Use a GitOps operator like OpenShift GitOps (Argo CD) on your RHACM hub cluster. Configure Argo CD to synchronize your Git repository containing these YAML definitions to the hub.
Benefit: This establishes a single source of truth for your cluster's desired state and its associated curation workflows. All changes are version-controlled, auditable, and can be managed through pull requests, streamlining collaboration and ensuring consistency. When you want to provision a new cluster with predefined curation, you simply commit the relevant YAML files to Git.

### Suggestion 2: Direct Digest Injection for Upgrades in Disconnected Environments:

The Challenge: A user recently attempted an upgrade using upgrade-allow-not-recommended-versions: 'true' with ClusterCurator. In their disconnected environment, ImageDigestMirrorSet was configured, meaning only image digests are accepted, not tags (like 4.15.39-multi). This led to ImagePullBackOff errors because ClusterCurator currently injects the tag, not the digest. Manually fetching the digest and patching the ClusterVersion was required to proceed.
How to improve: Enhance the ClusterCurator upgrade spec to directly accept an image digest for the desired update image, or add logic to intelligently resolve the digest in air-gapped scenarios.
Technical Implementation (Proposed):
Modify the ClusterCurator upgrade spec to allow an imageDigest field alongside or instead of version when upgrade-allow-not-recommended-versions is true.
Example (Hypothetical ClusterCurator spec modification):

```yaml
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate:
      # Option 1: Specify digest directly for disconnected environments
      imageDigest: "quay.io/openshift-release-dev/ocp-release@sha256:abcd...1234"
      # Option 2 (current, problematic for disconnected):
      # version: "4.15.39"
```


This would require a change in the cluster-curator-controller itself to prioritize the imageDigest field if provided, setting `ClusterVersion.spec.desiredUpdate.image` directly.

Benefit: This would avoid breaking automation by eliminating the need for an extra script to manually patch the ClusterVersion with the correct digest. It directly addresses a critical pain point for users operating in air-gapped or strictly disconnected environments, significantly streamlining their upgrade workflows.

### Conclusion

The cluster-curator-controller fills a critical gap in sophisticated multi-cluster environments, particularly for hosted control plane deployments. With support for install, upgrade (including EUS-to-EUS transitions, NodePool-only upgrades, and channel management), scale, and destroy operations -- each with pre/post Ansible hooks or fully custom Job pipelines -- it gives platform teams comprehensive lifecycle automation. Leveraging it alongside RHACM, driven by GitOps principles, empowers you to automate virtually every aspect of your cluster's lifecycle, from initial deployment to critical day-2 operations. This level of curation is what transforms a collection of clusters into a truly managed and standardized fleet.
