<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RHACM Policy Deployment Validation Report</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/3.7.0/chart.min.js"></script>
    <style>
        .cluster-group {
            max-height: 300px;
            overflow-y: auto;
        }
        .search-highlight {
            background-color: yellow;
        }
    </style>
</head>
<body class="bg-gray-50">
    <!-- Fixed Header -->
    <nav class="bg-gray-800 shadow fixed w-full z-10">
        <div class="max-w-7xl mx-auto px-4">
            <div class="flex justify-between h-16">
                <div class="flex items-center">
                    <h1 class="text-xl font-bold text-white">RHACM Policy Report</h1>
                </div>
                <div class="flex items-center space-x-4">
                    <span class="text-white text-sm">
                        Total Clusters: <span class="font-bold">{{ comparison.summary.all_managed_clusters|length }}</span>
                    </span>
                    <span class="text-red-400 text-sm">
                        Non-Compliant: <span class="font-bold">{{ comparison.summary.currently_noncompliant }}</span>
                    </span>
                    <span class="text-gray-300 text-sm">{{ date_time }}</span>
                </div>
            </div>
        </div>
    </nav>

    <!-- Main Content -->
    <div class="pt-16">
        <!-- Quick Stats Cards -->
        <div class="max-w-7xl mx-auto px-4 py-6">
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
                <div class="bg-white rounded-lg shadow p-6">
                    <div class="flex items-center justify-between">
                        <div>
                            <p class="text-sm font-medium text-gray-600">Total Policies</p>
                            <p class="text-2xl font-semibold text-gray-900">{{ comparison.summary.total_policies }}</p>
                        </div>
                        <div class="p-3 bg-blue-100 rounded-full">
                            <svg class="w-6 h-6 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2"/>
                            </svg>
                        </div>
                    </div>
                </div>

                <div class="bg-white rounded-lg shadow p-6">
                    <div class="flex items-center justify-between">
                        <div>
                            <p class="text-sm font-medium text-gray-600">Compliant</p>
                            <p class="text-2xl font-semibold text-green-600">
                                {{ comparison.summary.total_policies - comparison.summary.currently_noncompliant }}
                            </p>
                        </div>
                        <div class="p-3 bg-green-100 rounded-full">
                            <svg class="w-6 h-6 text-green-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/>
                            </svg>
                        </div>
                    </div>
                </div>

                <div class="bg-white rounded-lg shadow p-6">
                    <div class="flex items-center justify-between">
                        <div>
                            <p class="text-sm font-medium text-gray-600">Non-Compliant</p>
                            <p class="text-2xl font-semibold text-red-600">{{ comparison.summary.currently_noncompliant }}</p>
                        </div>
                        <div class="p-3 bg-red-100 rounded-full">
                            <svg class="w-6 h-6 text-red-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
                            </svg>
                        </div>
                    </div>
                </div>

                <div class="bg-white rounded-lg shadow p-6">
                    <div class="flex items-center justify-between">
                        <div>
                            <p class="text-sm font-medium text-gray-600">Changed</p>
                            <p class="text-2xl font-semibold text-yellow-600">{{ comparison.summary.policies_changed }}</p>
                        </div>
                        <div class="p-3 bg-yellow-100 rounded-full">
                            <svg class="w-6 h-6 text-yellow-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/>
                            </svg>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- Non-Compliant Policies Section -->
        <div class="max-w-7xl mx-auto px-4 py-6">
            <div class="bg-white rounded-lg shadow">
                <div class="px-4 py-5 border-b border-gray-200">
                    <div class="flex justify-between items-center">
                        <h2 class="text-lg font-medium text-gray-900">Non-Compliant Policies</h2>
                        <input type="text" 
                               id="policySearchInput" 
                               placeholder="Search policies..." 
                               class="px-4 py-2 border rounded-lg w-64">
                    </div>
                </div>
                <div class="divide-y divide-gray-200">
                    {% for policy_name, policy in comparison.policies.items() %}
                        {% if policy.noncompliant_clusters %}
                            <div class="policy-item p-4" data-policy-name="{{ policy_name }}">
                                <div class="flex justify-between items-start">
                                    <div>
                                        <h3 class="text-lg font-medium text-gray-900">{{ policy_name }}</h3>
                                        <p class="mt-1 text-sm text-gray-500">{{ policy.details.description }}</p>
                                    </div>
                                    <span class="px-2 py-1 text-xs font-semibold rounded-full
                                        {% if policy.details.remediation_action == 'enforce' %}
                                            bg-purple-100 text-purple-800
                                        {% else %}
                                            bg-blue-100 text-blue-800
                                        {% endif %}">
                                        {{ policy.details.remediation_action|title }}
                                    </span>
                                </div>
                                <div class="mt-4">
                                    <button class="text-sm text-blue-600 hover:text-blue-800"
                                            onclick="toggleClusterList('{{ policy_name }}')">
                                        Show {{ policy.noncompliant_clusters|length }} non-compliant clusters
                                    </button>
                                    <div id="clusters-{{ policy_name }}" class="hidden mt-2">
                                        <div class="cluster-group bg-gray-50 rounded-lg p-4 space-y-2">
                                            {% for cluster in policy.noncompliant_clusters %}
                                                <div class="bg-white p-3 rounded shadow-sm">
                                                    <div class="font-medium">{{ cluster.cluster }}</div>
                                                    <div class="text-sm text-gray-600">{{ cluster.message }}</div>
                                                </div>
                                            {% endfor %}
                                        </div>
                                    </div>
                                </div>
                            </div>
                        {% endif %}
                    {% endfor %}
                </div>
            </div>
        </div>

        <!-- Changed Policies Section -->
        <div class="max-w-7xl mx-auto px-4 py-6">
            <div class="bg-white rounded-lg shadow">
                <div class="px-4 py-5 border-b border-gray-200">
                    <div class="flex justify-between items-center">
                        <h2 class="text-lg font-medium text-gray-900">Changed Policies</h2>
                        <input type="text" 
                               id="changeSearchInput" 
                               placeholder="Search changes..." 
                               class="px-4 py-2 border rounded-lg w-64">
                    </div>
                </div>
                <div class="divide-y divide-gray-200">
                    {% for policy_name, policy in comparison.policies.items() %}
                        {% if policy.changed %}
                            <div class="change-item p-4" data-policy-name="{{ policy_name }}">
                                <div class="flex justify-between items-start">
                                    <div>
                                        <h3 class="text-lg font-medium text-gray-900">{{ policy_name }}</h3>
                                        <div class="mt-2 flex items-center space-x-2">
                                            <span class="px-2 py-1 text-xs font-semibold rounded-full
                                                {{ 'bg-green-100 text-green-800' if policy.pre_compliance == 'Compliant' else 'bg-red-100 text-red-800' }}">
                                                {{ policy.pre_compliance }}
                                            </span>
                                            <span class="text-gray-500">→</span>
                                            <span class="px-2 py-1 text-xs font-semibold rounded-full
                                                {{ 'bg-green-100 text-green-800' if policy.post_compliance == 'Compliant' else 'bg-red-100 text-red-800' }}">
                                                {{ policy.post_compliance }}
                                            </span>
                                        </div>
                                    </div>
                                </div>
                                <div class="mt-4">
                                    <button class="text-sm text-blue-600 hover:text-blue-800"
                                            onclick="toggleChangeList('{{ policy_name }}')">
                                        Show {{ policy.cluster_changes|length }} affected clusters
                                    </button>
                                    <div id="changes-{{ policy_name }}" class="hidden mt-2">
                                        <div class="cluster-group bg-gray-50 rounded-lg p-4 space-y-2">
                                            {% for change in policy.cluster_changes %}
                                                <div class="bg-white p-3 rounded shadow-sm">
                                                    <div class="font-medium">{{ change.cluster }}</div>
                                                    <div class="flex items-center space-x-2 mt-1">
                                                        <span class="text-sm {{ 'text-green-600' if change.pre_status == 'Compliant' else 'text-red-600' }}">
                                                            {{ change.pre_status }}
                                                        </span>
                                                        <span class="text-gray-500">→</span>
                                                        <span class="text-sm {{ 'text-green-600' if change.post_status == 'Compliant' else 'text-red-600' }}">
                                                            {{ change.post_status }}
                                                        </span>
                                                    </div>
                                                </div>
                                            {% endfor %}
                                        </div>
                                    </div>
                                </div>
                            </div>
                        {% endif %}
                    {% endfor %}
                </div>
            </div>
        </div>
    </div>

    <script>
        // Toggle cluster list visibility
        function toggleClusterList(policyName) {
            const clusterList = document.getElementById(`clusters-${policyName}`);
            clusterList.classList.toggle('hidden');
        }

        // Toggle change list visibility
        function toggleChangeList(policyName) {
            const changeList = document.getElementById(`changes-${policyName}`);
            changeList.classList.toggle('hidden');
        }

        // Policy search functionality
        document.getElementById('policySearchInput').addEventListener('input', function(e) {
            const searchText = e.target.value.toLowerCase();
            document.querySelectorAll('.policy-item').forEach(item => {
                const policyName = item.dataset.policyName.toLowerCase();
                item.style.display = policyName.includes(searchText) ? 'block' : 'none';
            });
        });

        // Change search functionality
        document.getElementById('changeSearchInput').addEventListener('input', function(e) {
            const searchText = e.target.value.toLowerCase();
            document.querySelectorAll('.change-item').forEach(item => {
                const policyName = item.dataset.policyName.toLowerCase();
                item.style.display = policyName.includes(searchText) ? 'block' : 'none';
            });
        });
    </script>
</body>
</html>
