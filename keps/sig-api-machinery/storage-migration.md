---
kep-number: 25
title: Migrating API objects to latest storage version
authors:
  - "@xuchao"
owning-sig: sig-api-machinery
reviewers:
  - "@deads2k"
  - "@lavalamp"
approvers:
  - "@deads2k"
  - "@lavalamp"
creation-date: 2018-08-06
last-updated: 2018-08-06
status: provisional
---

# Migrating API objects to latest storage version

## Table of Contents

   * [Migrating API objects to latest storage version](#migrating-api-objects-to-latest-storage-version)
      * [Table of Contents](#table-of-contents)
      * [Summary](#summary)
      * [Motivation](#motivation)
         * [Goals](#goals)
      * [Proposal](#proposal)
         * [Alpha workflow](#alpha-workflow)
         * [API](#api)
         * [Failure recovery](#failure-recovery)
         * [Beta workflow - Automation](#beta-workflow---automation)
            * [a. Exposing the configured storage versions via API](#a-exposing-the-configured-storage-versions-via-api)
            * [b. Using the apiserver version to trigger migration](#b-using-the-apiserver-version-to-trigger-migration)
         * [Risks and Mitigations](#risks-and-mitigations)
      * [Graduation Criteria](#graduation-criteria)
      * [Alternatives](#alternatives)
         * [“Active management”](#active-management)
         * [update-storage-objects.sh](#update-storage-objectssh)


## Summary

We propose to extend and improve the [oc adm migrate storage][] command line
tool to migrate the stored API objects in Kubernetes clusters. We will
deliver a standalone tool of alpha quality in 2018 Q3. We will integrate the
storage migration into the Kubernetes upgrade process in 2018 Q4. We will make
the migration automatically triggered in 2019.

[oc adm migrate storage]:https://www.mankier.com/1/oc-adm-migrate-storage

## Motivation

"Today it is possible to create API objects (e.g., HPAs) in one version of
Kubernetes, go through multiple upgrade cycles without touching those objects,
and eventually arrive at a version of Kubernetes that can’t interpret the stored
resource and crashes. See k8s.io/pr/52185."[1][]. We propose a solution to the
problem.

[1]:https://docs.google.com/document/d/1eoS1K40HLMl4zUyw5pnC05dEF3mzFLp5TPEEt4PFvsM

### Goals

A successful storage version migration tool must:
* work for Kubernetes built-in APIs, custom resources (CR), and aggregated APIs.
* do not add burden to cluster administrators or Kubernetes distributions.
* only cause insignificant load to apiservers. For example, if the master has
  10GB memory, the migration tool should generate less than 10 qps of single
  object operations(TODO: measure the memory consumption of PUT operations;
  study how well the default 10 Mbps bandwidth limit in the oc command work).
* work for big clusters that have ~10^6 instances of some resource types.
* make progress in flaky environment, e.g., flaky apiservers, or the migration
  process get preempted.
* allow system administrators to track the migration progress.

As to the deliverables,
* in the short term, providing system administrators with a tool to migrate
  the API objects to the proper storage versions.
* in the long term, automating the migration of Kubernetes built-in APIs, CR,
  aggregated APIs without further burdening system administrators or Kubernetes
  distributions.

## Proposal

### Alpha workflow

We will create a **kube-migrator** command line tool. At the alpha stage,
**kube-migrator** needs to be manually invoked, and does not handle custom
resources. 

After all the apiservers are at the desired version, including kube-apiservers
and aggregated apiservers, the cluster administrator runs `kube-migrator
install` command. It
* deletes any existing `deployment` of **kube-migrator controller** 
* creates a `deployment` of **kube-migrator controller** 
* uses the same kubeconfig file as **kubectl** does.

On start-up, the **kube-migrator controller** does the following,
* discovers all supported APIs.
* lists all custom resource definitions, excludes them from the API list
  retrieved in the previous step.
* for each non-custom resource, creates a `migration` CR (see the [API
  section][] for the schema) if it doesn't exist already.
* for each newly created `migration` instance, increments the
  `migration.spec.generation`, and logs the reason.
* the `ownerReferences` of the `migration` objects are set to the **kube-migrator
  controller** `deployment`. Thus, when user invokes the **kube-migrator** tool,
  the old `migration`s are deleted with the old `deployment`.

The control loop of **kube-migrator controller** does the following:
* watches the instances of the `migration` CR.
* if `migration.spec.generation != migration.status.observedGeneration`
    * deletes existing **migration worker** `job` for the resource type.
    * launches new **migration worker** `job` to migrate the resource type. The
      `job` is annotated with the generation.
    * makes sure there is only one **migration worker** `job` running at a time,
      to avoid overloading the apiserver.
    * updates `migration.status.observedGeneration`.
* watches the **migration worker** jobs. Updates
  `migration.status.completedGeneration` when a job completes.
* short-circuit: if `migration.spec.groupVersions` shows that the resource has
  only one version, then its `.status.observedGeneration` and
  `.status.completedGeneration` are updated directly without launching the
  worker `job`.

The **migrator worker** `job` runs the equivalence of `oc adm migrate storage
--include=<resource type>` to migrate a resource type. The **migrator worker**
uses API chunking to retrieve partial lists of a resource type and thus can
migrate a small chunk at a time. It stores the [continue token] in a `configmap`
to preserve progress. With the inconsistent continue token introduced in
[#67284][], the **migrator worker** does not need to worry about expired continue
token.

[#67284]:https://github.com/kubernetes/kubernetes/pull/67284
[API section]:#api

The cluster admin can run the `kube-migrator progress` command to check if the
migration has completed. The command lists all `migrations` instances and checks
if all of them have `.spec.generation == .status.observedGeneration`.

### API

We introduce the `migration` API to record the intention and progress of a
migration. The API will be a CRD defined in the `migration.k8s.io` group.

Read the [workflow section][] to understand how the API is used.

```golang
type StorageVersionMigration struct {
  metav1.TypeMeta
  metav1.ObjectMeta
  Spec StorageVersionMigrationSpec
  Status StorageVersionMigrationStatus
}

type StorageVersionMigrationSpec {
  // Resource is the identifier of the resource that's being migrated.
  // TBD: maybe resource=hash(etcd instance, etcd path prefix).
  // Immuntable.
  Resource string 
  // GroupVersions are all the groups and versions this resource is served by
  // the apiserver. The server preferred version should be groupVersions[0].
  // **Migration workers** GET & PUT via the preferred version by default, to
  // avoid conversions in the apiserver.
  GroupVersions []GroupVersion
  // Generation is the generation of the spec. Incrementing generation triggers
  // migration. We do not use the objectMeta.generation because its semantic is
  // not clear yet (e.g., objectMeta.generation might be incremented when
  // labels/annotations are updated [#67428][])
  Generation int64
  // Reason of the migration. Examples are "manually triggered", "apiserver
  // updated", etc.
  Reason string
}

type GroupVersion struct {
  Group string
  Version string
}

type StorageVersionMigrationStatus {
  // ObeservedGeneration is the spec generation that has been observed by the
  // **migration controller** It indicates a **migration worker** for this
  // generation has been launched.
  ObeservedGeneration int64
  // CompletedGeneration is the greatest spec generation whose **migration worker**
  // has completed.
  CompletedGeneration int64
}
```

[#67428]:https://github.com/kubernetes/kubernetes/issues/67428
[continue token]:https://github.com/kubernetes/kubernetes/blob/972e1549776955456d9808b619d136ee95ebb388/staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go#L82
[workflow section]:#alpha-workflow

We need to expand the discovery API to recognize APIs in different groups
that share the same storage, e.g., `extensions/v1beta1/deployment` and
`apps/v1/deployment` share the same key space in etcd. Specifically, we expand
the [APIResource][] API:

```golang
type APIResource struct {
  Name string
  ...
  // StorageUID=SHA-256(etcd server in the form of scheme://ip:port, etcd path),
  // obfuscated with a random key. E.g., for pods, it is
  // SHA-256(scheme://ip:port/registry/pods).
  // The string is obfuscated to avoid exposing server configuration details.
  StorageUID string
}
```

[APIResource]:https://github.com/kubernetes/kubernetes/blob/d38fb2c6451447ad0ecbdfed971a563419d60883/staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go#L874


### Failure recovery

As stated in the goals section, the migration has to make progress even if the
environment is flaky. This section describes how the migrator recovers from
failure.

Kubernetes **replicaset controller** restarts the **migration controller** `pod`
if it fails. The restarted **migration controller** follows the same bootstrap
workflow as described in the [workflow section][]. Because the **migration
controller** adopts existing `migration` objects, it can resume from where
it left off.

Kubernetes **job controller** ensures the completion of a **migration worker**.
The **migration worker** uses the continue token as the checkpoint.
* the worker stores the latest continue token returned by the apiserver in a
  `configmap`.
* when the worker starts, it resumes listing with the continue token.
* the **migration controller** creates the configmap when creates the worker `job`,
  and sets the worker job as the owner. When the **migration controller** deletes
  the completed `job`, the `configmap` is garbage collected.

[workflow section]:#alpha-workflow

### Beta workflow - Automation

It is a beta goal to automate the migration workflow. That is, migration does
not need to be triggered manually by cluster admins, or by custom control loops
of Kubernetes distributions.

The automated migration should work for Kubernetes built-in resource types,
custom resources, and aggregated resources.

Here are two alternatives and their requirements.

#### a. Exposing the configured storage versions via API

This is what the [Active Management][] design proposes. It requires:
* exposing the configured [storage versions][] of Kubernetes built-in resource
  types and aggregated resources. 

Note that custom resources already exposes this information. 

The migration of built-in resource types, aggregated resources, and custom
resources can then all be triggered by changes to the storage versions
configurations.

[Active Management]:https://docs.google.com/document/d/1eoS1K40HLMl4zUyw5pnC05dEF3mzFLp5TPEEt4PFvsM
[storage versions]:https://github.com/kubernetes/kubernetes/blob/db9545e69e630e5bc8c3c2dab14f9339dd3b871c/pkg/kubeapiserver/options/storage_versions.go#L98

#### b. Using the apiserver version to trigger migration

If the Kubernetes deprecation [policy][] is followed by every apiserver, and
every cluster uses the default [storage version configs][], then we can use the
apiserver version to drive the migration. More specifically,
* kube-apiserver removes the `--storagve-version` flag, so that every deployment
  of kube-apiserver uses the default storage versions. [#67678][]
* aggregated apiservers follow semantic versioning, and expose their version via
  API.
* aggregated apiservers follow the same API deprecation [policy][].

Then the migration of built-in resources and aggregated resources can be driven
by the changes of apiservers versions.

The migration of custom resources can still be triggered by changes to the
exposed storage versions configurations.

[storage version configs]:https://github.com/kubernetes/kubernetes/blob/db9545e69e630e5bc8c3c2dab14f9339dd3b871c/pkg/kubeapiserver/options/storage_versions.go#L98
[storage version]:https://github.com/kubernetes/kubernetes/blob/c1f7df2b0efe89fb9d26db2f2837bac11d57064e/staging/src/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/types.go#L66
[stored version]:https://github.com/kubernetes/kubernetes/blob/c1f7df2b0efe89fb9d26db2f2837bac11d57064e/staging/src/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/types.go#L177
[policy]:https://kubernetes.io/docs/reference/using-api/deprecation-policy/
[#67678]:https://github.com/kubernetes/kubernetes/pull/67678

In both alternatives, HA apiservers present additional challenges. The migration
must start after all HA apiservers have agreed on the storage versions or the
apiserver versions. The agreement protocol is worth its own KEP.

We will revisit this section in 2018 Q4. 

### Risks and Mitigations

The migration process does not change the objects, so it will not pollute
existing data.

If the rate limiting is not tuned well, the migration can overload the
apiserver. Users can delete the migration controller and the migration
jobs to mitigate.

Before upgrading or downgrading the cluster, the cluster administrator must run
the `kube-migrator progress` to make sure the migration has completed. Otherwise
the apiserver can crash, because it cannot interpret the serialized data in
etcd. To mitigate, the cluster administrator can rollback the apiserver to the
old version, and wait for the migration to complete. If the apiserver does not
crash after upgrading or downgrading, the `migration` objects lose track of
migration progress, because the default storage versions might have changed after
upgrading or downgrading, but no one increments the `migration.spec.generation`.
Administrator needs to re-launch the **kube-migrator** to recover.

TODO: it is safe to rollback an apiserver to the previous version without
waiting for the migration to complete. It is only unsafe to roll-forward or
rollback twice. We need to design how to record the previous version, especially
for the aggregated apiservers.

## Graduation Criteria

* alpha: delivering a tool that implements the "workflow" and "failure recovery"
  sections. ETA is 2018 Q3. The development will be in a separate repository
  (perhaps in kubernetes-sigs?). The tool will have its own CI that does NOT
  block Kubernetes 1.12 release.

* beta: integrating the migration into Kubernetes upgrade tests, and it will be a
  blocking test for 1.13. ETA is 2018 Q4.

* GA: implementing the automation.

We will revisit this section in 2018 Q4. 

## Alternatives

### “Active management”

[API Object Storage Version: Active Management][] (for simplicity, we’ll call it
“active management”) is another propose. *Active management* is different to
this proposal **kube-migrator** in that it requires the apiservers to expose the
storage versions. The exposed storage versions are used to drive the migration
of Kubernetes built-in resource types, customer resources, and aggregated
resources in the same fashion.

Another advantage is that the *active management* controller only
has to touch the resource types whose storage versions have changed, thus more
efficient, though the difference is not that significant, because
* the migration needs to run every time the master is upgraded to a
  different minor version, which happens once per quarter. 
* the migration is rate limited, the amortized resource saving is insignificant.
* **PUT** request doesn't result in etcd traffic if the storage version hasn't
  changed.

The disadvantage is added complexity to the discovery API.

[API Object Storage Version: Active Management]:https://docs.google.com/document/d/1eoS1K40HLMl4zUyw5pnC05dEF3mzFLp5TPEEt4PFvsM
[policy]:https://kubernetes.io/docs/reference/using-api/deprecation-policy/


### update-storage-objects.sh

The Kubernetes repo has an update-storage-objects.sh script. It is not
production ready: no rate limiting, hard-coded resource types, no persisted
migration states. We will delete it, leaving a breadcrumb for any users to
follow to the new tool.
