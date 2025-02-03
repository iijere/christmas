#!/usr/bin/env python3

ANSIBLE_METADATA = {
    'metadata_version': '1.1',
    'status': ['preview'],
    'supported_by': 'community'
}

DOCUMENTATION = r'''
---
module: rhacm_policy_validator

short_description: Validates Red Hat Advanced Cluster Management (RHACM) policies across multiple hub clusters

version_added: "1.0.0"

This Ansible module validates and reports on the compliance status of RHACM policies across multiple hub clusters.
It captures pre and post-deployment states of policies and generates detailed comparisons, including direct links
to policy details in the RHACM console.

Module Features:
    - Multi-hub cluster support with concurrent policy validation
    - Pre and post-deployment state comparison
    - Console URL generation for non-compliant policies
    - Detailed policy compliance reporting
    - Support for multiple managed clusters per hub

Options:
    hub_configs:
        description:
            - List of hub cluster configurations.
            - Each hub configuration must include 'name' and 'kubeconfig' fields.
        required: true
        type: list
        elements: dict
        example:
            - name: "hub1"
              kubeconfig: "/path/to/hub1/kubeconfig"
            - name: "hub2"
              kubeconfig: "/path/to/hub2/kubeconfig"
    
    namespace:
        description: Namespace where RHACM policies are located
        required: true
        type: str
    
    state:
        description:
            - Whether this is pre or post-deployment validation
            - 'pre' captures initial state
            - 'post' compares current state with saved pre-deployment state
        required: true
        choices: ['pre', 'post']
        type: str
    
    state_file:
        description: Path where pre-deployment state will be saved/loaded
        required: true
        type: str
    
    wait_time:
        description: Time to wait before post-deployment validation (seconds)
        required: false
        default: 300
        type: int

Requirements:
    - Python >= 3.6
    - kubernetes >= 12.0.0
    - PyYAML
    - requests

Classes:
    ConsoleURLBuilder:
        Handles generation of RHACM console URLs for policy viewing.
        Methods:
            - get_console_host(): Gets console route host for a hub cluster
            - get_policy_url(policy_name): Generates policy-specific console URL
    
    SingleHubValidator:
        Manages policy validation for a single hub cluster.
        Methods:
            - get_cluster_compliance(): Gets current policy compliance status
            - compare_states(): Compares pre/post states of policies
    
    MultiHubPolicyValidator:
        Orchestrates policy validation across multiple hub clusters.
        Methods:
            - get_all_hub_compliance(): Gets compliance data from all hubs
            - compare_multi_hub_states(): Compares states across all hubs

author:
    - ij ijere (iijere@redhat.com)

notes:
    - Requires appropriate RBAC permissions on the hub clusters
    - Kubeconfig files must have valid credentials
    - Tested with RHACM version 2.12+
    - Performance depends on number of policies and managed clusters

seealso:
    - name: RHACM Documentation
      link: https://access.redhat.com/documentation/red_hat_advanced_cluster_management
    - name: Kubernetes Python Client
      link: https://github.com/kubernetes-client/python
'''

EXAMPLES = r'''
# Capture pre-deployment state
- name: Capture pre-deployment compliance state
    rhacm_policy_validator:
    hub_configs:
        - name: hub1
        kubeconfig: /path/to/hub1/kubeconfig
        - name: hub2
        kubeconfig: /path/to/hub2/kubeconfig
    namespace: policies
    state: pre
    state_file: /tmp/pre_deployment_state.json

# Validate post-deployment state
- name: Validate post-deployment compliance
    rhacm_policy_validator:
    hub_configs:
        - name: hub1
        kubeconfig: /path/to/hub1/kubeconfig
        - name: hub2
        kubeconfig: /path/to/hub2/kubeconfig
    namespace: policies
    state: post
    state_file: /tmp/pre_deployment_state.json
    wait_time: 60
    register: validation_result
'''

