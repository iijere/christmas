#!/usr/bin/env python3

DOCUMENTATION = r'''
---
module: rhacm_policy_validator
short_description: Validates RHACM policies across Prime clusters
version_added: "1.0.0"
description:
    - Module for validating the compliance status of Red Hat Advanced Cluster Management (RHACM) policies across multiple prime clusters
    - Captures policy compliance state before and after RHACM policy release
    - Generates detailed reports highlighting policy changes and violations
    - Provides a consolidated view of non-compliant policies across all prime clusters

options:
    hub_configs:
        description:
            - List of prime cluster configurations
        required: true
        type: list
        elements: dict
        suboptions:
            name:
                description: Name identifier for the prime cluster, example:
                required: true
                type: str
            kubeconfig:
                description: Path to the kubeconfig file for prime cluster
                required: true
                type: str
    
    namespaces:
        description:
            - List of namespaces to monitor for policies
        required: true
        type: list
        elements: str
    
    state:
        description:
            - Determines whether to capture pre-deployment or post-deployment state
            - 'pre' captures initial policy state before policy release
            - 'post' captures the final policy state after release, and generates a comparison report
        required: true
        type: str
        choices: ['pre', 'post']
    
    state_file:
        description:
            - Path to store/retrieve the pre-deployment state data
        required: true
        type: str
    
    wait_time:
        description:
            - Time in seconds to wait before capturing post-deployment state
            - Allows policies to stabilize after changes
        required: false
        type: int
        default: 300

requirements:
    - kubernetes >= 12.0.0
    - PyYAML >= 5.1
    - jmespath
    - requests

attributes:
    check_mode:
        support: none
    diff_mode:
        support: none

author:
    - "iijere@redhat.com"

notes:
    - This module requires appropriate RBAC permissions to access RHACM policies
    - Kubeconfig files must have the necessary credentials for prime cluster access
    - Pre-deployment state is stored as JSON in the specified state file
    - Generated report includes both policy changes and current non-compliant state

example_output_dict:
    changed:
        description: Indicates if any policies became non-compliant
        type: bool
        example: true
    compliance_state:
        description: Current state of all policies across hub clusters
        type: dict
    comparison:
        description: Comparison between pre and post-states
        type: dict
        contains:
            summary:
                description: Statistical overview of policy states
                type: dict
                contains:
                    total_hubs: Number of hub clusters monitored
                    total_policies: Total number of policies across all hubs
                    compliant: Number of compliant policies
                    total_noncompliant: Number of non-compliant policies
                    newly_noncompliant: Number of policies that became non-compliant
                    newly_compliant: Number of policies that became compliant
                    changed: Total number of policies that changed state
            consolidated_view:
                description: Detailed view of policy changes and current state
                type: dict

examples:
    - name: Capture pre-deployment policy state
      rhacm_policy_validator:
        hub_configs:
            - name: prime1.
              kubeconfig: /path/to/prime1.kubeconfig
            - name: prime2
              kubeconfig: /path/to/prime2.kubeconfig
        namespaces:
            - policies
            - rhacm-prod
        state: pre
        state_file: /tmp/pre_deployment_state.json

    - name: Validate post-deployment policy state
      rhacm_policy_validator:
        hub_configs:
            - name: prime1.
              kubeconfig: /path/to/prime1.kubeconfig
            - name: prime2
              kubeconfig: /path/to/prime2.kubeconfig
        namespaces:
            - policies
            - rhacm-prod
        state: post
        state_file: /tmp/pre_deployment_state.json
        wait_time: 300

    - name: Generate HTML report with policy validation results
      template:
        src: policy-release-report.j2
        dest: /path/to/policy-release-report.html
      vars:
        comparison: "{{ validation_result.comparison }}"
        compliance_state: "{{ validation_result.compliance_state }}"

report_sections:
    header:
        description: Displays total hub clusters and non-compliant managed clusters
    
    dashboard:
        description: Visual overview of policy compliance
        contains:
            - Donut chart showing compliance distribution
            - Bar chart showing non-compliant policies by hub
            - Summary statistics cards
    
    policy_changes:
        description: Lists policies that changed compliance status
        features:
            - Color-coded cards (red for non-compliant, green for compliant)
            - Policy details and remediation actions
            - Affected clusters and console links
    
    all_non_compliant:
        description: Groups all currently non-compliant policies
        features:
            - Policies grouped by name across hubs
            - Per-hub violation details
            - Console links for detailed investigation

notes_on_html_report:
    - Uses TailwindCSS for styling
    - Includes Charts.js for visualizations
    - Responsive design for various screen sizes
    - Interactive elements like hover states and console links
'''

