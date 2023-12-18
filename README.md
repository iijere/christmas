```bash
gitops
├── cluster-sets
│   ├── base
│   │   ├── kustomization.yaml
│   │   ├── managed-clusterset.yaml
│   │   └── namespace.yaml
│   ├── main
│   │   ├── prime-app-all
│   │   ├── uni-all
│   │   └── uni-all-l0
│   └── staging
│       ├── kustomization.yaml
│       ├── prime-app-all
│       │   ├── kustomization.yaml
│       │   ├── managed-clusterset-binding.yaml
│       │   ├── patch-managed-clusterset.yaml
│       │   ├── patch-namespace.yaml
│       │   └── placement.yaml
│       ├── uni-all
│       └── uni-all-l0
└── policies
    ├── policy-alertforwarder
    │   └── manifests
    └── policy-cluster-monitoring
        ├── kustomization.yaml
        ├── manifests
        │   ├── cluster-monitoring-config-cm.yaml
        │   ├── cluster-object-2.yaml
        │   ├── cluster-object-3.yaml
        │   └── cluster-object-n.yaml
        └── policygenerator.yaml
```