RETURN = r'''
changed:
    description: Whether any policies changed compliance state
    type: bool
    returned: always

compliance_state:
    description: Current compliance state of policies
    type: dict
    returned: always
    contains:
        hub_name:
            description: Name of hub cluster
            type: dict
            contains:
                policy_name:
                    description: Name of policy
                    type: dict
                    contains:
                        overall_compliance:
                            description: Overall compliance status
                            type: str
                        cluster_status:
                            description: Per-cluster compliance status
                            type: dict
                        console_url:
                            description: URL to view policy in RHACM console
                            type: str

comparison:
    description: Comparison between pre and post states
    type: dict
    returned: when state=post
    contains:
        summary:
            description: Overall statistics
            type: dict
        hubs:
            description: Per-hub comparison details
            type: dict
'''

from ansible.module_utils.basic import AnsibleModule
from datetime import datetime
from kubernetes import client, config
from typing import Dict, List, Optional, Set
import logging
import json
import time
import concurrent.futures

class ConsoleURLBuilder:
    def __init__(self, custom_api: client.CustomObjectsApi, hub_name: str):
        self.hub_name = hub_name
        self.custom_api = custom_api
        self.logger = logging.getLogger(f'ConsoleURLBuilder-{hub_name}')
        self._console_host = None

    def get_console_host(self) -> str:
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
                if not self._console_host:
                    self.logger.error(f"No host found in console route for hub {self.hub_name}")
            except Exception as e:
                self.logger.error(f"Failed to get console route for hub {self.hub_name}: {e}")
        return self._console_host

    def get_policy_url(self, policy_name: str) -> Optional[str]:
        """Get the policy console URL"""
        host = self.get_console_host()
        if host:
            return f"https://{host}/multicloud/governance/policies/details/policies/{policy_name}/results"
        return None