from ansible.module_utils.basic import AnsibleModule
from datetime import datetime
from kubernetes import client, config
from typing import Dict, List, Optional
import logging
import json
import time
import concurrent.futures

class ConsoleURLBuilder:
    """Generates RHACM console URLs for policy viewing"""
    def __init__(self, custom_api: client.CustomObjectsApi, hub_name: str):
        self.hub_name = hub_name
        self.custom_api = custom_api
        self._console_host = None

    def get_console_host(self) -> Optional[str]:
        """Get and cache the console route host"""
        if not self._console_host:
            try:
                route = self.custom_api.get_namespaced_custom_object(
                    group="route.openshift.io",
                    version="v1",
                    namespace="openshift-console",
                    plural="routes",
                    name="console"
                )
                self._console_host = route.get('spec', {}).get('host')
            except Exception as e:
                logging.error(f"Failed to get console route: {e}")
        return self._console_host

    def get_policy_url(self, policy_name: str, namespace: str) -> Optional[str]:
        """Generate policy-specific console URL"""
        host = self.get_console_host()
        if host:
            return f"https://{host}/multicloud/governance/policies/details/{namespace}/{policy_name}/results"
        return None

class PolicyValidator:
    """Handles policy validation for a single hub cluster"""
    def __init__(self, kubeconfig: str, hub_name: str):
        self.hub_name = hub_name
        config.load_kube_config(kubeconfig)
        self.custom_api = client.CustomObjectsApi()
        self.url_builder = ConsoleURLBuilder(self.custom_api, hub_name)

    def get_policies(self, namespace: str) -> Dict:
        """Get current policy compliance status"""
        try:
            policies = self.custom_api.list_namespaced_custom_object(
                group="policy.open-cluster-management.io",
                version="v1",
                namespace=namespace,
                plural="policies"
            ).get('items', [])

            compliance_data = {}
            for policy in policies:
                policy_name = policy.get('metadata', {}).get('name')
                if not policy_name:
                    continue

                status = policy.get('status', {})
                compliance_status = status.get('compliant')
                
                # Use a composite key that includes namespace to avoid collisions
                policy_key = f"{namespace}/{policy_name}"
                
                # Store ALL policies, not just non-compliant ones
                compliance_data[policy_key] = {
                    'overall_compliance': compliance_status,
                    'cluster_status': self._get_cluster_status(policy_name, status, namespace) if compliance_status == 'NonCompliant' else {},
                    'namespace': namespace,
                    'details': self._get_policy_details(policy)
                }

            return compliance_data
        except Exception as e:
            logging.error(f"Failed to get policies: {e}")
            raise

    def _get_cluster_status(self, policy_name: str, status: Dict, namespace: str) -> Dict:
        """Process cluster-specific compliance status"""
        cluster_status = {}
        console_url = self.url_builder.get_policy_url(policy_name, namespace)

        for cluster_info in status.get('status', []):
            cluster_name = cluster_info.get('clustername')
            if cluster_name and cluster_info.get('compliant') == 'NonCompliant':
                cluster_status[cluster_name] = {
                    'compliant': 'NonCompliant',
                    'console_url': console_url
                }
        return cluster_status

    def _get_policy_details(self, policy: Dict) -> Dict:
        """Extract relevant policy details"""
        metadata = policy.get('metadata', {})
        spec = policy.get('spec', {})
        templates = spec.get('policy-templates', [])
        
        remediation = 'inform'
        if templates:
            remediation = templates[0].get('objectDefinition', {}).get('spec', {}).get('remediationAction', 'inform')
        
        return {
            'name': metadata.get('name', 'unknown'),
            'remediation_action': remediation,
            'description': spec.get('description', '')
        }

