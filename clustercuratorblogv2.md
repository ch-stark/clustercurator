# Orchestrating OpenShift Cluster Lifecycle Operations with ClusterCurator

OpenShift cluster lifecycle operations often involve more than a version change or a provisioning request. Teams may need to validate infrastructure, confirm backups, prepare external services, and verify applications afterward. ClusterCurator connects these preparation and verification tasks to the cluster operation itself.

The `cluster-curator-controller` watches `ClusterCurator` resources on the Red Hat Advanced Cluster Management for Kubernetes (RHACM) hub. It creates a Kubernetes Job that coordinates the requested operation and its Ansible prehooks and posthooks. The upstream project handles standalone clusters, including Hive-provisioned clusters, and hosted clusters using hosted control planes.

For upgrades, the key responsibility is straightforward: **hooks prepare and validate the environment; ClusterCurator initiates and monitors the upgrade.** External automation can submit an upgrade request to ClusterCurator without taking over the upgrade itself.

> **Version scope:** This article describes the upstream implementation reviewed on November 16, 2026. Features present on the upstream `main` branch are not necessarily available or supported in every RHACM release. Check the documentation and installed CRD for your deployed version before using a field or operation. Release numbers in examples illustrate configuration structure, not recommended upgrade destinations.

## How ClusterCurator Works

A `ClusterCurator` resource declares the requested operation through `spec.desiredCuration`. The controller creates a curator Job on the hub, with ordered steps for that operation.

For an upgrade with hooks, the sequence is:

```text
Upgrade request submitted to the hub
  → Ansible prehooks, in order
  → ClusterCurator initiates the upgrade
  → ClusterCurator monitors completion
  → Ansible posthooks, in order
  → Curation completes
```

Installation uses a different sequence that can include credential preparation, activation of provisioning, and cluster import monitoring. The precise steps depend on the operation and cluster type; they are not one universal pipeline for every request. See the upstream [job launcher](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/controller/launcher/job.go).

### Operations and Availability

| Operation | Purpose | Upstream implementation reviewed |
| --- | --- | --- |
| `install` | Coordinate provisioning and optional automation | Implemented |
| `upgrade` | Coordinate version or update configuration changes and optional automation | Implemented |
| `destroy` | Coordinate cluster deletion and cleanup automation | Implemented; behavior depends on cluster type |
| `scale` | Represent scaling intent in the API | Defined in the API; no scaling workflow implemented |
| `delete-cluster-namespace` | Remove the cluster namespace after applicable checks | Implemented; not a substitute for cluster destruction |

This table describes upstream code availability. Product support can be narrower, especially for destroy hooks and newer upgrade modes.

## Connecting an External Automation Controller

Ansible hook templates can reside on an external AAP automation controller. The controller does not have to be deployed on the ACM hub. ClusterCurator uses the Secret referenced by the operation's `towerAuthSecret` to identify the controller endpoint and credentials.

For a standalone managed cluster, create the Secret in its cluster namespace on the hub:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: external-controller-access
  namespace: my-managed-cluster
type: Opaque
stringData:
  host: https://automation.example.com
  token: <CONTROLLER_API_TOKEN>
