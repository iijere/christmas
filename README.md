RH ACM Engineering's workaround works. I can apply overrides to values in the Helm chart's values.yaml file.

The workaround:

1. Create a Channel object on the HUB that points to the Helm repo. This can be managed by an ACM Policy with Placement targeting the HUB.
2. Create an ACM Policy that is propagated to the managed cluster. The policy contains a Subscription which subscribes to the Channel with the following details:
2.1. Add an apps.open-cluster-management.io/hosting-subscription=<sub-ns/sub-name> annotation to indicate it is not a HUB appsub.
2.2 Set the Subscription's spec.placement to local: true, which indicates to the Subscription that it is a standalone Placement without needing a Placement reference.

Any overrides to variables defined in the Helm Chart's values.yaml file should be included in the Subscription.

Note: The Channel object created on the HUB must exist in the same namespace as the policy containing the Subscription object that is propagated to the managed cluster. The Subscription object on the managed cluster will not reconcile if the Channel object on the HUB is in a different namespace.
