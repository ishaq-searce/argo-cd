# Changelog

## v0.12.1 (2019-04-09)

## Changes since v0.12.0

- [UI] applications view blows up when user does not have  permissions (#1368)
- Add k8s objects circular dependency protection to getApp method (#1374)
- App controller unnecessary set namespace to cluster level resources (#1404)
- Changing SSO login URL to be a relative link so it's affected by basehref (#101) (@arnarg)
- CLI diff should take into account resource customizations (#1294)
- Don't try deleting application resource if it already has `deletionTimestamp` (#1406)
- Fix invalid group filtering in 'patch-resource' command (#1319)
- Fix null pointer dereference error in 'argocd app wait' (#1366)
- kubectl v1.13 fails to convert extensions/NetworkPolicy (#1012)
- Patch APIs are not audited (#1397)

+ 'argocd app wait' should fail sooner if app transitioned to Degraded state (#733)
+ Add mapping to new canonical Ingress API group - kubernetes 1.14 support (#1348) (@twz123)
+ Adds support for `kustomize edit set image`. (#1275)
+ Allow using any name for secrets which store cluster credentials (#1218)
+ Update argocd-util import/export to support proper backup and restore (#1048)

## v0.12.0 (2019-03-20)

### New Features

#### Improved UI

Many improvements to the UI were made, including:

* Table view when viewing applications
* Filters on applications
* Table view when viewing application resources
* YAML editor in UI
* Switch to text-based diff instead of json diff
* Ability to edit application specs

#### Custom Health Assessments (CRD Health)

Argo CD has long been able to perform health assessments on resources, however this could only
assess the health for a few native kubernetes types (deployments, statefulsets, daemonsets, etc...).
Now, Argo CD can be extended to gain understanding of any CRD health, in the form of Lua scripts.
For example, using this feature, Argo CD now understands the CertManager Certificate CRD and will
report a Degraded status when there are issues with the cert.

#### Configuration Management Plugins

Argo CD introduces Config Management Plugins to support custom configuration management tools other
than the set that Argo CD provides out-of-the-box (Helm, Kustomize, Ksonnet, Jsonnet). Using config
management plugins, Argo CD can be configured to run specified commands to render manifests. This
makes it possible for Argo CD to support other config management tools (kubecfg, kapitan, shell
scripts, etc...).

#### High Availability

Argo CD is now fully HA. A set HA of manifests are provided for users who wish to run Argo CD in
a highly available manner. NOTE: The HA installation will require at least three different nodes due
to pod anti-affinity roles in the specs.

#### Improved Application Source

* Support for Kustomize 2
* YAML/JSON/Jsonnet Directories can now be recursed
* Support for Jsonnet external variables and top-level arguments

#### Additional Prometheus Metrics

Argo CD provides the following additional prometheus metrics:
* Sync counter to track sync activity and results over time
* Application reconciliation (refresh) performance to track Argo CD performance and controller activity
* Argo CD API Server metrics for monitoring HTTP/gRPC requests

#### Fuzzy Diff Logic

Argo CD can now be configured to ignore known differences for resource types by specifying a json
pointer to the field path to ignore. This helps prevent OutOfSync conditions when a user has no
control over the manifests. Ignored differences can be configured either at an application level, 
or a system level, based on a group/kind.

#### Resource Exclusions

Argo CD can now be configured to completely ignore entire classes of resources group/kinds.
Excluding high-volume resources improves performance and memory usage, and reduces load and
bandwidth to the Kubernetes API server. It also allows users to fine-tune the permissions that
Argo CD needs to a cluster by preventing Argo CD from attempting to watch resources of that
group/kind.

#### gRPC-Web Support

The argocd CLI can be now configured to communicate to the Argo CD API server using gRPC-Web
(HTTP1.1) using a new CLI flag `--grpc-web`. This resolves some compatibility issues users were
experiencing with ingresses and gRPC (HTTP2), and should enable argocd CLI to work with virtually
any load balancer, ingress controller, or API gateway.

#### CLI features

Argo CD introduces some additional CLI commands:

* `argocd app edit APPNAME` - to edit an application spec using preferred EDITOR
* `argocd proj edit PROJNAME` - to edit an project spec using preferred EDITOR
* `argocd app patch APPNAME` - to patch an application spec
* `argocd app patch-resource APPNAME` - to patch a specific resource which is part of an application


### Breaking Changes

#### Label selector changes, dex-server rename

The label selectors for deployments were been renamed to use kubernetes common labels
(`app.kuberentes.io/name=NAME` instead of `app=NAME`). Since K8s deployment label selectors are
immutable, during an upgrade from v0.11 to v0.12, the old deployments should be deleted using
`--cascade=false` which allows the new deployments to be created without introducing downtime.
Once the new deployments are ready, the older replicasets can be deleted. Use the following
instructions to upgrade from v0.11 to v0.12 without introducing downtime:

```
# delete the deployments with cascade=false. this orphan the replicasets, but leaves the pods running
kubectl delete deploy --cascade=false argocd-server argocd-repo-server argocd-application-controller

# apply the new manifests and wait for them to finish rolling out
kubectl apply <new install manifests>
kubectl rollout status deploy/argocd-application-controller
kubectl rollout status deploy/argocd-repo-server
kubectl rollout status deploy/argocd-application-controller

# delete old replicasets which are using the legacy label
kubectl delete rs -l app=argocd-server
kubectl delete rs -l app=argocd-repo-server
kubectl delete rs -l app=argocd-application-controller

# delete the legacy dex-server which was renamed
kubectl delete deploy dex-server
```

#### Deprecation of spec.source.componentParameterOverrides

For declarative application specs, the `spec.source.componentParameterOverrides` field is now
deprecated in favor of application source specific config. They are replaced with new fields
specific to their respective config management. For example, a Helm application spec using the
legacy field:

```yaml
spec:
  source:
    componentParameterOverrides:
    - name: image.tag
      value: v1.2
```

should move to:

```yaml
spec:
  source:
    helm:
      parameters:
      - name: image.tag
        value: v1.2
```

Argo CD will automatically duplicate the legacy field values to the new locations (and vice versa)
as part of automatic migration. The legacy `spec.source.componentParameterOverrides` field will be
kept around for the v0.12 release (for migration purposes) and will be removed in the next Argo CD
release.

#### Removal of spec.source.environment and spec.source.valuesFiles

The `spec.source.environment` and `spec.source.valuesFiles` fields, which were deprecated in v0.11,
are now completely removed from the Application spec.


#### API/CLI compatibility

Due to API spec changes related to the deprecation of componentParameterOverrides, Argo CD v0.12
has a minimum client version of v0.12.0. Older CLI clients will be rejected.


### Changes since v0.11:
+ Improved UI
+ Custom Health Assessments (CRD Health)
+ Configuration Management Plugins
+ High Availability
+ Fuzzy Diff Logic
+ Resource Exclusions
+ gRPC-Web Support
+ CLI features
+ Additional prometheus metrics
+ Sample Grafana dashboard (#1277) (@hartman17)
+ Support for Kustomize 2
+ YAML/JSON/Jsonnet Directories can now be recursed
+ Support for Jsonnet external variables and top-level arguments
+ Optimized reconciliation performance for applications with very active resources (#1267)
+ Support a separate OAuth2 CLI clientID different from server (#1307)
+ argocd diff: only print to stdout, if there is a diff + exit code (#1288) (@marcb1)
+ Detection and handling of duplicated resource definitions (#1284)
+ Support kustomize apps with remote bases in private repos in the same host (#1264)
+ Support patching resource using REST API (#1186)
* Deprecate componentParameterOverrides in favor of source specific config (#1207)
* Support talking to Dex using local cluster address instead of public address (#1211)
* Use Recreate deployment strategy for controller (#1315)
* Honor os environment variables for helm commands (#1306) (@1337andre)
* Disable CGO_ENABLED for server/controller binaries (#1286)
* Documentation fixes and improvements (@twz123, @yann-soubeyrand, @OmerKahani, @dulltz)
- Fix CRD creation/deletion handling (#1249)
- Git cloning via SSH was not verifying host public key (#1276)
- Fixed multiple goroutine leaks in controller and api-server
- Fix isssue where `argocd app set -p` required repo privileges. (#1280)
- Fix local diff of non-namespaced resources. Also handle duplicates in local diff (#1289)
- Deprecated resource kinds from 'extensions' groups are not reconciled correctly (#1232)
- Fix issue where CLI would panic after timeout when cli did not have get permissions (#1209)
- invalidate repo cache on delete (#1182) (@narg95)

## v0.11.2 (2019-02-19)
+ Adds client retry. Fixes #959 (#1119)
- Prevent deletion hotloop (#1115)
- Fix EncodeX509KeyPair function so it takes in account chained certificates (#1137) (@amarruedo)
- Exclude metrics.k8s.io from watch (#1128)
- Fix issue where dex restart could cause login failures (#1114)
- Relax ingress/service health check to accept non-empty ingress list (#1053)
- [UI] Correctly handle empty response from repository/<repo>/apps API

## v0.11.1 (2019-01-18)
+ Allow using redis as a cache in repo-server (#1020)
- Fix controller deadlock when checking for stale cache (#1044)
- Namespaces are not being sorted during apply (#1038)
- Controller cache was susceptible to clock skew in managed cluster
- Fix ability to unset ApplicationSource specific parameters
- Fix force resource delete API (#1033)
- Incorrect PermissionDenied error during app creation when using project roles + user-defined RBAC (#1019)
- Fix `kubctl convert` issue preventing deployment of extensions/NetworkPolicy (#1012)
- Do not allow metadata.creationTimestamp to affect sync status (#1021)
- Graceful handling of clusters where API resource discovery is partially successful (#1018)
- Handle k8s resources circular dependency (#1016)
- Fix `app diff --local` command (#1008)

## v0.11.0 (2019-01-10)
This is Argo CD's biggest release ever and introduces a completely redesigned controller architecture.

### New Features

#### Performance & Scalability
The application controller has a completely redesigned architecture which improved performance and
scalability during application reconciliation.

This was achieved by introducing an in-memory, live state cache of lightweight Kubernetes object 
metadata. During reconciliation, the controller no longer performs expensive, in-line queries of app
related resources in K8s API server, instead relying on the metadata available in the live state 
cache. This dramatically improves performance and responsiveness, and is less burdensome to the K8s
API server.

#### Object relationship visualization for CRDs
With the new controller design, Argo CD is now able to understand ownership relationship between
*all* Kubernetes objects, not just the built-in types. This enables Argo CD to visualize
parent/child relationships between all kubernetes objects, including CRDs.

#### Multi-namespaced applications
During sync, Argo CD will now honor any explicitly set namespace in a manifest. Manifests without a
namespace will continue deploy to the "preferred" namespace, as specified in app's
`spec.destination.namespace`. This enables support for a class of applications which install to
multiple namespaces. For example, Argo CD can now install the
[prometheus-operator](https://github.com/helm/charts/tree/master/stable/prometheus-operator)
helm chart, which deploys some resources into `kube-system`, and others into the
`prometheus-operator` namespace.

#### Large application support
Full resource objects are no longer stored in the Application CRD object status. Instead, only
lightweight metadata is stored in the status, such as a resource's sync and health status.
This change enabled Argo CD to support applications with a very large number of resources 
(e.g. istio), and reduces the bandwidth requirements when listing applications in the UI.

#### Resource lifecycle hook improvements
Resource lifecycle hooks (e.g. PreSync, PostSync) are now visible/manageable from the UI.
Additionally, bare Pods with a restart policy of Never can now be used as a resource hook, as an
alternative to Jobs, Workflows.

#### K8s recommended application labels
The tracking label for resources has been changed to use `app.kubernetes.io/instance`, as
recommended in [Kubernetes recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/),
(changed from `applications.argoproj.io/app-name`). This will enable applications managed by Argo CD
to interoperate with other tooling which are also converging on this labeling, such as the
Kubernetes dashboard. Additionally, Argo CD no longer injects any tracking labels at the
`spec.template.metadata` level.

#### External OIDC provider support
Argo CD now supports auth delegation to an existing, external OIDC providers without the need for
running Dex (e.g. Okta, OneLogin, Auth0, Microsoft, etc...)

The optional, [Dex IDP OIDC provider](https://github.com/dexidp/dex) is still bundled as part of the
default installation, in order to provide a seamless out-of-box experience, enabling Argo CD to
integrate with non-OIDC providers, and to benefit from Dex's full range of
[connectors](https://github.com/dexidp/dex/tree/master/Documentation/connectors).

#### OIDC group bindings to Project Roles
OIDC group claims from an OAuth2 provider can now be bound to a Argo CD project roles. Previously,
group claims could only be managed in the centralized ConfigMap, `argocd-rbac-cm`. They can now be
managed at a project level. This enables project admins to self service access to applications
within a project.

#### Declarative Argo CD configuration
Argo CD settings can be now be configured either declaratively, or imperatively. The `argocd-cm`
ConfigMap now has a `repositories` field, which can reference credentials in a normal Kubernetes
secret which you can create declaratively, outside of Argo CD.

#### Helm repository support
Helm repositories can be configured at the system level, enabling the deployment of helm charts
which have a dependency to external helm repositories.

### Breaking changes:

* Argo CD's resource names were renamed for consistency. For example, the application-controller
  deployment was renamed to argocd-application-controller. When upgrading from v0.10 to v0.11,
  the older resources should be pruned to avoid inconsistent state and controller in-fighting.

* As a consequence to moving to recommended kubernetes labels, when upgrading from v0.10 to v0.11,
  all applications will immediately be OutOfSync due to the change in tracking labels. This will
  correct itself with another sync of the application. However, since Pods will be recreated, please
  take this into consideration, especially if your applications are configured with auto-sync.

* There was significant reworking of the `app.status` fields to reduce the payload size, simplify
  the datastructure and remove fields which were no longer used by the controller. No breaking
  changes were made in `app.spec`.

* An older Argo CD CLI (v0.10 and below) will not be compatible with Argo CD v0.11. To keep
  CI pipelines in sync with the API server, it is recommended to have pipelines download the CLI
  directly from the API server https://${ARGOCD_SERVER}/download/argocd-linux-amd64 during the CI
  pipeline.

### Changes since v0.10:
* Improve Application state reconciliation performance (#806)
* Refactor, consolidate and rename resource type data structures
+ Declarative setup and configuration of ArgoCD (#536)
+ Declaratively add helm repositories (#747)
+ Switch to k8s recommended app.kubernetes.io/instance label (#857)
+ Ability for a single application to deploy into multiple namespaces (#696)
+ Self service group access to project applications (#742)
+ Support for Pods as a sync hook (#801)
+ Support 'crd-install' helm hook (#355)
+ Use external 'diff' utility to render actual vs target state difference
+ Show sync policy in app list view
* Remove resources state from application CRD (#758)
* API server & UI should serve argocd binaries instead of linking to GitHub (#716)
* Update versions for kubectl (v1.13.1), helm (v2.12.1), ksonnet (v0.13.1)
* Update version of aws-iam-authenticator (0.4.0-alpha.1)
* Ability to force refresh of application manifests from git
* Improve diff assessment for Secrets, ClusterRoles, Roles
- Failed to deploy helm chart with local dependencies and no internet access (#786)
- Out of sync reported if Secrets with stringData are used (#763)
- Unable to delete application in K8s v1.12 (#718)

## v0.10.6 (2018-11-14)
- Fix issue preventing in-cluster app sync due to go-client changes (issue #774)

## v0.10.5 (2018-11-13)
+ Increase concurrency of application controller
* Update dependencies to k8s v1.12 and client-go v9.0 (#729)
- add argo cluster permission to view logs (#766) (@conorfennell)
- Fix issue where applications could not be deleted on k8s v1.12
- Allow 'syncApplication' action to reference target revision rather then hard-coding to 'HEAD' (#69) (@chrisgarland)
- Issue #768 - Fix application wizard crash

## v0.10.4 (2018-11-07)
* Upgrade to Helm v0.11.0 (@amarrella)
- Health check is not discerning apiVersion when assessing CRDs (issue #753)
- Fix nil pointer dereference in util/health (@mduarte)

## v0.10.3 (2018-10-28)
* Fix applying TLS version settings
* Update to kustomize 1.0.10 (@twz123)

## v0.10.2 (2018-10-25)
* Update to kustomize 1.0.9 (@twz123)
- Fix app refresh err when k8s patch is too slow

## v0.10.1 (2018-10-24)

- Handle case where OIDC settings become invalid after dex server restart (issue #710)
- git clean also needs to clean files under gitignore (issue #711)

## v0.10.0 (2018-10-19)

### Changes since v0.9:

+ Allow more fine-grained sync (issue #508)
+ Display init container logs (issue #681)
+ Redirect to /auth/login instead of /login when SSO token is used for authenticaion (issue #348)
+ Support ability to use a helm values files from a URL (issue #624)
+ Support public not-connected repo in app creation UI (issue #426)
+ Use ksonnet CLI instead of ksonnet libs (issue #626)
+ We should be able to select the order of the `yaml` files while creating a Helm App (#664)
* Remove default params from app history (issue #556)
* Update to ksonnet v0.13.0
* Update to kustomize 1.0.8
- API Server fails to return apps due to grpc max message size limit  (issue #690)
- App Creation UI for Helm Apps shows only files prefixed with `values-` (issue #663)
- App creation UI should allow specifying values files outside of helm app directory bug (issue #658)
- argocd-server logs credentials in plain text when adding git repositories (issue #653)
- Azure Repos do not work as a repository (issue #643)
- Better update conflict error handing during app editing (issue #685)
- Cluster watch needs to be restarted when CRDs get created (issue #627)
- Credentials not being accepted for Google Source Repositories (issue #651)
- Default project is created without permission to deploy cluster level resources (issue #679)
- Generate role token click resets policy changes (issue #655)
- Input type text instead of password on Connect repo panel (issue #693)
- Metrics endpoint not reachable through the metrics kubernetes service (issue #672)
- Operation stuck in 'in progress' state if application has no resources (issue #682)
- Project should influence options for cluster and namespace during app creation (issue #592)
- Repo server unable to execute ls-remote for private repos (issue #639)
- Resource is always out of sync if it has only 'ksonnet.io/component' label (issue #686)
- Resource nodes are 'jumping' on app details page (issue #683)
- Sync always suggest using latest revision instead of target UI bug (issue #669)
- Temporary ignore service catalog resources (issue #650)

## v0.9.2 (2018-09-28)

* Update to kustomize 1.0.8
- Fix issue where argocd-server logged credentials in plain text during repo add (issue #653)
- Credentials not being accepted for Google Source Repositories (issue #651)
- Azure Repos do not work as a repository (issue #643)
- Temporary ignore service catalog resources (issue #650)
- Normalize policies by always adding space after comma

## v0.9.1 (2018-09-24)

- Repo server unable to execute ls-remote for private repos (issue #639)

## v0.9.0 (2018-09-24)

### Notes about upgrading from v0.8
* Cluster wide resources should be allowed in default project (due to issue #330):

```
argocd project allow-cluster-resource default '*' '*'
```

* Projects now provide the ability to allow or deny deployments of cluster-scoped resources
(e.g. Namespaces, ClusterRoles, CustomResourceDefinitions). When upgrading from v0.8 to v0.9, to
match the behavior of v0.8 (which did not have restrictions on deploying resources) and continue to
allow deployment of cluster-scoped resources, an additional command should be run:

```bash
argocd proj allow-cluster-resource default '*' '*'
```

The above command allows the `default` project to deploy any cluster-scoped resources which matches
the behavior of v0.8.

* The secret keys in the argocd-secret containing the TLS certificate and key, has been renamed from
  `server.crt` and `server.key` to the standard `tls.crt` and `tls.key` keys. This enables Argo CD
  to integrate better with Ingress and cert-manager. When upgrading to v0.9, the `server.crt` and
  `server.key` keys in argocd-secret should be renamed to the new keys.

### Changes since v0.8:
+ Auto-sync option in application CRD instance (issue #79)
+ Support raw jsonnet as an application source (issue #540)
+ Reorder K8s resources to correct creation order (issue #102)
+ Redact K8s secrets from API server payloads (issue #470)
+ Support --in-cluster authentication without providing a kubeconfig (issue #527)
+ Special handling of CustomResourceDefinitions (issue #613)
+ Argo CD should download helm chart dependencies (issue #582)
+ Export Argo CD stats as prometheus style metrics (issue #513)
+ Support restricting TLS version (issue #609)
+ Use 'kubectl auth reconcile' before 'kubectl apply' (issue #523)
+ Projects need controls on cluster-scoped resources (issue #330)
+ Support IAM Authentication for managing external K8s clusters (issue #482)
+ Compatibility with cert manager (issue #617)
* Enable TLS for repo server (issue #553)
* Split out dex into it's own deployment (instead of sidecar) (issue #555)
+ [UI] Support selection of helm values files in App creation wizard (issue #499)
+ [UI] Support specifying source revision in App creation wizard allow (issue #503)
+ [UI] Improve resource diff rendering (issue #457)
+ [UI] Indicate number of ready containers in pod (issue #539)
+ [UI] Indicate when app is overriding parameters (issue #503)
+ [UI] Provide a YAML view of resources (issue #396)
+ [UI] Project Role/Token management from UI (issue #548)
+ [UI] App creation wizard should allow specifying source revision (issue #562)
+ [UI] Ability to modify application from UI (issue #615)
+ [UI] indicate when operation is in progress or has failed (issue #566)
- Fix issue where changes were not pulled when tracking a branch (issue #567)
- Lazy enforcement of unknown cluster/namespace restricted resources (issue #599)
- Fix controller hot loop when app source contains bad manifests (issue #568)
- Fix issue where Argo CD fails to deploy when resources are in a K8s list format (issue #584)
- Fix comparison failure when app contains unregistered custom resource (issue #583)
- Fix issue where helm hooks were being deployed as part of sync (issue #605)
- Fix race conditions in kube.GetResourcesWithLabel and DeleteResourceWithLabel (issue #587)
- [UI] Fix issue where projects filter does not work when application got changed
- [UI] Creating apps from directories is not obvious (issue #565)
- Helm hooks are being deployed as resources (issue #605)
- Disagreement in three way diff calculation (issue #597)
- SIGSEGV in kube.GetResourcesWithLabel (issue #587)
- Argo CD fails to deploy resources list (issue #584)
- Branch tracking not working properly (issue #567)
- Controller hot loop when application source has bad manifests (issue #568)

## v0.8.2 (2018-09-12)
- Downgrade ksonnet from v0.12.0 to v0.11.0 due to quote unescape regression
- Fix CLI panic when performing an initial `argocd sync/wait`

## v0.8.1 (2018-09-10)
+ [UI] Support selection of helm values files in App creation wizard (issue #499)
+ [UI] Support specifying source revision in App creation wizard allow (issue #503)
+ [UI] Improve resource diff rendering (issue #457)
+ [UI] Indicate number of ready containers in pod (issue #539)
+ [UI] Indicate when app is overriding parameters (issue #503)
+ [UI] Provide a YAML view of resources (issue #396)
- Fix issue where changes were not pulled when tracking a branch (issue #567)
- Fix controller hot loop when app source contains bad manifests (issue #568)
- [UI] Fix issue where projects filter does not work when application got changed

## v0.8.0 (2018-09-04)

### Notes about upgrading from v0.7
* The RBAC model has been improved to support explicit denies. What this means is that any previous
RBAC policy rules, need to be rewritten to include one extra column with the effect:
`allow` or `deny`. For example, if a rule was written like this:
    ```
    p, my-org:my-team, applications, get, */*
    ```
    It should be rewritten to look like this:
    ```
    p, my-org:my-team, applications, get, */*, allow
    ```

### Changes since v0.7:
+ Support kustomize as an application source (issue #510)
+ Introduce project tokens for automation access (issue #498)
+ Add ability to delete a single application resource to support immutable updates (issue #262)
+ Update RBAC model to support explicit denies (issue #497)
+ Ability to view Kubernetes events related to application projects for auditing
+ Add PVC healthcheck to controller (issue #501)
+ Run all containers as an unprivileged user (issue #528)
* Upgrade ksonnet to v0.12.0
* Add readiness probes to API server (issue #522)
* Use gRPC error codes instead of fmt.Errorf (#532)
- API discovery becomes best effort when partial resource list is returned (issue #524)
- Fix `argocd app wait` printing incorrect Sync output (issue #542)
- Fix issue where argocd could not sync to a tag (#541)
- Fix issue where static assets were browser cached between upgrades (issue #489)

## v0.7.2 (2018-08-21)
- API discovery becomes best effort when partial resource list is returned (issue #524)

## v0.7.1 (2018-08-03)
+ Surface helm parameters to the application level (#485)
+ [UI] Improve application creation wizard (#459)
+ [UI] Show indicator when refresh is still in progress (#493)
* [UI] Improve data loading error notification (#446)
* Infer username from claims during an `argocd relogin` (#475)
* Expand RBAC role to be able to create application events. Fix username claims extraction
- Fix scalability issues with the ListApps API (#494)
- Fix issue where application server was retrieving events from incorrect cluster (#478)
- Fix failure in identifying app source type when path was '.'
- AppProjectSpec SourceRepos mislabeled (#490)
- Failed e2e test was not failing CI workflow
* Fix linux download link in getting_started.md (#487) (@chocopowwwa)

## v0.7.0 (2018-07-27)
+ Support helm charts and yaml directories as an application source
+ Audit trails in the form of API call logs
+ Generate kubernetes events for application state changes
+ Add ksonnet version to version endpoint (#433)
+ Show CLI progress for sync and rollback
+ Make use of dex refresh tokens and store them into local config
+ Expire local superuser tokens when their password changes
+ Add `argocd relogin` command as a convenience around login to current context
- Fix saving default connection status for repos and clusters
- Fix undesired fail-fast behavior of health check
- Fix memory leak in the cluster resource watch
- Health check for StatefulSets, DaemonSet, and ReplicaSets were failing due to use of wrong converters

## v0.6.2 (2018-07-23)
- Health check for StatefulSets, DaemonSet, and ReplicaSets were failing due to use of wrong converters

## v0.6.1 (2018-07-18)
- Fix regression where deployment health check incorrectly reported Healthy
+ Intercept dex SSO errors and present them in Argo login page

## v0.6.0 (2018-07-16)
+ Support PreSync, Sync, PostSync resource hooks
+ Introduce Application Projects for finer grain RBAC controls
+ Swagger Docs & UI
+ Support in-cluster deployments internal kubernetes service name
+ Refactoring & Improvements
* Improved error handling, status and condition reporting
* Remove installer in favor of kubectl apply instructions
* Add validation when setting application parameters
* Cascade deletion is decided during app deletion, instead of app creation
- Fix git authentication implementation when using using SSH key
- app-name label was inadvertently injected into spec.selector if selector was omitted from v1beta1 specs

## v0.5.4 (2018-06-27)
- Refresh flag to sync should be optional, not required

## v0.5.3 (2018-06-20)
+ Support cluster management using the internal k8s API address https://kubernetes.default.svc (#307)
+ Support diffing a local ksonnet app to the live application state (resolves #239) (#298)
+ Add ability to show last operation result in app get. Show path in app list -o wide (#297)
+ Update dependencies: ksonnet v0.11, golang v1.10, debian v9.4 (#296)
+ Add ability to force a refresh of an app during get (resolves #269) (#293)
+ Automatically restart API server upon certificate changes (#292)

## v0.5.2 (2018-06-14)
+ Resource events tab on application details page (#286)
+ Display pod status on application details page (#231)

## v0.5.1 (2018-06-13)
- API server incorrectly compose application fully qualified name for RBAC check (#283)
- UI crash while rendering application operation info if operation failed

## v0.5.0 (2018-06-12)
+ RBAC access control
+ Repository/Cluster state monitoring
+ Argo CD settings import/export
+ Application creation UI wizard
+ argocd app manifests for printing the application manifests
+ argocd app unset command to unset parameter overrides
+ Fail app sync if prune flag is required (#276)
+ Take into account number of unavailable replicas to decided if deployment is healthy or not #270
+ Add ability to show parameters and overrides in CLI (resolves #240)
- Repo names containing underscores were not being accepted (#258)
- Cookie token was not parsed properly when mixed with other site cookies

## v0.4.7 (2018-06-07)
- Fix argocd app wait health checking logic

## v0.4.6 (2018-06-06)
- Retry argocd app wait connection errors from EOF watch. Show detailed state changes

## v0.4.5 (2018-05-31)
+ Add argocd app unset command to unset parameter overrides
- Cookie token was not parsed properly when mixed with other site cookies

## v0.4.4 (2018-05-30)
+ Add ability to show parameters and overrides in CLI (resolves #240)
+ Add Events API endpoint
+ Issue #238 - add upsert flag to 'argocd app create' command
+ Add repo browsing endpoint (#229)
+ Support subscribing to settings updates and auto-restart of dex and API server
- Issue #233 - Controller does not persist rollback operation result
- App sync frequently fails due to concurrent app modification

## v0.4.3 (2018-05-21)
- Move local branch deletion as part of git Reset() (resolves #185) (#222)
- Fix exit code for app wait (#219)

## v0.4.2 (2018-05-21)
+ Show URL in argocd app get
- Remove interactive context name prompt during login which broke login automation
* Rename force flag to cascade in argocd app delete

## v0.4.1 (2018-05-18)
+ Implemented argocd app wait command

## v0.4.0 (2018-05-17)
+ SSO Integration
+ GitHub Webhook
+ Add application health status
+ Sync/Rollback/Delete is asynchronously handled by controller
* Refactor CRUD operation on clusters and repos
* Sync will always perform kubectl apply
* Synced Status considers last-applied-configuration annotatoin
* Server & namespace are mandatory fields (still inferred from app.yaml)
* Manifests are memoized in repo server
- Fix connection timeouts to SSH repos

## v0.3.2 (2018-05-03)
+ Application sync should delete 'unexpected' resources #139
+ Update ksonnet to v0.10.1
+ Detect unexpected resources
- Fix: App sync frequently fails due to concurrent app modification #147
- Fix: improve app state comparator: #136, #132

## v0.3.1 (2018-04-24)
+ Add new rollback RPC with numeric identifiers
+ New argo app history and argo app rollback command
+ Switch to gogo/protobuf for golang code generation
- Fix: create .argocd directory during argo login (issue #123)
- Fix: Allow overriding server or namespace separately (issue #110)

## v0.3.0 (2018-04-23)
+ Auth support
+ TLS support
+ DAG-based application view
+ Bulk watch
+ ksonnet v0.10.0-alpha.3
+ kubectl apply deployment strategy
+ CLI improvements for app management

## v0.2.0 (2018-04-03)
+ Rollback UI
+ Override parameters

## v0.1.0 (2018-03-12)
+ Define app in Github with dev and preprod environment using KSonnet
+ Add cluster Diff App with a cluster Deploy app in a cluster
+ Deploy a new version of the app in the cluster
+ App sync based on Github app config change - polling only
+ Basic UI: App diff between Git and k8s cluster for all environments Basic GUI