```

Use an account authorized to launch the required templates. Ensure the hub can reach the controller and that controller access, certificates, and template permissions are configured correctly. Keep the token out of Git; manage it through your established secret-management process.

### What Does “Without the AAP Operator” Mean?

There are two different requirements to distinguish:

- **Keeping the AAP controller outside the hub:** The hook integration can reference an external controller through its credential Secret.
- **Removing the Ansible job resource integration:** Standard ClusterCurator Ansible hooks create `AnsibleJob` resources. The Ansible Automation Platform Resource Operator is still required to process those resources and communicate with the external controller.

Using an external controller therefore does not make standard Ansible hooks operator-free. The Resource Operator requirement is documented in the [RHACM automation prerequisites](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.14/html/clusters/cluster_mce_overview).

AWX is the upstream project associated with automation controller; it is not another name for a supported AAP deployment. Confirm the support status of the customer's specific AAP, legacy Tower, or AWX version separately from whether its API is technically compatible.

### Can ClusterCurator Run a Playbook Directly from Git?

The standard Ansible hook path references a controller job or workflow template. It does not embed an Ansible execution engine that directly runs a raw playbook from Git.

To use a playbook stored in Git, configure the repository as a Project in the external controller and expose the playbook through a Job Template. ClusterCurator then invokes that template through the Ansible job integration.

## Configuring Prehooks and Posthooks

The `prehook` and `posthook` fields are arrays. Each entry identifies a template on the controller selected by `towerAuthSecret`.

| Field | Purpose |
| --- | --- |
| `name` | Name of the controller job or workflow template |
| `type` | `Job` for a job template, or `Workflow` for a workflow template |
| `extra_vars` | Variables passed to the template |
| `job_tags` | Task tags to include for a job template |
| `skip_tags` | Task tags to exclude for a job template |

In the reviewed implementation, `job_tags` and `skip_tags` apply to job templates, not workflow templates. The controller also supplies cluster context through additional variables, with the available information depending on the operation and cluster resources. See the [hook implementation](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/jobs/ansible/job.go).

### Can a Prehook Perform Several Actions?

Yes. Choose the level at which you want to organize the automation:

1. **Multiple tasks in one playbook:** Use one job template to perform a related set of checks or preparation steps.
2. **Multiple entries in `prehook`:** Use separate job templates when each action should have its own controller job and result.
3. **An Ansible workflow template:** Use `type: Workflow` when dependencies, branching, or parallel execution should be managed by the external controller.

ClusterCurator executes hook entries **sequentially, in their listed order**. It waits for each to complete successfully before starting the next. A failure stops progression through that phase; a failed prehook prevents the upgrade step from starting. Posthooks follow the same sequential execution pattern.

For a workflow entry, ClusterCurator waits for the workflow's reported result. Configure failure handling within that workflow so a failed readiness check produces the result you intend to use as the upgrade gate.

### Example: Several Preparation Jobs Before an Upgrade

The following example checks cluster health, verifies a backup, and enables maintenance mode before ClusterCurator upgrades the cluster. Afterward, it validates applications and disables maintenance mode.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-managed-cluster
  namespace: my-managed-cluster
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate: "<VALID_TARGET_VERSION>"
    monitorTimeout: 180
    towerAuthSecret: external-controller-access
    prehook:
      - name: CheckClusterHealth
        type: Job
      - name: VerifyBackup
        type: Job
        extra_vars:
          cluster_name: my-managed-cluster
      - name: EnableMaintenanceMode
        type: Job
    posthook:
      - name: ValidateApplications
        type: Job
      - name: DisableMaintenanceMode
        type: Job
```

Replace `<VALID_TARGET_VERSION>` with a valid upgrade destination and create the named templates on the external controller. All hooks in this upgrade configuration use `external-controller-access`; individual hook entries do not select different controller credential Secrets.

Make templates idempotent so retries can safely repeat completed preparation tasks. Also plan recovery for partial execution: posthooks are not a guaranteed cleanup or `finally` mechanism. For example, a failure after maintenance mode is enabled can require a separate recovery action.

## Who Owns the Upgrade?

The customer's question often combines two different ways to trigger work: asking ClusterCurator to perform an upgrade, and directly changing the OpenShift upgrade configuration.

### Starting an Upgrade Inside a Prehook

An Ansible template with sufficient permissions could technically update a managed cluster's `ClusterVersion`, or the corresponding hosted-cluster resources. However, doing this inside an upgrade prehook does **not** replace or disable ClusterCurator's own upgrade step.

After a successful prehook, ClusterCurator continues with its configured upgrade initiation and monitoring. That creates overlapping control of the upgrade, with possible conflicts in target version, timing, or status interpretation. It does not necessarily cause OpenShift to perform two separate upgrades, but it is not a reliable coordination pattern.

Keep prehooks focused on preparation and readiness. Let ClusterCurator initiate the configured upgrade after those gates succeed. See the [upgrade pipeline](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/controller/launcher/job.go).

### Starting the Overall Workflow from External Ansible

External Ansible can instead submit the upgrade request to the ACM hub:

```text
External controller template
  → Create or update the hub's ClusterCurator resource
  → ClusterCurator runs readiness prehooks
  → ClusterCurator initiates and monitors the upgrade
  → ClusterCurator runs verification posthooks
```

This is a practical integration pattern when an existing external workflow must remain the entry point. Give that workflow the required hub permissions, and have it submit the desired `ClusterCurator` configuration before curation begins. It should not submit another upgrade request from within the same upgrade's prehook.

## Coordinating Hosted Cluster Provisioning

For hosted clusters, ClusterCurator can coordinate provisioning that has initially been paused. In the upstream provisioning pattern, both the `HostedCluster` and its associated `NodePool` resources have `spec.pausedUntil: 'true'`.

These are partial resource examples; a deployable hosted cluster requires additional platform and release configuration:

```yaml
apiVersion: hypershift.openshift.io/v1beta1
kind: HostedCluster
metadata:
  name: my-hosted-cluster
  namespace: clusters
spec:
  pausedUntil: 'true'
  # Add the required hosted-cluster configuration.
---
apiVersion: hypershift.openshift.io/v1beta1
kind: NodePool
metadata:
  name: my-hosted-cluster-workers
  namespace: clusters
spec:
  clusterName: my-hosted-cluster
  pausedUntil: 'true'
  # Add the required node-pool configuration.
```

