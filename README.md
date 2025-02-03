<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Multi-Hub RHACM Policy Validation Report</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/3.7.0/chart.min.js"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        ms: {
                            blue: '#004B87',
                            lightblue: '#E5EEF4',
                            gray: '#58595B',
                            lightgray: '#F4F4F4'
                        }
                    }
                }
            }
        }
    </script>
    <style>
        .cluster-group {
            max-height: 300px;
            overflow-y: auto;
        }
        .policy-card {
            transition: all 0.2s ease-in-out;
        }
        .policy-card:hover {
            box-shadow: 0 4px 6px -1px rgba(0, 75, 135, 0.1), 0 2px 4px -1px rgba(0, 75, 135, 0.06);
            transform: translateY(-2px);
        }
    </style>
</head>
<body class="bg-ms-lightgray">
    <!-- Fixed Header -->
    <nav class="bg-ms-blue shadow fixed w-full z-10">
        <div class="max-w-7xl mx-auto px-4">
            <div class="flex justify-between h-16">
                <div class="flex items-center">
                    <h1 class="text-xl font-bold text-white">RHACM Policy Report</h1>
                </div>
                <div class="flex items-center space-x-6">
                    <div class="px-3 py-2 rounded-lg bg-white/10">
                        <span class="text-white text-sm">
                            Prime Clusters: <span class="font-bold">{{ comparison.summary.total_hubs }}</span>
                        </span>
                    </div>
                    <div class="px-3 py-2 rounded-lg bg-white/10">
                        <span class="text-white text-sm">
                            Total Policies: <span class="font-bold">{{ comparison.summary.total_policies }}</span>
                        </span>
                    </div>
                    <div class="px-3 py-2 rounded-lg bg-red-500/20">
                        <span class="text-white text-sm">
                            Non-Compliant: <span class="font-bold">{{ comparison.summary.currently_noncompliant }}</span>
                        </span>
                    </div>
                    <span class="text-gray-300 text-sm">{{ date_time }}</span>
                </div>
            </div>
        </div>
    </nav>

    <!-- Main Content -->
    <div class="pt-20">
        <!-- Summary Dashboard -->
        <div class="max-w-7xl mx-auto px-4 mb-8">
            <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
                <!-- Compliance Overview Chart -->
                <div class="bg-white rounded-lg shadow p-6">
                    <h3 class="text-lg font-medium text-ms-blue mb-4">Policy Compliance Overview</h3>
                    <canvas id="complianceChart" height="200"></canvas>
                </div>
                
                <!-- Policy Changes Chart -->
                <div class="bg-white rounded-lg shadow p-6">
                    <h3 class="text-lg font-medium text-ms-blue mb-4">Policy Changes by Prime Cluster</h3>
                    <canvas id="changesChart" height="200"></canvas>
                </div>
            </div>
        </div>

        <!-- Hub Navigation -->
        <div class="max-w-7xl mx-auto px-4 mb-8">
            <div class="bg-white rounded-lg shadow">
                <div class="px-6 py-4 border-b border-gray-200">
                    <h2 class="text-lg font-medium text-ms-blue">Prime Clusters Assessed</h2>
                </div>
                <div class="p-6">
                    <div class="flex flex-wrap gap-4">
                        {% for hub_name, hub_data in comparison.hubs.items() %}
                        <button
                            onclick="showHub('{{ hub_name }}')"
                            class="hub-button px-4 py-2 rounded-lg text-sm font-medium transition-colors
                                   {% if hub_data.summary.currently_noncompliant > 0 %}
                                   bg-red-50 text-red-800 hover:bg-red-100
                                   {% else %}
                                   bg-ms-lightblue text-ms-blue hover:bg-opacity-75
                                   {% endif %}">
                            {{ hub_name }}
                            {% if hub_data.summary.currently_noncompliant > 0 %}
                            ({{ hub_data.summary.currently_noncompliant }} issues)
                            {% endif %}
                        </button>
                        {% endfor %}
                    </div>
                </div>
            </div>
        </div>

        <!-- Hub Details -->
        {% for hub_name, hub_data in comparison.hubs.items() %}
        <div id="hub-{{ hub_name }}" class="hub-content hidden">
            <div class="max-w-7xl mx-auto px-4">
                <!-- Hub Summary Stats -->
                <div class="grid grid-cols-1 md:grid-cols-4 gap-4 mb-8">
                    <div class="bg-white rounded-lg shadow p-6">
                        <p class="text-sm text-ms-gray">Total Policies</p>
                        <p class="text-2xl font-bold text-ms-blue">{{ hub_data.summary.total_policies }}</p>
                    </div>
                    <div class="bg-white rounded-lg shadow p-6">
                        <p class="text-sm text-ms-gray">Compliant</p>
                        <p class="text-2xl font-bold text-green-600">
                            {{ hub_data.summary.total_policies - hub_data.summary.currently_noncompliant }}
                        </p>
                    </div>
                    <div class="bg-white rounded-lg shadow p-6">
                        <p class="text-sm text-ms-gray">Non-Compliant</p>
                        <p class="text-2xl font-bold text-red-600">{{ hub_data.summary.currently_noncompliant }}</p>
                    </div>
                    <div class="bg-white rounded-lg shadow p-6">
                        <p class="text-sm text-ms-gray">Changed</p>
                        <p class="text-2xl font-bold text-yellow-600">{{ hub_data.summary.policies_changed }}</p>
                    </div>
                </div>



                <!-- Policy Categories -->
                <!-- Newly Non-Compliant Policies Section -->
                <div class="mb-8">
                    <div class="bg-white rounded-lg shadow">
                        <div class="px-6 py-4 border-b border-gray-200">
                            <h3 class="text-lg font-medium text-red-800 flex items-center">
                                <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/>
                                </svg>
                                Newly Non-Compliant Policies
                            </h3>
                        </div>
                        <div class="p-6">
                            {% set has_new_violations = false %}
                            {% for policy_name, policy in hub_data.policies.items() %}
                                {% if policy.changed and policy.pre_compliance != 'NonCompliant' and policy.post_compliance == 'NonCompliant' %}
                                    {% set has_new_violations = true %}
                                    <div class="policy-card bg-red-50 rounded-lg p-6 mb-4 last:mb-0">
                                        <div class="flex justify-between items-start">
                                            <div>
                                                <h4 class="text-lg font-medium text-red-900">{{ policy_name }}</h4>
                                                {% if policy.details.description %}
                                                    <p class="text-sm text-red-800 mt-1">{{ policy.details.description }}</p>
                                                {% endif %}
                                            </div>
                                            <span class="px-3 py-1 text-xs font-semibold rounded-full {{ 'bg-purple-100 text-purple-800' if policy.details.remediation_action == 'enforce' else 'bg-blue-100 text-blue-800' }}">
                                                {{ policy.details.remediation_action|title }}
                                            </span>
                                        </div>
                                        {% if policy.noncompliant_clusters %}
                                            <div class="mt-4">
                                                <h5 class="text-sm font-medium text-red-900 mb-2">Affected Clusters:</h5>
                                                <div class="space-y-3">
                                                    {% for cluster in policy.noncompliant_clusters %}
                                                        <div class="bg-white/50 rounded-lg p-4">
                                                            <div class="font-medium text-red-900">{{ cluster.cluster }}</div>
                                                            {% if cluster.console_url %}
                                                                <div class="mt-2">
                                                                    <a href="{{ cluster.console_url }}" target="_blank" rel="noopener noreferrer" 
                                                                    class="inline-flex items-center text-red-800 hover:text-red-600">
                                                                        <svg class="w-4 h-4 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                                                                                d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"/>
                                                                        </svg>
                                                                        View Violation Message in Console
                                                                    </a>
                                                                </div>
                                                            {% endif %}
                                                        </div>
                                                    {% endfor %}
                                                </div>
                                            </div>
                                        {% endif %}
                                    </div>
                                {% endif %}
                            {% endfor %}
                            {% if not has_new_violations %}
                                <div class="text-ms-gray italic">These policies were compliant prior, but became non-compliant after deployment.</div>
                            {% endif %}
                        </div>
                    </div>
                </div>

                <!-- Non-Compliant Policies by Cluster Section -->
                <div class="bg-white rounded-lg shadow mb-8">
                    <div class="px-6 py-4 border-b border-gray-200">
                        <h3 class="text-lg font-medium text-ms-blue flex items-center">
                            <svg class="w-5 h-5 mr-2 text-red-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 21V5a2 2 0 00-2-2H7a2 2 0 00-2 2v16m14 0h2m-2 0h-5m-9 0H3m2 0h5M9 7h1m-1 4h1m4-4h1m-1 4h1m-5 10v-5a1 1 0 011-1h2a1 1 0 011 1v5m-4 0h4"/>
                            </svg>
                            Non-Compliant Policies
                        </h3>
                    </div>
                    <div class="p-6">
                        {% set policies_with_violations = {} %}
                        {% for policy_name, policy in hub_data.policies.items() %}
                            {% if policy.noncompliant_clusters %}
                                {% set _ = policies_with_violations.update({
                                    policy_name: {
                                        'clusters': policy.noncompliant_clusters,
                                        'details': policy.details,
                                        'compliance_category': policy.compliance_category
                                    }
                                }) %}
                            {% endif %}
                        {% endfor %}

                        {% if policies_with_violations %}
                            <div class="space-y-6">
                                {% for policy_name, policy_data in policies_with_violations.items() %}
                                    <div class="policy-card bg-ms-lightblue rounded-lg p-6">
                                        <div class="flex justify-between items-start mb-4">
                                            <div>
                                                <h4 class="text-lg font-medium text-ms-blue">{{ policy_name }}</h4>
                                                {% if policy_data.details.description %}
                                                    <p class="text-sm text-ms-gray mt-1">{{ policy_data.details.description }}</p>
                                                {% endif %}
                                            </div>
                                            <div class="flex items-center space-x-2">
                                                <span class="px-2 py-1 text-xs font-semibold rounded-full
                                                    {{ 'bg-purple-100 text-purple-800' if policy_data.details.remediation_action == 'enforce' else 'bg-blue-100 text-blue-800' }}">
                                                    {{ policy_data.details.remediation_action|title }}
                                                </span>
                                                <span class="px-2 py-1 text-xs font-semibold rounded-full
                                                    {{ 'bg-red-100 text-red-800' if policy_data.compliance_category == 'newly_noncompliant' else 'bg-orange-100 text-orange-800' }}">
                                                    {{ 'Newly Non-Compliant' if policy_data.compliance_category == 'newly_noncompliant' else 'Still Non-Compliant' }}
                                                </span>
                                            </div>
                                        </div>
                                        <div class="space-y-4">
                                            <h5 class="font-medium text-ms-blue">Affected Clusters:</h5>
                                            {% for cluster in policy_data.clusters %}
                                                <div class="bg-white rounded-lg p-4 shadow-sm">
                                                    <div class="font-medium text-ms-blue">{{ cluster.cluster }}</div>
                                                    {% if cluster.console_url %}
                                                        <div class="mt-2">
                                                            <a href="{{ cluster.console_url }}" target="_blank" rel="noopener noreferrer" 
                                                            class="inline-flex items-center text-ms-blue hover:text-ms-blue/80">
                                                                <svg class="w-4 h-4 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                                                                        d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"/>
                                                                </svg>
                                                                View Violation Message in Console
                                                            </a>
                                                        </div>
                                                    {% endif %}
                                                </div>
                                            {% endfor %}
                                        </div>
                                    </div>
                                {% endfor %}
                            </div>
                        {% else %}
                            <div class="text-ms-gray italic">No non-compliant policies found</div>
                        {% endif %}
                    </div>
                </div>

                <!-- Rest of the sections remain similar but with updated styling -->
            </div>
        </div>
        {% endfor %}
    </div>

    <script>
        function initializeCharts() {
            // Compliance Overview Chart
            const complianceCtx = document.getElementById('complianceChart').getContext('2d');
            new Chart(complianceCtx, {
                type: 'doughnut',
                data: {
                    labels: ['Compliant', 'Non-Compliant', 'Changed'],
                    datasets: [{
                        data: [
                            {{ comparison.summary.total_policies - comparison.summary.currently_noncompliant }},
                            {{ comparison.summary.currently_noncompliant }},
                            {{ comparison.summary.policies_changed }}
                        ],
                        backgroundColor: [
                            '#22C55E', // Green for compliant
                            '#EF4444', // Red for non-compliant
                            '#EAB308'  // Yellow for changed
                        ],
                        borderWidth: 0
                    }]
                },
                options: {
                    responsive: true,
                    plugins: {
                        legend: {
                            position: 'bottom'
                        }
                    },
                    cutout: '70%'
                }
            });

            // Policy Changes by Hub Chart
            const changesCtx = document.getElementById('changesChart').getContext('2d');
            new Chart(changesCtx, {
                type: 'bar',
                data: {
                    labels: [{% for hub_name in comparison.hubs.keys() %}'{{ hub_name }}',{% endfor %}],
                    datasets: [{
                        label: 'Non-Compliant Policies',
                        data: [{% for hub_data in comparison.hubs.values() %}{{ hub_data.summary.currently_noncompliant }},{% endfor %}],
                        backgroundColor: '#004B87'
                    }]
                },
                options: {
                    responsive: true,
                    scales: {
                        y: {
                            beginAtZero: true,
                            ticks: {
                                precision: 0
                            }
                        }
                    },
                    plugins: {
                        legend: {
                            display: false
                        }
                    }
                }
            });
        }

        function showHub(hubName) {
            document.querySelectorAll('.hub-content').forEach(el => el.classList.add('hidden'));
            document.getElementById(`hub-${hubName}`).classList.remove('hidden');
            
            document.querySelectorAll('.hub-button').forEach(btn => {
                btn.classList.remove('ring-2', 'ring-ms-blue', 'ring-offset-2');
            });
            event.target.closest('.hub-button').classList.add('ring-2', 'ring-ms-blue', 'ring-offset-2');
        }

        document.addEventListener('DOMContentLoaded', () => {
            initializeCharts();
            const firstHubButton = document.querySelector('.hub-button');
            if (firstHubButton) {
                firstHubButton.click();
            }
        });
    </script>
</body>
</html>