class SingleHubValidator:
    def __init__(self, kubeconfig: str, hub_name: str):
        self.hub_name = hub_name
        self.logger = logging.getLogger(f'SingleHubValidator-{hub_name}')
        try:
            config.load_kube_config(kubeconfig)
            self.custom_api = client.CustomObjectsApi()
            self.url_builder = ConsoleURLBuilder(self.custom_api, hub_name)
        except Exception as e:
            self.logger.error(f"Kubernetes client initialization failed for hub {hub_name}: {e}")
            raise

    def get_cluster_compliance(self, namespace: str) -> Dict:
        """Get compliance status for all policies in namespace"""
        try:
            policies = self._get_policies(namespace)
            compliance_data = {}
            
            for policy in policies:
                policy_name = policy.get('metadata', {}).get('name')
                if not policy_name:
                    continue
                
                status = policy.get('status', {})
                cluster_statuses = self._process_cluster_statuses(policy_name, status)
                
                compliance_data[policy_name] = {
                    'overall_compliance': status.get('compliant', 'Unknown'),
                    'cluster_status': cluster_statuses,
                    'timestamp': datetime.now().isoformat(),
                    'details': self._get_policy_details(policy)
                }
            
            return compliance_data
        except Exception as e:
            self.logger.error(f"Failed to get policy compliance: {e}")
            raise

    def _get_policies(self, namespace: str) -> List:
        return self.custom_api.list_namespaced_custom_object(
            group="policy.open-cluster-management.io",
            version="v1",
            namespace=namespace,
            plural="policies"
        ).get('items', [])

    def _process_cluster_statuses(self, policy_name: str, status: Dict) -> Dict:
        cluster_status = {}
        console_url = self.url_builder.get_policy_url(policy_name)
        self.logger.info(f"Generated console URL for policy {policy_name}: {console_url}")

        for cluster_info in status.get('status', []):
            cluster_name = cluster_info.get('clustername')
            cluster_ns = cluster_info.get('clusternamespace')
            
            if cluster_name and cluster_ns:
                cluster_status[cluster_name] = {
                    'compliant': cluster_info.get('compliant', 'Unknown'),
                    'console_url': console_url,
                    'cluster_namespace': cluster_ns
                }
        return cluster_status

    def _get_policy_details(self, policy: Dict) -> Dict:
        try:
            metadata = policy.get('metadata', {})
            spec = policy.get('spec', {})
            templates = spec.get('policy-templates', [])
            
            remediation = 'inform'
            if templates:
                template_spec = templates[0].get('objectDefinition', {}).get('spec', {})
                remediation = template_spec.get('remediationAction', 'inform')
            
            return {
                'name': metadata.get('name', 'unknown'),
                'remediation_action': remediation,
                'description': spec.get('description', '')
            }
        except Exception as e:
            self.logger.error(f"Error extracting policy details: {e}")
            return {'name': 'unknown', 'remediation_action': 'inform', 'description': ''}

    def compare_states(self, pre_state: Dict, post_state: Dict) -> Dict:
        all_clusters = self._get_all_clusters([pre_state, post_state])
        
        comparison = {
            'policies': {},
            'summary': {
                'total_policies': len(post_state),
                'policies_changed': 0,
                'currently_noncompliant': 0,
                'clusters_with_changes': set(),
                'all_managed_clusters': all_clusters
            }
        }

        for policy_name in set(pre_state) | set(post_state):
            pre = pre_state.get(policy_name, {})
            post = post_state.get(policy_name, {})
            
            policy_comparison = self._compare_policy_states(pre, post, policy_name)
            
            if policy_comparison['changed']:
                comparison['summary']['policies_changed'] += 1
                comparison['summary']['clusters_with_changes'].update(
                    c['cluster'] for c in policy_comparison['cluster_changes']
                )
            
            if policy_comparison['post_compliance'] == 'NonCompliant':
                comparison['summary']['currently_noncompliant'] += 1
            
            comparison['policies'][policy_name] = policy_comparison

        comparison['summary']['clusters_with_changes'] = list(comparison['summary']['clusters_with_changes'])
        comparison['summary']['all_managed_clusters'] = list(comparison['summary']['all_managed_clusters'])
        
        return comparison

    def _get_all_clusters(self, states: List[Dict]) -> Set[str]:
        clusters = set()
        for state in states:
            for policy in state.values():
                if 'cluster_status' in policy:
                    clusters.update(policy['cluster_status'].keys())
        return clusters

    def _compare_policy_states(self, pre: Dict, post: Dict, policy_name: str) -> Dict:
        if pre is None:
            pre = {}
        if post is None:
            post = {}
            
        pre_compliance = pre.get('overall_compliance', 'Unknown')
        post_compliance = post.get('overall_compliance', 'Unknown')
            
        comparison = {
            'changed': False,
            'pre_compliance': pre_compliance,
            'post_compliance': post_compliance,
            'cluster_changes': [],
            'noncompliant_clusters': [],
            'compliance_category': self._determine_compliance_category(
                pre_compliance,
                post_compliance
            ),
            'details': post.get('details', pre.get('details', {
                'name': policy_name,
                'remediation_action': 'inform',
                'description': ''
            }))
        }

        pre_clusters = pre.get('cluster_status', {})
        post_clusters = post.get('cluster_status', {})
        
        for cluster in set(pre_clusters.keys()) | set(post_clusters.keys()):
            pre_status = pre_clusters.get(cluster, {})
            post_status = post_clusters.get(cluster, {})
            
            if pre_status.get('compliant') != post_status.get('compliant'):
                comparison['cluster_changes'].append({
                    'cluster': cluster,
                    'pre_status': pre_status.get('compliant', 'Unknown'),
                    'post_status': post_status.get('compliant', 'Unknown')
                })

            if post_status.get('compliant') == 'NonCompliant':
                comparison['noncompliant_clusters'].append({
                    'cluster': cluster,
                    'console_url': post_status.get('console_url')
                })

        comparison['changed'] = bool(comparison['cluster_changes'])
        return comparison

    def _determine_compliance_category(self, pre_compliance: str, post_compliance: str) -> str:
        """Determine the compliance category based on pre and post states"""
        if pre_compliance in ['Compliant', 'Unknown'] and post_compliance == 'NonCompliant':
            return 'newly_noncompliant'
        elif pre_compliance == 'NonCompliant' and post_compliance == 'NonCompliant':
            return 'still_noncompliant'
        return 'compliant'

