object-templates-raw: |
  {{- $configData := (lookup "v1" "ConfigMap" "mks-registry-controller" "lift-providers").data }}
  {{- $providers := fromJson $configData.providers }}
  {{- $devProviders := $providers.dev }}
  {{- $prodProviders := $providers.prod }}
  - complianceType: mustonlyhave
    objectDefinition:
      apiVersion: constraints.gatekeeper.sh/v1beta1
      kind: MksAllowedRegistries
      metadata:
        name: mks-allowed-registries
      spec:
        parameters:
          registries:
            {{- range $provider := $devProviders }}
            - "{{ $provider }}-eonid-11111111.docker.dev.up.lift.ms.com/"
            {{- end }}
            {{- range $provider := $prodProviders }}
            - "{{ $provider }}-eonid-11111111.docker.prod.up.lift.ms.com/"
            {{- end }}
