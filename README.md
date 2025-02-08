---
# tasks file for ansible/roles/rhacm-policy-validator
- name: Install pip packages
  ## become: true
  ansible.builtin.pip:
    name:
      - jmespath
      - kubernetes
      - requests

- name: Create report directory
  ansible.builtin.file:
    path: "{{ report_dir | default('./reports') }}/tmp"
    state: directory
    mode: '0755'

- name: Create var to display date/time
  ansible.builtin.set_fact:
    date_time: "{{ '%Y-%m-%d %X' | strftime }}"

- name: Prepare hub configurations
  ansible.builtin.set_fact:
    hub_clusters: "{{ hub_clusters | default([]) + [{'name': item, 'kubeconfig': kubeconfig_path}] }}"
  loop: "{{ clusters }}"

- name: Capture pre-deployment compliance state
  rhacm_policy_validator:
    hub_configs: "{{ hub_clusters }}"
    namespaces: "{{ policy_namespaces }}"
    state: pre
    state_file: "{{ report_dir }}/tmp/pre_deployment_state.json"

- name: Validate post-deployment compliance
  rhacm_policy_validator:
    hub_configs: "{{ hub_clusters }}"
    namespaces: "{{ policy_namespaces }}"
    state: post
    state_file: "{{ report_dir }}/tmp/pre_deployment_state.json"
    wait_time: "{{ wait_time | default(300) }}"  # Adjust based on your needs
  register: validation_result

- name: Generate validation report
  ansible.builtin.template:
    src: summary-report.html.j2
    dest: "{{ report_dir }}/summary-report.html"
    mode: '0644'
  vars:
    comparison: "{{ validation_result.comparison }}"
    compliance_state: "{{ validation_result.compliance_state }}"
    ## date_time: "{{ date_time }}"
