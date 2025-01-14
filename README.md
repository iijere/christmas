#!/usr/bin/env python3

from ansible.module_utils.basic import AnsibleModule
from datetime import datetime
import kubernetes.client as k8s
from kubernetes import config
import logging
import json
import time
import sys
import os

ANSIBLE_METADATA = {
    'metadata_version': '1.1',
    'status': ['preview'],
    'supported_by': 'community'
}

DOCUMENTATION = r'''
---
module: rhacm_policy_validator
short_description: Pre/Post deployment RHACM policy compliance validator
description:
    - Captures pre-deployment policy compliance state
    - Validates post-deployment compliance status
    - Identifies non-compliant clusters
    - Generates detailed compliance change report
options:
    kubeconfig:
        description: Path to kubeconfig file
        required: true
        type: str
    namespace:
        description: Namespace containing the policies
        required: true
        type: str
    state:
        description: Whether this is pre or post deployment check
        required: true
        type: str
        choices: ['pre', 'post']
    state_file:
        description: Path to store/read pre-deployment state
        required: true
        type: str
    wait_time:
        description: Time to wait for policy evaluation in post state (seconds)
        required: false
        type: int
        default: 300
'''

class ComplianceState:
    """Class to handle policy compliance state storage and comparison"""
    def __init__(self, state_file):
        self.state_file = state_file
        
    def save_state(self, compliance_data):
        """Save pre-deployment compliance state"""
        with open(self.state_file, 'w') as f:
            json.dump(compliance_data, f)
            
    def load_state(self):
        """Load pre-deployment compliance state"""
        try:
            with open(self.state_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            return None

class RHACMPolicyValidator:
    def __init__(self, kubeconfig_path):
        """Initialize validator with kubeconfig"""
        self.logger = self._setup_logging()
        try:
            config.load_kube_config(kubeconfig_path)
            self.custom_api = k8s.CustomObjectsApi()
            self.core_api = k8s.CoreV1Api()
        except Exception as e:
            self.logger.error(f"Failed to initialize: {str(e)}")
            raise

    def _setup_logging(self):
        """Configure logging with structured format"""
        logger = logging.getLogger('RHACMPolicyValidator')
        logger.setLevel(logging.INFO)
        if not logger.handlers:
            handler = logging.StreamHandler()
            formatter = logging.Formatter(
                '%(asctime)s - %(levelname)s - %(message)s'
            )
            handler.setFormatter(formatter)
            logger.addHandler(handler)
        return logger

    def get_managed_clusters(self):
        """Get list of all managed clusters"""
        try:
            clusters = self.custom_api.list_cluster_custom_object(
                group="cluster.open-cluster-management.io",
                version="v1",
                plural="managedclusters"
            )
            return [c['metadata']['name'] for c in clusters['items']]
        except Exception as e:
            self.logger.error(f"Error getting managed clusters: {str(e)}")
            raise

    def get_policy_compliance(self, namespace):
        """Get compliance status for all policies in namespace"""
        try:
            policies = self.custom_api.list_namespaced_custom_object(
                group="policy.open-cluster-management.io",
                version="v1",
                namespace=namespace,
                plural="policies"
            )
            
            # Ensure we're working with a dictionary
            if isinstance(policies, str):
                policies = json.loads(policies)
            
            compliance_data = {}
            for policy in policies.get('items', []):
                policy_name = policy.get('metadata', {}).get('name')
                if not policy_name:
                    continue
                    
                status = policy.get('status', {})
                if isinstance(status, str):
                    status = json.loads(status)
                
                # Get per-cluster compliance details
                cluster_status = {}
                cluster_statuses = status.get('status', [])
                if isinstance(cluster_statuses, str):
                    cluster_statuses = json.loads(cluster_statuses)
                
                for cluster_info in cluster_statuses:
                    if isinstance(cluster_info, str):
                        cluster_info = json.loads(cluster_info)
                    
                    cluster_name = cluster_info.get('clustername')
                    if cluster_name:
                        cluster_status[cluster_name] = {
                            'compliant': cluster_info.get('compliant', 'Unknown'),
                            'message': cluster_info.get('message', ''),
                            'reason': cluster_info.get('reason', '')
                        }
                
                policy_details = self._get_policy_details(policy)
                compliance_data[policy_name] = {
                    'overall_compliance': status.get('compliant', 'Unknown'),
                    'cluster_status': cluster_status,
                    'timestamp': datetime.now().isoformat(),
                    'details': policy_details
                }
                
            self.logger.info(f"Found {len(compliance_data)} policies in namespace {namespace}")
            return compliance_data
        except Exception as e:
            self.logger.error(f"Error getting policy compliance: {str(e)}")
            raise

    def _get_policy_details(self, policy):
        """Extract policy metadata and specifications"""
        try:
            if isinstance(policy, str):
                policy = json.loads(policy)
                
            metadata = policy.get('metadata', {})
            if isinstance(metadata, str):
                metadata = json.loads(metadata)
                
            spec = policy.get('spec', {})
            if isinstance(spec, str):
                spec = json.loads(spec)
                
            # Get remediation action with proper default
            remediation_action = spec.get('remediationAction', 'inform').lower()
            
            return {
                'name': metadata.get('name', 'unknown'),
                'description': spec.get('description', ''),
                'remediation_action': remediation_action,
                'annotations': metadata.get('annotations', {})
            }
        except Exception as e:
            self.logger.error(f"Error extracting policy details: {str(e)}")
            return {
                'name': 'unknown',
                'description': '',
                'remediation_action': '',
                'severity': 'unknown',
                'categories': [],
                'standards': []
            }

    def compare_states(self, pre_state, post_state):
        """Compare pre and post deployment states"""
        comparison = {
            'policies': {},
            'summary': {
                'total_policies': len(post_state),
                'policies_changed': 0,
                'currently_noncompliant': 0,
                'clusters_with_changes': set(),
                'all_managed_clusters': set()  # Track all clusters
            }
        }

        all_policies = set(pre_state.keys()) | set(post_state.keys())
        
        for policy_name in all_policies:
            pre = pre_state.get(policy_name, {})
            post = post_state.get(policy_name, {})
            
            # Collect all clusters from both pre and post states
            all_clusters_pre = set(pre.get('cluster_status', {}).keys())
            all_clusters_post = set(post.get('cluster_status', {}).keys())
            comparison['summary']['all_managed_clusters'].update(all_clusters_pre)
            comparison['summary']['all_managed_clusters'].update(all_clusters_post)
            
            # Ensure we have a details structure
            details = post.get('details', pre.get('details', {
                'name': policy_name,
                'description': '',
                'remediation_action': '',
                'severity': 'unknown',
                'categories': [],
                'standards': []
            }))
            
            policy_comparison = {
                'changed': False,
                'pre_compliance': pre.get('overall_compliance', 'Unknown'),
                'post_compliance': post.get('overall_compliance', 'Unknown'),
                'cluster_changes': [],
                'noncompliant_clusters': [],
                'details': details
            }

            # Compare cluster-level compliance
            all_clusters = set(pre.get('cluster_status', {}).keys()) | \
                         set(post.get('cluster_status', {}).keys())
                         
            for cluster in all_clusters:
                pre_cluster = pre.get('cluster_status', {}).get(cluster, {})
                post_cluster = post.get('cluster_status', {}).get(cluster, {})
                
                if pre_cluster.get('compliant') != post_cluster.get('compliant'):
                    policy_comparison['cluster_changes'].append({
                        'cluster': cluster,
                        'pre_status': pre_cluster.get('compliant', 'Unknown'),
                        'post_status': post_cluster.get('compliant', 'Unknown'),
                        'message': post_cluster.get('message', '')
                    })
                    comparison['summary']['clusters_with_changes'].add(cluster)
                
                # Track currently non-compliant clusters
                if post_cluster.get('compliant') == 'NonCompliant':
                    policy_comparison['noncompliant_clusters'].append({
                        'cluster': cluster,
                        'reason': post_cluster.get('reason', ''),
                        'message': post_cluster.get('message', '')
                    })

            policy_comparison['changed'] = bool(policy_comparison['cluster_changes'])
            if policy_comparison['changed']:
                comparison['summary']['policies_changed'] += 1
                
            if policy_comparison['post_compliance'] == 'NonCompliant':
                comparison['summary']['currently_noncompliant'] += 1
                
            comparison['policies'][policy_name] = policy_comparison

        comparison['summary']['clusters_with_changes'] = \
            list(comparison['summary']['clusters_with_changes'])
            
        return comparison

def run_module():
    module_args = dict(
        kubeconfig=dict(type='str', required=True),
        namespace=dict(type='str', required=True),
        state=dict(type='str', required=True, choices=['pre', 'post']),
        state_file=dict(type='str', required=True),
        wait_time=dict(type='int', required=False, default=300)
    )

    result = dict(
        changed=False,
        compliance_state={},
        comparison=None,
        debug_info={}
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    if module.check_mode:
        module.exit_json(**result)

    try:
        validator = RHACMPolicyValidator(module.params['kubeconfig'])
        state_handler = ComplianceState(module.params['state_file'])
        
        if module.params['state'] == 'pre':
            # Pre-deployment state capture
            compliance_state = validator.get_policy_compliance(
                module.params['namespace']
            )
            state_handler.save_state(compliance_state)
            result['compliance_state'] = compliance_state
            
        else:
            # Post-deployment validation
            if module.params['wait_time'] > 0:
                time.sleep(module.params['wait_time'])
                
            pre_state = state_handler.load_state()
            if not pre_state:
                module.fail_json(
                    msg="No pre-deployment state found. Run pre-deployment check first.",
                    **result
                )
                
            post_state = validator.get_policy_compliance(
                module.params['namespace']
            )
            
            comparison = validator.compare_states(pre_state, post_state)
            result['compliance_state'] = post_state
            result['comparison'] = comparison
            result['changed'] = comparison['summary']['policies_changed'] > 0

    except Exception as e:
        module.fail_json(msg=str(e), **result)

    module.exit_json(**result)

def main():
    run_module()

if __name__ == '__main__':
    main()