Create the controller credential Secret in the hosted cluster's namespace, then create a `ClusterCurator` with the same name and namespace as the `HostedCluster`:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-hosted-cluster
  namespace: clusters
spec:
  desiredCuration: install
  install:
    towerAuthSecret: external-controller-access
    prehook:
      - name: PrepareExternalNetworking
        type: Job
    posthook:
      - name: ConfigureClusterServices
        type: Job
      - name: ValidateClusterAccess
        type: Job
```

ClusterCurator runs the prehook before proceeding with activation. Its hosted-cluster installation flow removes the provisioning pause and monitors the operation before running posthooks. See the upstream [hosted-cluster provisioning example](https://github.com/stolostron/cluster-curator-controller#hosted-cluster-provisioning-example-kubevirt).

## Hosted Cluster Upgrade Strategies

The reviewed upstream implementation includes upgrade modes for the hosted control plane and worker NodePools. Verify that your RHACM version supports the requested mode and resulting version skew.

### Upgrade the Control Plane and NodePools

With no `upgradeType` specified, the default flow upgrades the control plane and associated NodePools:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-hosted-cluster
  namespace: clusters
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate: "<VALID_TARGET_VERSION>"
    monitorTimeout: 180
```

### Upgrade Only the Control Plane

Where supported, select the control plane explicitly:

```yaml
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate: "<VALID_TARGET_VERSION>"
    upgradeType: ControlPlane
```

### Upgrade Selected NodePools

Where supported, use `upgradeType: NodePools` and optionally list the worker pools to update:

```yaml
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate: "<VALID_TARGET_VERSION>"
    upgradeType: NodePools
    nodePoolNames:
      - my-hosted-cluster-workers-az1
      - my-hosted-cluster-workers-az2
```

