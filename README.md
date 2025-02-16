mkdir -pv ~/ee

cd ~/ee

cat <<EOF > bindep.txt
gcc [platform:rpm]
make [platform:rpm]
which [platform:rpm]
EOF

cat <<EOF > execution-environment.yaml
---
version: 3

build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: '--pre'

options:
  ## Fix for "/usr/bin/dnf: No such file or directory"
  package_manager_path: /usr/bin/microdnf

dependencies:
  galaxy: requirements.yaml
  python: requirements.txt
  system: bindep.txt

images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:2.17.8-4

additional_build_steps:
  ## Fix for When building an EE which includes the redhat.openshift collection
  prepend_builder:
    - ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --setopt=rhocp-4.17-for-rhel-9-x86_64-rpms.enabled=true"
  prepend_final:
    - ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --setopt=rhocp-4.17-for-rhel-9-x86_64-rpms.enabled=true"
    - RUN microdnf clean all
EOF

cat <<EOF > requirements.txt
ansible-modules-hashivault
requests-oauthlib
cryptography
kubernetes
jsonpatch
pynetbox
requests
jmespath
paramiko
PyYAML
EOF

cat <<EOF > requirements.yaml
---
collections:
  - name: ansible.posix
  - name: community.general
  - name: community.crypto

# Include collections below if targeting a Kubernetes /OpenShift environment.
  - name: community.kubernetes
  - name: kubernetes.core
  - name: community.okd

# Include collection below if building a virtual host.
  ## - name: https://github.com/ansible-collections/community.libvirt
  ##   type: git
  ##   version: main

# Include collections below if playbook deploys to cloud.
  ## - name: community.aws
  ## - name: google.cloud
  ## - name: community.azure
  ## - name: community.azure
  ## - name: community.aws
  ## - name: community.digitalocean

  # Added for community.dns.hetzner_dns_record.
  ## - name: community.dns

  # Added for community.docker
  ## - name: community.docker
  EOF

  ansible-builder build --t quay.io/rh_ee_iijere/rh-ocp-ee:2025021501 --prune-images -v3
