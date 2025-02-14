## PolicyGen (Policy Generator)
The PolicyGen library is used to facilitate creating ACM policies based on a set of provided source CRs (custom resources) and [PolicyGenTemplate](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/ran-crd/policy-gen-template-crd.yaml) CR which describe how to customize those source CRs.
The full list of the CRs that ztp RAN solution provide to deploy ACM policies are under the [ztp/source-crs](https://github.com/openshift-kni/cnf-features-deploy/tree/master/ztp/source-crs). PolicyGenTemplate constructs the ACM policies by offering the following customization mechanisms:
  1. Overlay: The given CRs that will be constructed into ACM policy may have some or all of their contents replaced by values specified in the PolicyGenTemplate.
  1. Grouping: Policies defined in the PolicyGenTemplate will be created under the same namespace and share the same PlacmentRules and PlacementBinding.

- Example 1: Consider the PolicyGenTemplate below to create ACM policies for both [ConsoleOperatorDisable.yaml](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/source-crs/ConsoleOperatorDisable.yaml) and [ClusterLogging.yaml](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/source-crs/ClusterLogging.yaml).
```
apiVersion: ran.openshift.io/v1
kind: PolicyGenTemplate
metadata:
 name: "group-du-sno"
 namespace: "group-du-sno"
spec:
  bindingRules:
    group-du-sno: ""
  mcp: "master"
  sourceFiles:
    - fileName: ConsoleOperatorDisable.yaml
      policyName: "console-policy"
    - fileName: ClusterLogging.yaml
      policyName: "log-policy"
      spec:
        curation:
          curator:
            schedule: "30 3 * * *"
        collection:
          logs:
            type: "fluentd"
            fluentd: {}
```

The generated policies will be:

```
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/standards: NIST SP 800-53
  name: group-du-sno-console-policy
  namespace: group-du-sno-policies
spec:
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: group-du-sno-console-policy-config
      spec:
        namespaceselector:
          exclude:
          - kube-*
          include:
          - '*'
        object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: operator.openshift.io/v1
            kind: Console
            metadata:
              annotations:
                include.release.openshift.io/ibm-cloud-managed: "false"
                include.release.openshift.io/self-managed-high-availability: "false"
                include.release.openshift.io/single-node-developer: "false"
                release.openshift.io/create-only: "true"
              name: cluster
            spec:
              logLevel: Normal
              managementState: Removed
              operatorLogLevel: Normal
        remediationAction: enforce
        severity: low
  remediationAction: enforce
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/standards: NIST SP 800-53
  name: group-du-sno-log-policy
  namespace: group-du-sno-policies
spec:
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: group-du-sno-log-policy-config
      spec:
        namespaceselector:
          exclude:
          - kube-*
          include:
          - '*'
        object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: logging.openshift.io/v1
            kind: ClusterLogging
            metadata:
              name: instance
              namespace: openshift-logging
            spec:
              collection:
                logs:
                  fluentd: {}
                  type: fluentd
              curation:
                curator:
                  schedule: 30 3 * * *
                type: curator
              managementState: Managed
        remediationAction: enforce
        severity: low
  remediationAction: enforce
```

The placement binding and rules of the generated policies will be:

```
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: group-du-sno-placementbinding
  namespace: group-du-sno-policies
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: group-du-sno-placementrules
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: group-du-sno-console-policy
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: group-du-sno-log-policy
---
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: group-du-sno-placementrules
  namespace: group-du-sno-policies
spec:
  clusterSelector:
    matchExpressions:
    - key: group-du-sno
      operator: In
      values:
      - ""
```
## Build and execute
- Requirement
  - golang is installed

- Run the following command to build the policygenerator binary:
```
    $ make build
```

- Run the following command to execute the unit tests:
```
    $ make test
```

- Run the following command to execute policygenerator binary with a PolicyGenTemplate example
```
    $ ./policygenerator  -sourcePath ../source-crs ../ran-crd/policy-gen-template-ex.yaml
```  

- Run the following command to see the command's help text:
```
./policygenerator  --help
Usage of ./policygenerator:
  -outPath string
    	Directory to write the genrated policies (default "__unset_value__")
  -pgtPath string
    	Directory where policyGenTemp files exist (default "__unset_value__")
  -sourcePath string
    	Directory where source-crs files exist (default "source-crs")
  -wrapInPolicy
    	Wrap the CRs in acm Policy (default true)
```

- For using policygenerator library as kustomize plugin, see the [policy-generator-kustomize-plugin](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/policygenerator-kustomize-plugin/README.md). 
