def get_policy_url(self, policy_name: str, namespace: str) -> Optional[str]:
    """Get the policy console URL including namespace"""
    host = self.get_console_host()
    if host:
        return f"https://{host}/multicloud/governance/policies/details/{namespace}/{policy_name}/results"
    return None


def _process_cluster_statuses(self, policy_name: str, status: Dict, namespace: str) -> Dict:
    cluster_status = {}
    console_url = self.url_builder.get_policy_url(policy_name, namespace)
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
            cluster_statuses = self._process_cluster_statuses(policy_name, status, namespace)  # Pass namespace here
            
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
