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
