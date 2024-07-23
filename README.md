## Ho! Ho!! Ho!!!

.
├── placements
│   ├── base
│   │   ├── kustomization.yaml
│   │   ├── managedclsetbinding.yaml
│   │   └── plcmnt-cluster-environment.yaml
│   ├── prod
│   │   └── kustomization.yaml
│   └── staging
│       └── kustomization.yaml
├── policies
│   ├── kustomization.yaml
│   └── policy-web-app
│       ├── kustomization.yaml
│       ├── manifests
│       │   ├── configmap.yaml
│       │   ├── deployment.yaml
│       │   ├── namespace.yaml
│       │   ├── route.yaml
│       │   └── service.yaml
│       └── policygenerator.yaml
├── policy-overlays
│   ├── prod
│   │   └── kustomization.yaml
│   └── staging
│       └── kustomization.yaml
└── subscriptions
    ├── base
    │   ├── kustomization.yaml
    │   └── subscription.yaml
    ├── prod
    │   └── kustomization.yaml
    └── staging
        └── kustomization.yaml