class MultiHubPolicyValidator:
    def __init__(self, hub_configs: List[Dict[str, str]]):
        self.logger = self._setup_logger()
        self.hub_validators = {}
        for hub_config in hub_configs:
            try:
                validator = SingleHubValidator(
                    kubeconfig=hub_config['kubeconfig'],
                    hub_name=hub_config['name']
                )
                self.hub_validators[hub_config['name']] = validator
            except Exception as e:
                self.logger.error(f"Failed to initialize hub {hub_config['name']}: {e}")

    def _setup_logger(self) -> logging.Logger:
        logger = logging.getLogger('MultiHubPolicyValidator')
        logger.setLevel(logging.DEBUG)
        if not logger.handlers:
            handler = logging.StreamHandler()
            handler.setFormatter(logging.Formatter('%(asctime)s - %(levelname)s - %(message)s'))
            logger.addHandler(handler)
        return logger

    def get_all_hub_compliance(self, namespace: str) -> Dict:
        hub_data = {}
        with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
            future_to_hub = {
                executor.submit(validator.get_cluster_compliance, namespace): hub_name
                for hub_name, validator in self.hub_validators.items()
            }
            for future in concurrent.futures.as_completed(future_to_hub):
                hub_name = future_to_hub[future]
                try:
                    hub_data[hub_name] = future.result()
                except Exception as e:
                    self.logger.error(f"Error getting compliance for hub {hub_name}: {e}")
                    hub_data[hub_name] = {"error": str(e)}
        
        return hub_data

    def compare_multi_hub_states(self, pre_states: Dict, post_states: Dict) -> Dict:
        comparison = {
            'hubs': {},
            'summary': {
                'total_hubs': len(pre_states),
                'total_policies': 0,
                'policies_changed': 0,
                'currently_noncompliant': 0,
                'all_managed_clusters': set()
            }
        }

        for hub_name in pre_states.keys():
            if hub_name not in post_states:
                continue

            hub_validator = self.hub_validators.get(hub_name)
            if not hub_validator:
                continue

            hub_comparison = hub_validator.compare_states(
                pre_states[hub_name],
                post_states[hub_name]
            )

            comparison['hubs'][hub_name] = hub_comparison
            comparison['summary']['total_policies'] += hub_comparison['summary']['total_policies']
            comparison['summary']['policies_changed'] += hub_comparison['summary']['policies_changed']
            comparison['summary']['currently_noncompliant'] += hub_comparison['summary']['currently_noncompliant']
            comparison['summary']['all_managed_clusters'].update(hub_comparison['summary']['all_managed_clusters'])

        comparison['summary']['all_managed_clusters'] = list(comparison['summary']['all_managed_clusters'])
        return comparison

def main():
    module = AnsibleModule(
        argument_spec=dict(
            hub_configs=dict(type='list', required=True, elements='dict'),
            namespace=dict(type='str', required=True),
            state=dict(type='str', required=True, choices=['pre', 'post']),
            state_file=dict(type='str', required=True),
            wait_time=dict(type='int', default=300)
        ),
        supports_check_mode=True
    )

    if module.check_mode:
        module.exit_json(changed=False)

    try:
        validator = MultiHubPolicyValidator(module.params['hub_configs'])
        result = {'changed': False, 'compliance_state': {}, 'comparison': None}
        
        if module.params['state'] == 'pre':
            result['compliance_state'] = validator.get_all_hub_compliance(module.params['namespace'])
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
            
            post_state = validator.get_all_hub_compliance(module.params['namespace'])
            comparison = validator.compare_multi_hub_states(pre_state, post_state)
            
            result.update({
                'compliance_state': post_state,
                'comparison': comparison,
                'changed': comparison['summary']['policies_changed'] > 0
            })

        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=str(e))

if __name__ == '__main__':
    main()
