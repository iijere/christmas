apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: ijljere
  namespace: bry-tam-policies-prod
spec:
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: cluster-roles
        spec:
          object-templates:
            - complianceType: mustonlyhave
              objectDefinition:
                apiVersion: rbac.authorization.k8s.io/v1
                kind: ClusterRole
                metadata:
                  annotations:
                    rbac.authorization.ms.com/type: tam
                  labels:
                    rbac.authorization.ms.com/aggregate-to-cluster-owner: 'true'
                  name: resource-utilization-read
                rules:
                  - apiGroups:
                      - ''
                    resources:
                      - resourcequotas
                    verbs:
                      - get
                      - list
                      - watch
            - complianceType: musthave
              objectDefinition:
                aggregationRule:
                  clusterRoleSelectors:
                    - matchLabels:
                        rbac.authorization.ms.com/aggregate-to-cluster-owner: 'true'
                apiVersion: rbac.authorization.k8s.io/v1
                kind: ClusterRole
                metadata:
                  annotations:
                    rbac.authorization.ms.com/type: tam
                  name: cluster-owner
          remediationAction: enforce
          severity: low
