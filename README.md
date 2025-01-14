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

- name: Capture pre-deployment compliance state
  rhacm_policy_validator:
    kubeconfig: "{{ kubeconfig_path }}"
    namespace: "{{ policy_namespace }}"
    state: pre
    state_file: "{{ report_dir }}/tmp/pre_deployment_state.json"

- name: Validate post-deployment compliance
  rhacm_policy_validator:
    kubeconfig: "{{ kubeconfig_path }}"
    namespace: "{{ policy_namespace }}"
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

- name: Print summary
  ansible.builtin.debug:
    msg: |
      =============================================================
      {{ "%-35s" | format(RHACM Policy Validation Report) }}
      =============================================================
      Generated: {{ ansible_date_time.iso8601 }}

      SUMMARY
      =============================================================
      Total Policies........: {{ validation_result.comparison.summary.total_policies }}
      Managed Clusters.....: {{ validation_result.comparison.summary.all_managed_clusters|length }}
      Policies Changed.....: {{ validation_result.comparison.summary.policies_changed }}
      Currently Non-Compliant: {{ validation_result.comparison.summary.currently_noncompliant }}


- name: Create report file
  ansible.builtin.copy:
    content: |
      =============================================================
                    RHACM Policy Validation Report                  
      =============================================================
      Generated: {{ ansible_date_time.iso8601 }}

      SUMMARY
      =============================================================
      Total Policies........: {{ validation_result.comparison.summary.total_policies }}
      Managed Clusters.....: {{ validation_result.comparison.summary.all_managed_clusters|length }}
      Policies Changed.....: {{ validation_result.comparison.summary.policies_changed }}
      Currently Non-Compliant: {{ validation_result.comparison.summary.currently_noncompliant }}

      {% if validation_result.comparison.summary.currently_noncompliant > 0 %}
      NON-COMPLIANT POLICIES
      =============================================================
      {% for policy_name, policy in validation_result.comparison.policies.items() %}
      {% if policy.noncompliant_clusters %}
      [{{ policy.details.remediation_action|upper }}] {{ policy_name }}
      -----------------------------------------------------------
      Non-Compliant Clusters:
      {% for cluster in policy.noncompliant_clusters %}
      * {{ cluster.cluster | join('\n') }}
        {% if cluster.message %}Message: {{ cluster.message }}{% endif %}
      {% endfor %}

      {% endif %}
      {% endfor %}
      {% endif %}

      POLICY DEPLOYMENT CHANGES
      =============================================================
      {% for policy_name, policy in validation_result.comparison.policies.items() %}
      {% if policy.changed %}
      [{{ policy.details.remediation_action|upper }}] {{ policy_name }}
      -----------------------------------------------------------
      Status Change: {{ policy.pre_compliance }} → {{ policy.post_compliance }}
      
      Affected Clusters:
      {% for change in policy.cluster_changes %}
      * {{ change.cluster }}
        Before: {{ change.pre_status }}
        After : {{ change.post_status }}
      {% endfor %}

      {% endif %}
      {% endfor %}
      {% if not validation_result.comparison.summary.policies_changed %}
      No policy changes detected during deployment.
      {% endif %}
    dest: "/tmp/rhacm_validation_report.txt"
    mode: '0644'

- name: Display validation report
  ansible.builtin.debug:
    msg: "{{ lookup('file', '/tmp/rhacm_validation_report.txt') | split('\n') }}"

- name: Clean up report file
  ansible.builtin.file:
    path: "/tmp/rhacm_validation_report.txt"
    state: absent

## - name: Generate validation report in logs
##   ansible.builtin.command: "{{ ansible_playbook_python }}"
##   args:
##     stdin: |
##       print("=============================================================")
##       print("              RHACM Policy Validation Report                 ")
##       print("=============================================================")
##       print("""Generated: {{ ansible_date_time.iso8601 }}""")

## - name: Validate policy compliance
##   rhacm_policy_validator:
##     kubeconfig: "{{ kubeconfig_path }}"
##     namespace: "{{ policy_namespace }}"
##     wait_time: "{{ wait_time | default(300) }}" ## Default is a 5 minutes wait time.
##   register: validation_result
##   vars:
##     ansible_python_interpreter: "{{ ansible_playbook_python }}"
##   environment:
##     KUBECONFIG: "{{ kubeconfig_path }}"
## 
## - name: Generate summary report in markdown
##   ansible.builtin.template:
##     src: summary-report.j2
##     dest: "{{ report_dir }}/summary-report.md"
##     mode: '0644'
##   when: validation_result.validation_results | length > 0
##   vars:
##     results: "{{ validation_result.validation_results }}"
## 
## - name: Generate summary report in HTML
##   ansible.builtin.template:
##     src: summary-report.html.j2
##     dest: "{{ report_dir }}/summary-report.html"
##     mode: '0644'
##   when: validation_result.validation_results | length > 0
## 
## - name: Generate compliance reports in markdown
##   ansible.builtin.template:
##     src: compliance-report.j2
##     dest: "{{ report_dir }}/{{ item.policy_name }}-compliance-report.md"
##     mode: '0644'
##   loop: "{{ validation_result.validation_results }}"
## 
## - name: Show report location
##   ansible.builtin.debug:
##     msg: "Report generated at: {{ report_dir }}"
## 
## - name: Check for non-compliant policies
##   ansible.builtin.fail:
##     msg: "Policy {{ item.policy_name }} is non-compliant"
##   when: item.after == 'NonCompliant'
##   loop: "{{ validation_result.validation_results }}"