class MultiHubValidator:
    """Orchestrates policy validation across multiple hubs"""
    def __init__(self, hub_configs: List[Dict[str, str]]):
        self.validators = {}
        for config in hub_configs:
            try:
                validator = PolicyValidator(
                    kubeconfig=config['kubeconfig'],
                    hub_name=config['name']
                )
                self.validators[config['name']] = validator
            except Exception as e:
                logging.error(f"Failed to initialize hub {config['name']}: {e}")

    def get_compliance_state(self, namespaces: List[str]) -> Dict:
        """Get non-compliant policies from all hubs"""
        hub_data = {}
        with concurrent.futures.ThreadPoolExecutor() as executor:
            futures = []
            for hub_name, validator in self.validators.items():
                for namespace in namespaces:
                    future = executor.submit(validator.get_policies, namespace)
                    futures.append((hub_name, namespace, future))

            for hub_name, namespace, future in futures:
                try:
                    if hub_name not in hub_data:
                        hub_data[hub_name] = {}
                    hub_data[hub_name].update(future.result())
                except Exception as e:
                    logging.error(f"Error getting compliance for {hub_name}: {e}")

        return hub_data

    def generate_report(self, pre_state: Dict, post_state: Dict) -> Dict:
        """Generate comparison report between states"""
        return {
            'summary': self._generate_summary(pre_state, post_state),
            'consolidated_view': self._generate_consolidated_view(pre_state, post_state)
        }

    def _generate_summary(self, pre_state: Dict, post_state: Dict) -> Dict:
        """Generate overall summary statistics"""
        total_policies = 0
        total_compliant = 0
        total_noncompliant = 0
        policies_became_noncompliant = set()
        policies_became_compliant = set()
        noncompliant_by_hub = {}
        seen_policies = set()
        
        # Use a set of tuples (hub_name, cluster_name) instead of just cluster_name
        noncompliant_managed_clusters = set()
        
        # Process each hub's policies
        for hub_name, hub_policies in post_state.items():
            hub_noncompliant = 0
            pre_hub_state = pre_state.get(hub_name, {})
            
            for policy_name, policy in hub_policies.items():
                # Only count unique policies
                policy_key = (policy_name, policy.get('namespace', ''))
                if policy_key not in seen_policies:
                    seen_policies.add(policy_key)
                    total_policies += 1
                    
                    # Count compliant vs non-compliant
                    if policy.get('overall_compliance') == 'NonCompliant':
                        total_noncompliant += 1
                        # Track non-compliant managed clusters with hub context
                        cluster_status = policy.get('cluster_status', {})
                        for cluster_name in cluster_status.keys():
                            noncompliant_managed_clusters.add((hub_name, cluster_name))
                    elif policy.get('overall_compliance') == 'Compliant':
                        total_compliant += 1

                # Track non-compliant policies per hub
                if policy.get('overall_compliance') == 'NonCompliant':
                    hub_noncompliant += 1

                # Check for policy status changes
                pre_policy = pre_hub_state.get(policy_name, {})
                if pre_policy:
                    pre_compliance = pre_policy.get('overall_compliance')
                    post_compliance = policy.get('overall_compliance')
                    if pre_compliance != post_compliance:
                        if post_compliance == 'NonCompliant':
                            policies_became_noncompliant.add(policy_key)
                        elif post_compliance == 'Compliant':
                            policies_became_compliant.add(policy_key)

            noncompliant_by_hub[hub_name] = hub_noncompliant

        return {
            'total_hubs': len(post_state),
            'total_noncompliant_clusters': len(noncompliant_managed_clusters),  # Now counts unique hub+cluster combinations
            'total_policies': total_policies,
            'compliant': total_compliant,
            'total_noncompliant': total_noncompliant,
            'newly_noncompliant': len(policies_became_noncompliant),
            'newly_compliant': len(policies_became_compliant),
            'changed': len(policies_became_noncompliant) + len(policies_became_compliant),
            'cluster_names': list(noncompliant_by_hub.keys()),
            'cluster_noncompliant': list(noncompliant_by_hub.values())
        }

    def _generate_consolidated_view(self, pre_state: Dict, post_state: Dict) -> Dict:
        """Generate consolidated view of policy changes"""
        policies = {}
        namespaces = set()
        
        for hub_name, hub_policies in post_state.items():
            pre_hub_state = pre_state.get(hub_name, {})
            
            for policy_key, policy in hub_policies.items():
                hub_namespace = policy['namespace']
                namespaces.add(hub_namespace)
                
                # Extract policy name from the composite key
                # The format is namespace/policy_name
                key_parts = policy_key.split('/', 1)
                if len(key_parts) != 2:
                    continue  # Skip invalid keys
                    
                policy_namespace = key_parts[0]
                policy_name = key_parts[1]
                
                # Look for pre-state with same composite key
                pre_policy = pre_hub_state.get(policy_key, {})
                pre_compliance = pre_policy.get('overall_compliance')
                post_compliance = policy.get('overall_compliance')
                
                # Track both newly non-compliant and newly compliant policies
                if pre_compliance != post_compliance:
                    # Create a unique display key that includes namespace
                    display_key = f"{policy_name}_{policy_namespace}"
                    
                    if display_key not in policies:
                        policies[display_key] = {
                            'details': policy['details'],
                            'violation_count': 0,
                            'status_change': 'became_noncompliant' if post_compliance == 'NonCompliant' else 'became_compliant',
                            'hubs': {},
                            'policy_name': policy_name,
                            'namespace': policy_namespace
                        }

                    # Add hub-specific information
                    policies[display_key]['hubs'][hub_name] = {
                        'namespace': hub_namespace,
                        'previous_status': pre_compliance,
                        'current_status': post_compliance,
                        'noncompliant_clusters': (
                            [
                                {'cluster': cluster, 'console_url': data['console_url']}
                                for cluster, data in policy['cluster_status'].items()
                            ] if post_compliance == 'NonCompliant' else []
                        )
                    }
                    
                    if post_compliance == 'NonCompliant':
                        policies[display_key]['violation_count'] += len(
                            policies[display_key]['hubs'][hub_name]['noncompliant_clusters']
                        )

        return {
            'policies': policies,
            'namespaces': sorted(list(namespaces))
        }

    def _count_new_noncompliant(self, pre_state: Dict, post_state: Dict) -> int:
        """Count newly non-compliant policies"""
        count = 0
        for hub_name, hub_policies in post_state.items():
            pre_hub = pre_state.get(hub_name, {})
            for policy_name in hub_policies:
                if (policy_name not in pre_hub or
                    pre_hub[policy_name].get('overall_compliance') != 'NonCompliant'):
                    count += 1
        return count

