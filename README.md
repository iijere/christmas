The OperatorPolicy is designed to enhance the management of Kubernetes-native applications that are packaged with the Operator Lifecycle Manager (OLM) across multiple clusters. With the OperatorPolicy, you can:

- **Control Upgrades**: Treat the initial installation separately from subsequent upgrades.

- **Manage Alerts**: Configure which situations are deemed violations or simply informational.

- **Uninstall**: Utilize new functionalities to ensure that installations are completely removed.

When defining the OperatorPolicy, most `spec` fields operate similarly to those in the ConfigurationPolicy. However, the fields listed below are specific to the OperatorPolicy:

- **`operatorGroup` (optional)**: This field is used to specify the details required for the `OperatorGroup`. You can include the `name` and `namespace`, as well as definitions from an `OperatorGroup` `.spec`. If this field is left unset, the policy will accept any existing `OperatorGroup` in the namespace and can also create a default AllNamespaces group if the policy is enforced.

- **`subscription` (required)**: This field defines the Operator, typically indicating what the policy wants to be installed and where to look for updates. Any fields available in a `Subscription` `.spec` can be specified here.

- **`versions` (optional)**: This field accepts a list of allowed Operator versions. When specified, the installed version of the Operator on the cluster must match one of the versions in this list for the policy to be considered compliant. If not specified, any version of the Operator can be deemed compliant.

Below is a manifest that installs the OpenShift Logging operator, as tested in a lab environment.

**Observations**:

- When `complianceType` is set to 'inform', the OperatorPolicy reports on the status of the Operator in the managed cluster, indicating whether it is installed (compliant) or not installed (non-compliant). This means that the OperatorPolicy will not enforce the installation or removal of the Operator but will provide information about its current state.

**Note**:

- The `OperatorPolicy` `.spec.subscription.name` defines both the Package name as listed in the catalog and the `Subscription` `metadata.name` field. In other words, the `.metadata.name` and `.spec.name` of a Subscription managed by the OperatorPolicy will always be identical.

