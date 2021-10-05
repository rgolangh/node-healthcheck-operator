## Node Healthcheck Controller

A Node entering an unready state after 5 minutes is an obvious sign that a
failure occurred, but depending on your physical environment, workloads, and
tolerance for risk, there may be other criteria or thresholds that are
appropriate.

The [Node Healthcheck Controller]() checks each Node's set of [NodeConditions]()
against the criteria and thresholds defined for it in [NodeHealthCheck]() CRs.
If the Node is deemed to be in a failed state, and remediation is appropriate,
the controller will instantiate a RemediationRequest template (defined as part
of the CR) that specifies the mechansim/controller to be used for recovery.

Should the Node recover on its own, the NH controller removes the instantiated
RemediationRequest.  In all other respects, the RemediationRequest is owned by
the [target remediation mechanism]() and will persist until that controller is
satisfied remediation is complete.  For some mechanisms that may mean the Node
has entered a safe state (eg. the underlying "hardware" has been deprovisioned),
for others it may be the Node coming back online (eg. after a reboot).

Remediation is not always the correct response to a failure.  Especially in
larger clusters, we want to protect against failures that appear to take out
large portions of compute capacity but are really the result of failures on or
near the control plane.

For this reason, the [healthcheck CR](#nodehealthcheck-custom-resource) includes
the ability to define a percentage or total number of nodes that can be
considered candidates for concurrent remediation.

When the controller starts it will create a default [healthcheck CR](#nodehealthcheck-custom-resource),
if there is no healthcheck CR at all in the cluster already(supporting upgrades
with existing configurations). The default CR works with [poison-pill], that
should be installed automatically if you deployed using the [operator hub].
The CR uses all defaults except a selector to select only worker nodes.

## Cluster Upgrade awareness

Cluster upgrade usually draw workers reboot, mainly to apply OS updates, and this
disruption can cause other nodes to overload to compensate for the lost compute capacity,
and may even start to appear unhealthy. Making remediation decisions at this moment may
interfere with the course of the upgrade and may even fail it completely.
For that NHC will stop remediating new unhealthy nodes in case in detects that a cluster is upgrading.
At the moment only OpenShift is supported since [ClusterVersionOperator](https://github.com/openshift/cluster-version-operator)
is automatically managing the cluster upgrade state.

## Installation

Install the Node Healthcheck operator using [operator hub]. The installation
will also install the [poison-pill] operator as a default remediator.

For development environments you can run `make deploy deploy-poison-pill`.
See the [Makefile](./Makefile) for more variables.

On start the controller creates a default resource name `nhc-worker-default`,
that remediates using Poison Pill, and will check for worker-only heath issues.
See [NodeHealthCheck Custom Resource](#nodehealthcheck-custom-resource) for the default properties.
If there is an existing resource the controller will not create a default one.

## NodeHealthCheck Custom Resource

Here is the default NHC resource the operator creates on start:

```yaml
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: nhc-worker-default
spec:
  # mandatory
  remediationTemplate:
    kind: PoisonPillRemediationTemplate
    apiVersion: medik8s.io/v1alpha1
    name: poison-pill-default-template
    namespace: poison-pill
  # see k8s doc on selectors https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#resources-that-support-set-based-requirements
  selector:
    matchExpressions:
      - key: node-role.kubernetes.io/worker
        operator: Exists
  maxUnhealthy: "49%"
  unhealthyConditions:
    - type: Ready
      status: "False"
      duration: 300s
    - type: Ready
      status: Unknown
      duration: 300s
```

| Field | Mandatory | Default Value | Description |
| --- | --- | --- | --- |
| _remediationTemplate_ | yes | n/a | A reference to a remediation template provided by an infrastructure provider. If a node needs remediation the controller will create an object from this template and then it should be picked up by a remediation provider.|
| _selector_ | no | empty selector that selects all nodes | a nodes selector of type [metav1.LabelSelector](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#LabelSelector) | 
| _maxUnhealthy_ | no | 49% | Any further remediation is only allowed if at most "MaxUnhealthy" nodes selected by "selector" are not healthy.| 
| _unhealthyConditions_ | no | `[{type: Ready, status: False, duration: 300s},{type: Ready, status: Unknown, duration: 300s}]` | list of the conditions that determine whether a node is considered unhealthy.  The conditions are combined in a logical OR, i.e. if any of the conditions is met, the node is unhealthy.|

## NodeHealthCheck life-cycle

When a node is unhealthy:
  - sum up how many other nodes are unhealthy.
  - if the number of unhealthy nodes < maxUnhealthy the controllers creates the external remediation object
  - the external remediation object has an OwnerReference on the NodeHeathCheck object
  - controller updates the NodeHealthCheck.Status

When a node turns healthy:
  - the controller deletes the external remediation object
  - the controller updates the NodeHealthCheck.Status 


### External Remediation Resources

External remediation resources are custom resource meant to be reconciled by speciallized remediation providers.
The NHC object has a property of a External Remediaiton Template, and this template Spec will be
copied over to the External Remediation Object Spec.
For example this example NHC has this template defined:

```yaml
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: nodehealthcheck-sameple
spec:
  remediationTemplate:
    kind: PoisonPillRemediationTemplate
    apiVersion: poison-pill.medik8s.io/v1alpha1
    name: group-x
    namespace: default


```

- it is the admin's responsiblity to create a template object from the template kind `ProviderXRemdiationTemplate`
  with the name `group-x`.

```yaml
apiVersion: poison-pill.medik8s.io/v1alpha1
kind: PoisonPillRemediationTemplate
metadata:
  name: group-x
  namespace: default
spec:
  template:
    spec: {}
```

- the controller will create an object with the kind `ProviderPillRemdiation` (postfix 'Template' trimmed)
  and the object will have ownerReference set to the co-responding NHC object

```yaml
apiVersion: poison-pill.medik8s.io/v1alpha1
kind: PoisonPillRemdiation
metadata:
  # named after the target node
  name: worker-0-21
  namespace: default
  ownerReferences:
    - kind: NodeHealthCheck
      apiVersion: remediation.medik8s.io/v1alpha1
      name: nhc-worker-default
spec: {}

```

## Remediation Providers responsibility

  It is upto the remediation provider to delete the external remediation object if the node is deleted and another is
  reprovisioned. In that specific scenario the controller can not assume a successful node remediation because the
  node with that name doesn't exist, and instead there will be a new one.

### RBAC rules aggregation

Each provider must label it's rules with `rbac.ext-remediation/aggregate-to-ext-remediation: true` so the controller
will aggreate its rules and will have the proper permission to create/delete external remediation objects.

[operator hub]: https://operatorhub.io/operator/node-healthcheck-operator
[poison-pill]: https://github.com/medik8s/poison-pill