def main():
    module = AnsibleModule(
        argument_spec=dict(
            hub_configs=dict(type='list', required=True, elements='dict'),
            namespaces=dict(type='list', required=True, elements='str'),
            state=dict(type='str', required=True, choices=['pre', 'post']),
            state_file=dict(type='str', required=True),
            wait_time=dict(type='int', default=300)
        )
    )

    try:
        validator = MultiHubValidator(module.params['hub_configs'])
        result = {'changed': False}

        if module.params['state'] == 'pre':
            result['compliance_state'] = validator.get_compliance_state(
                module.params['namespaces']
            )
            with open(module.params['state_file'], 'w') as f:
                json.dump(result['compliance_state'], f)
        else:
            if module.params['wait_time'] > 0:
                time.sleep(module.params['wait_time'])

            try:
                with open(module.params['state_file'], 'r') as f:
                    pre_state = json.load(f)
            except FileNotFoundError:
                module.fail_json(msg="Pre-deployment state file not found")

            post_state = validator.get_compliance_state(module.params['namespaces'])
            comparison = validator.generate_report(pre_state, post_state)

            result.update({
                'compliance_state': post_state,
                'comparison': comparison,
                'changed': comparison['summary']['newly_noncompliant'] > 0
            })

        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=str(e))

if __name__ == '__main__':
    main()
