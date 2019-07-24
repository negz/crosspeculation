# crosspeculation

This repository contains some speculative design changes to Crossplane managed
resources. The first commit (well, second if you include Github's "initial
commit") adds a contemporary Crossplane `KubernetesApplication` that deploys
Wordpress. It uses resource classes and claims to create a `MySQLInstance` and a
`KubernetesCluster`, then deploys the `KubernetesApplication` to that
`KubernetesCluster`. Subsequent commits tweak and build on this existing
foundation.

## Proposed Changes

Note that everything in this repository is pure exploratory speculation. With
that in mind, we introduce a few changes. In summary:

* Several resource kinds become cluster scoped rather than namespaced.
* Resource classes become 'strongly typed', i.e. tightly coupled with the
  managed resources they configure.
* We introduce several network connectivity related managed resources.
* We introduce the concept of a 'resource usage'
* We introduce the concept of a 'resource connection'
* We introduce the concept of a 'resource claim policy'
* Resource classes, resource connections, and providers are all specified at the
  resource claim level.
* The resource claim policy sets default classes, connections, and providers
  when they are omitted.

The following resource kinds are cluster scoped:

* Managed resources (e.g. `CloudSQLInstance`, `GKECluster`)
* Resource classes (e.g. `CloudSQLInstanceClass`)
* Resource connections (e.g. `Connection`)
* Providers (e.g. `Provider`)
* Resource claim policies (e.g. `MySQLInstanceClusterPolicy`)

While the remainder are namespaced, including:

* Resource claims (e.g. `MySQLInstance`, `KubernetesCluster`)
* Resource usages (e.g. `MySQLInstanceUsage`)
* Workloads (e.g. `KubernetesApplication`, `KubernetesApplicationResource`)
* Resource claim policies (e.g. `MySQLInstancePolicy`)

You may notice that resource claim policies have both cluster scoped and
namespaced variants (similar to `RoleBinding` and `ClusterRoleBinding`). This
allows policy to be applied at either the cluster scope or the namespace scope.

### Resource Connections

A resource connection is an abstract representation of a the connectivity needs
of one or more resource claims. You could think of it a little like a 'peering'
of resources. An example resource connection:

```yaml
---
apiVersion: gcp.crossplane.io/v1alpha1
kind: Connection
metadata:
  name: experimental
strategies:
- colocate
# The below fields are optional. If you reference an existing Network resource
# we'll colocate any managed resources in that network. If you omit it, we'll
# create a VPC network for you automatically and colocate managed resources in
# that. Similar for regions, except we'd pick a sane default rather than
# creating one. You could imagine zoneRef, subnetworkRef in here too as optional
# fields.
colocation:
  networkRef:
  - name: experimental
  regionRef:
  - name: us-central1
```

Resource connections specify one or more strategies used to _attempt_ to ensure
connectivity between multiple managed resources. The above example uses the
'colocate' strategy, which in this case ensures any managed resources bound to
its associated resource claims are in the same GCP region and VPC network.

### Resource Claims

Resource claims may now reference a resource class, resource connection, and a
provider. For example:

```yaml
---
apiVersion: database.crossplane.io/v1alpha1
kind: MySQLInstance
metadata:
  name: experimental
  namespace: default
  labels:
    app: wordpress
spec:
  engineVersion: "5.7"
  classRef:
    apiVersion: database.gcp.crossplane.io/v1alpha1
    kind: CloudSQLInstanceClass
    name: cloudsqlinstance
  connectionRef:
    apiVersion: gcp.crossplane.io/v1alpha1
    kind: Connection
    name: colocate
  providerRef:
    apiVersion: gcp.crossplane.io/v1alpha1
    kind: Provider
    name: developmentaccount
```

If the class, connection, or provider ref are omitted they are provided by a
`MySQLInstancePolicy` like:

```yaml
apiVersion: core.crossplane.io/v1alpha1
kind: MySQLInstancePolicy
metadata:
  name: mysql-aws-policy
  namespace: crossplane-system
defaultClassRef:
  kind: rdsinstanceclass.database.aws.crossplane.io/v1alpha1
  name: standard-mysql
  namespace: crossplane-system
defaultProviderRef:
  kind: provider.aws.crossplane.io/v1alpha1
  name: developmentaccount
  namespace: crossplane-system
defaultConnectionRef:
  apiVersion: gcp.crossplane.io/v1alpha1
  kind: Connection
  name: colocate
```

This allows the cluster operator persona to specify _how_ managed resources are
created, _where_ (i.e. with which provider) they're created, and how they're
_connected_ through a rudimentary policy.


### Resource Usages

I won't go into too much detail here, as at the current time these are mostly
the same as what is outlined in https://github.com/crossplaneio/crossplane/pull/564,
including the pending changes captured in https://github.com/crossplaneio/crossplane/pull/564#issuecomment-511638333. 

An example usage:

```yaml
---
apiVersion: storage.crossplane.io/v1alpha1
kind: MySQLInstanceUsage
metadata:
  name: experimental
  namespace: default
spec:
  claimRef:
    apiVersion: storage.crossplane.io/v1alpha1
    kind: MySQLInstance
    name: experimental
  publish:
    service:
      name: experimental
      targetTemplate: "{{ .instance.status.endpoint }}"
    secret:
      name: experimental
      dataTemplate:
        config.json: |
          {
              "host": "experimental.default.svc.cluster.local",
              "port": "{{ .instance.status.port }}",
              "user": "{{ .user.spec.name }}
              "password": "{{ .user.secret.password }}"
          }
```

## User experience

Let's assume we want to deploy an application to a `KubernetesCluster`, and
ensure said application could access the databases of a `MySQLInstance`. Let's
also assume these resource claims would be satified by a `GKECluster` and a
`CloudSQLInstance` in GCP, because the default resource classes for these claims
were the `GKEClusterClass` and `CloudSQLInstanceClass` respectively. Note that
GKE clusters and CloudSQL instances are deployed in Google-operated VPC networks
rather than your own, but both allow you to specify a VPC network to which they
will be peered and reachable from. 'Colocating' in the context of these
resources means requesting they both be attached to (rather than created in) the
same VPC network.

To enable all of this the administrator persona creates:

1. A `Provider` (in the `gcp.crossplane.io` API group), and points it at a
   `Secret` containing provider credentials.
1. A `GKEClusterClass` and `CloudSQLInstanceClass`. These classes contain
   managed-resource-specific configuration such as database size and cluster
   node count, but do not specify a provider, VPC network, region, etc.
1. A `Connection` (in the `gcp.crossplane.io` API group), with the `colocate`
   strategy.
1. A `MySQLInstancePolicy` and a `KubernetesClusterPolicy` specifying that any
   `MySQLInstance` or `KubernetesCluster` claim that does not specifically
   request a provider, class, or connection use the ones created in steps one
   through three.

The app operator then creates `MySQLInstance` and `KubernetesCluster` claims.
Both claims omit their optional `providerRef`, `classRef`, and `connectionRef`,
causing them to use the defaults set by the aforementioned policies.

## Behind the Scenes

At a controller level:

* Providers and resource classes are not reconciled by their own controllers
  today. Providers are used by managed resource controllers to create cloud
  provider clients, and resource classes are used by the resource claim
  controllers to create managed resources. This does not change.
* Connections would have their own controller, and this controller is
  responsible for satisfying connectivity needs as magically as possible. In the
  example above the connection controller would be responsible for creating a
  VPC network for the two claims to use if one was not specififed.
* Managed resource controllers _never_ magic anything into existence. i.e. The
  `CloudSQLInstance` managed resource is blissfully unconcerned with policies
  and defaults. It requires its creator to explicitly tell it what VPC to peer
  with. This allows a cluster administrator to fall back to lovingly hand
  crafting managed resources for static binding to claims.
* The resource claim controller is responsible for using the `Connection` in
  addition to the resource class to form a managed resource; it configures the
  managed resource's fields (VPC network etc) appropriately based on the
  `Connection` just as it sets the disk size etc based on the resource class.

## TODO

* Publishing connection details to remote clusters
* Embedding usages in KubernetesApplication