The upstream implementation validates the requested NodePool version against the control plane version. These partial snippets illustrate alternative operation configurations, not manifests to apply together. Consult the [hosted-cluster upgrade guide](https://github.com/stolostron/cluster-curator-controller/blob/main/README.md#hosted-cluster-upgrade-example) for the implementation available in the branch you deploy.

### Change an Update Channel

An upgrade operation can also request an update channel change. A channel-only request changes update configuration; it does not itself select a target release:

```yaml
spec:
  desiredCuration: upgrade
  upgrade:
    channel: "<VALID_UPDATE_CHANNEL>"
```

## EUS-to-EUS Upgrades for Standalone Clusters

ClusterCurator's `intermediateUpdate` field supports an upgrade sequence through an intermediate release before reaching the final target. This is useful for supported Extended Update Support (EUS) transitions.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-managed-cluster
  namespace: my-managed-cluster
spec:
  desiredCuration: upgrade
  upgrade:
    intermediateUpdate: "<VALID_INTERMEDIATE_VERSION>"
    desiredUpdate: "<VALID_FINAL_VERSION>"
    monitorTimeout: 180
```

The flow initiates and monitors the intermediate upgrade before initiating and monitoring the final upgrade. The reviewed upstream code also includes API-removal acknowledgment handling. Automated acknowledgment does not establish that applications are compatible with removed APIs; complete that assessment before initiating the workflow.

Select a supported upgrade path, validate both release targets, and review the field's validation and immutability rules in your installed [ClusterCurator CRD](https://github.com/stolostron/cluster-curator-controller/tree/main/deploy).

## Non-Recommended Versions and Disconnected Environments

For the standalone upgrade path, the reviewed upstream implementation recognizes the following annotation to allow a target outside the versions normally accepted by its recommendation check:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: my-managed-cluster
  namespace: my-managed-cluster
  annotations:
    cluster.open-cluster-management.io/upgrade-allow-not-recommended-versions: "true"
spec:
  desiredCuration: upgrade
  upgrade:
    desiredUpdate: "<EXPLICIT_TARGET_VERSION>"
```

`desiredUpdate` is a version string. It is not an object containing a nested `version` or `imageDigest` field.

Allowing a non-recommended target does not establish that an upgrade path is supported or suitable for the cluster. Review applicable update risks and product guidance before using this option.

Disconnected environments also require the appropriate release content, mirroring configuration, and image references. In the reviewed upstream standalone implementation, this path attempts to resolve a release image digest from the cluster's conditional or available updates and falls back to a tag when no digest is found. Older deployed versions may behave differently. See the [version validation and image selection code](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/jobs/hive/hive.go).

A proposal to accept an explicit release image or digest should be documented as an enhancement until it exists in the deployed API. Do not add hypothetical fields to production manifests or run an independent `ClusterVersion` patch from a prehook as an implicit workaround for ClusterCurator's upgrade logic.

## Destroy Operations and Custom Job Flows

The upstream destroy flow includes hook handling, but cluster type and product version determine what is available and supported. For example, the API comments distinguish standalone destroy prehooks from hosted-cluster prehooks and posthooks. Validate your exact workflow before relying on cleanup after deletion.

`overrideJob` also requires care. In the reviewed launcher, the override is read from `spec.install.overrideJob`; the presence of an override field elsewhere in the API does not establish an implemented upgrade override. The current upstream implementation also restricts override containers to the trusted curator image and commands. It is not a general-purpose mechanism for running arbitrary images or raw playbooks.

Use the [job launcher](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/controller/launcher/job.go) and the samples for your deployed version when evaluating custom installation flows.

## Driving Upgrades Through GitOps

For teams already using OpenShift GitOps, storing `ClusterCurator` manifests in Git provides a reviewable record of upgrade intent:

```text
Git change
  → OpenShift GitOps applies ClusterCurator on the hub
  → ClusterCurator executes hooks and the upgrade
  → Cluster status and curation results inform promotion
```

This is an architecture pattern to validate for your environment, rather than a blanket support statement for every combination of GitOps and lifecycle tooling.

Use one owner for upgrade execution. If GitOps manages the hub's `ClusterCurator` request, avoid simultaneously managing the same upgrade through a separate desired-version declaration on the managed cluster's `ClusterVersion` or hosted-cluster release resources.

### Track Execution Separately from Manifest Sync

A successful GitOps sync means the resource was applied. It does not mean the upgrade has completed. A health assessment should consider curation status, the curator Job, and the actual cluster state before declaring success.

ClusterCurator updates execution-related fields during the workflow. Define which fields GitOps owns, how controller changes are handled, and how a completed request is retired or advanced. Validate that reconciliation does not replay an already completed request.

### Promote Through Environments

Use separate manifests or Applications for development, staging, and production. Upgrade a canary first, verify its result, and promote the target version only after the preceding stage meets your readiness criteria.

Do not use a lower desired version to correct drift. OpenShift does not support cluster downgrades. If an emergency upgrade occurs outside the normal process, reconcile the recorded intent with the actual cluster version before resuming automation.

For policy-driven batch upgrades, also evaluate Topology Aware Lifecycle Manager and `ClusterGroupUpgrade` against your fleet requirements. Choose lifecycle tooling based on required coordination and supported workflows, rather than cluster count alone.

## Access Control and Troubleshooting

Use Kubernetes RBAC to control who can create and update `ClusterCurator` resources, read status, and access credential Secrets. Creating a curation request can initiate a significant lifecycle operation, so grant that access deliberately. External workflows submitting upgrade requests need hub permissions scoped to their intended resources.

Start troubleshooting by inspecting the request and curator Jobs:

```bash
oc -n my-managed-cluster get clustercurator my-managed-cluster -o yaml
oc -n my-managed-cluster get jobs
```

Find the curator Job associated with the request, then inspect its pods and the relevant container logs:

```bash
oc -n my-managed-cluster get pods -l job-name=<CURATOR_JOB_NAME>
oc -n my-managed-cluster logs job/<CURATOR_JOB_NAME> -c prehook-ansiblejob
oc -n my-managed-cluster logs job/<CURATOR_JOB_NAME> -c upgrade-cluster
oc -n my-managed-cluster logs job/<CURATOR_JOB_NAME> -c posthook-ansiblejob
```

Container names depend on the operation. For example, EUS workflows use intermediate and final upgrade stages. Inspect the Job before selecting a container.

For Ansible failures, inspect the generated `AnsibleJob` resource and the corresponding external controller job. Distinguish failures to discover or launch a job from failures within the playbook itself. `monitorTimeout` controls cluster-operation monitoring; `jobMonitorTimeout` concerns Ansible job discovery and should not be treated as an overall playbook runtime limit.

## A Practical Operating Model

Use external automation or GitOps to submit lifecycle intent to the hub. Use prehooks for readiness and preparation, ClusterCurator for the cluster operation and monitoring, and posthooks for verification and follow-up configuration.

Multiple prehooks let teams compose those gates without putting every action into a single playbook. An external automation controller can host the templates, while the standard Resource Operator integration connects their results to the curator workflow. Keeping upgrade execution under one owner makes the sequence easier to reason about, troubleshoot, and recover.

## References

- [Cluster Curator Controller upstream project](https://github.com/stolostron/cluster-curator-controller)
- [ClusterCurator API types](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/api/v1beta1/clustercurator_types.go)
- [Curator Job launcher and operation pipelines](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/controller/launcher/job.go)
- [Ansible hook creation and sequential execution](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/jobs/ansible/job.go)
- [Standalone upgrade implementation](https://github.com/stolostron/cluster-curator-controller/blob/main/pkg/jobs/hive/hive.go)
- [RHACM 2.14 cluster lifecycle documentation and automation prerequisites](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.14/html/clusters/cluster_mce_overview)
