<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RHACM Policy Report</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/3.9.1/chart.min.js"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        brand: {
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
        .policy-card {
            transition: transform 0.2s ease-in-out;
        }
        .policy-card:hover {
            transform: translateY(-2px);
        }
        .status-badge {
            @apply px-3 py-1 text-xs font-semibold rounded-full;
        }
        .status-badge.became-noncompliant {
            @apply bg-red-100 text-red-800;
        }
        .status-badge.became-compliant {
            @apply bg-green-100 text-green-800;
        }
    </style>
</head>
<body class="bg-brand-lightgray min-h-screen">
    
    <!-- Fixed Header -->
    <nav class="bg-brand-blue shadow fixed w-full z-10">
        <div class="max-w-7xl mx-auto px-4">
            <div class="flex justify-between h-16">
                <div class="flex items-center">
                    <h1 class="text-xl font-bold text-white">RHACM Policy Report</h1>
                </div>
                <div class="flex items-center space-x-4">
                    <div class="px-3 py-2 rounded-lg bg-white/10">
                        <span class="text-white text-sm">
                            Prime Clusters: <span class="font-bold">{{ comparison.summary.total_hubs|default(0) }}</span>
                        </span>
                    </div>
                    <div class="px-3 py-2 rounded-lg bg-red-500/20">
                        <span class="text-white text-sm">
                            Managed Clusters: <span class="font-bold">{{ comparison.summary.total_noncompliant_clusters|default(0) }}</span>
                        </span>
                    </div>
                    <span class="text-gray-300 text-sm">{{ date_time }}</span>
                </div>
            </div>
        </div>
    </nav>

    <!-- Main Content -->
    <div class="pt-20">
        <div class="max-w-7xl mx-auto px-4">

        <!-- Dashboard -->
        <div class="mb-8">
            <!-- Charts Grid -->
            <div class="grid grid-cols-1 md:grid-cols-2 gap-6 my-8">
                <!-- Donut Chart -->
                <div class="bg-white rounded-lg shadow">
                    <div class="p-4 border-b border-gray-200">
                        <h3 class="text-lg font-medium text-gray-900">Policy Compliance Overview</h3>
                    </div>
                    <div class="p-4">
                        <div class="relative" style="height: 280px;">
                            <canvas id="complianceChart"></canvas>
                        </div>
                    </div>
                </div>
                
                <!-- Bar Chart -->
                <div class="bg-white rounded-lg shadow">
                    <div class="p-4 border-b border-gray-200">
                        <h3 class="text-lg font-medium text-gray-900">Non-Compliant Policies by Prime Cluster</h3>
                    </div>
                    <div class="p-4">
                        <div class="relative" style="height: 280px;">
                            <canvas id="clusterChart"></canvas>
                        </div>
                    </div>
                </div>
            </div>
            
            <!-- Stats Cards -->
            <div class="grid grid-cols-2 md:grid-cols-4 gap-6">
                <div class="bg-white rounded-lg shadow p-4">
                    <div class="text-sm text-gray-600 mb-1">Total Policies</div>
                    <div class="text-2xl font-semibold text-brand-blue">{{ comparison.summary.total_policies|default(0) }}</div>
                </div>
                <div class="bg-white rounded-lg shadow p-4">
                    <div class="text-sm text-gray-600 mb-1">Compliant</div>
                    <div class="text-2xl font-semibold text-green-600">{{ comparison.summary.compliant|default(0) }}</div>
                </div>
                <div class="bg-white rounded-lg shadow p-4">
                    <div class="text-sm text-gray-600 mb-1">Non-Compliant</div>
                    <div class="text-2xl font-semibold text-red-600">{{ comparison.summary.total_noncompliant|default(0) }}</div>
                </div>
                <div class="bg-white rounded-lg shadow p-4">
                    <div class="text-sm text-gray-600 mb-1">Changed</div>
                    <div class="text-2xl font-semibold text-yellow-600">{{ comparison.summary.changed|default(0) }}</div>
                </div>
            </div>
        </div>

        <!-- Policy Changes Section -->
        <div class="bg-white rounded-lg shadow mb-8">
            <div class="px-6 py-4 border-b border-gray-200">
                <h3 class="text-lg font-medium text-gray-900 flex items-center">
                    <svg class="w-5 h-5 mr-2 text-yellow-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                            d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/>
                    </svg>
                    Policy Changes
                </h3>
            </div>

            <div class="p-6">
                {% if comparison.summary.changed > 0 %}
                    {% for policy_name, policy in comparison.consolidated_view.policies.items() %}
                        {% if policy.status_change in ['became_noncompliant', 'became_compliant'] %}
                            <div class="policy-card {% if policy.status_change == 'became_noncompliant' %}bg-red-50{% else %}bg-green-50{% endif %} rounded-lg p-6 mb-4 last:mb-0">
                                <div class="flex justify-between items-start">
                                    <div>
                                        <h4 class="text-lg font-medium {% if policy.status_change == 'became_noncompliant' %}text-red-900{% else %}text-green-900{% endif %}">
                                            {{ policy_name }}
                                        </h4>
                                        {% if policy.details.description %}
                                            <p class="text-sm {% if policy.status_change == 'became_noncompliant' %}text-red-800{% else %}text-green-800{% endif %} mt-2">
                                                {{ policy.details.description }}
                                            </p>
                                        {% endif %}
                                    </div>
                                    <div class="flex space-x-2">
                                        <span class="px-3 py-1 text-xs font-semibold rounded-full 
                                            {{ 'bg-purple-100 text-purple-800' if policy.details.remediation_action == 'enforce' 
                                            else 'bg-blue-100 text-blue-800' }}">
                                            {{ policy.details.remediation_action|title }}
                                        </span>
                                        <span class="status-badge {{ 'became-noncompliant' if policy.status_change == 'became_noncompliant' else 'became-compliant' }}">
                                            {% if policy.status_change == 'became_noncompliant' %}
                                                Became Non-Compliant
                                            {% else %}
                                                Became Compliant
                                            {% endif %}
                                        </span>
                                    </div>
                                </div>

                                <div class="mt-4 space-y-3">
                                    {% for hub_name, hub_data in policy.hubs.items() %}
                                        <div class="{% if policy.status_change == 'became_noncompliant' %}bg-white/50{% else %}bg-white/50{% endif %} rounded-lg p-4">
                                            <div class="font-medium {% if policy.status_change == 'became_noncompliant' %}text-red-900{% else %}text-green-900{% endif %} pb-2 border-b {% if policy.status_change == 'became_noncompliant' %}border-red-100{% else %}border-green-100{% endif %}">
                                                <div>
                                                    <span>Prime Cluster: {{ hub_name }}</span>
                                                    <p class="text-sm {% if policy.status_change == 'became_noncompliant' %}text-red-700{% else %}text-green-700{% endif %} mt-1">
                                                        Namespace: {{ hub_data.namespace }}
                                                    </p>
                                                </div>
                                            </div>
                                            {% if policy.status_change == 'became_noncompliant' and hub_data.noncompliant_clusters %}
                                                <div class="mt-2 space-y-2">
                                                    {% for cluster in hub_data.noncompliant_clusters %}
                                                        <div class="flex items-center justify-between py-2">
                                                            <span class="text-sm text-red-800">{{ cluster.cluster }}</span>
                                                            {% if cluster.console_url %}
                                                                <a href="{{ cluster.console_url }}" 
                                                                target="_blank"
                                                                class="text-sm text-red-800 hover:text-red-600 flex items-center">
                                                                    View in Console
                                                                    <svg class="w-4 h-4 ml-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                                                                            d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"/>
                                                                    </svg>
                                                                </a>
                                                            {% endif %}
                                                        </div>
                                                    {% endfor %}
                                                </div>
                                            {% endif %}
                                        </div>
                                    {% endfor %}
                                </div>
                            </div>
                        {% endif %}
                    {% endfor %}
                {% else %}
                    <div class="text-brand-gray italic text-center py-8">
                        <svg class="w-16 h-16 mx-auto mb-4 text-green-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                                d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"/>
                        </svg>
                        <p>No policy changes detected.</p>
                    </div>
                {% endif %}

                <div class="mt-6 pt-4 border-t border-gray-200">
                    <p class="text-gray-600 text-sm italic">This card shows policies that changed compliance status after policy release.</p>
                </div>
            </div>
        </div>

        <!-- All Non-Compliant Policies Section -->
        <div class="bg-white rounded-lg shadow mb-8">
            <div class="px-6 py-4 border-b border-gray-200">
                <h3 class="text-lg font-medium text-brand-blue flex items-center">
                    <svg class="w-5 h-5 mr-2 text-red-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                            d="M9 12l2 2 4-4m5.618-4.016A11.955 11.955 0 0112 2.944a11.955 11.955 0 01-8.618 3.04A12.02 12.02 0 003 9c0 5.591 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.042-.133-2.052-.382-3.016z"/>
                    </svg>
                    All Non-Compliant Policies
                </h3>
            </div>
            <div class="p-6 space-y-6">
                {% set grouped_policies = {} %}
                {% for hub_name, hub_policies in compliance_state.items() %}
                    {% for policy_key, policy in hub_policies.items() %}
                        {% if policy.overall_compliance == 'NonCompliant' %}
                            {% set key_parts = policy_key.split('/') %}
                            {% set namespace = key_parts[0] %}
                            {% set policy_name = key_parts[1:] | join('/') %}
                            
                            {% set display_key = policy_name ~ ' (' ~ namespace ~ ')' %}
                            
                            {% if display_key not in grouped_policies %}
                                {% set _ = grouped_policies.update({
                                    display_key: {
                                        'details': policy.details,
                                        'hubs': {},
                                        'total_violations': 0,
                                        'namespace': namespace,
                                        'policy_name': policy_name
                                    }
                                }) %}
                            {% endif %}
                            {% set _ = grouped_policies[display_key].hubs.update({
                                hub_name: {
                                    'namespace': policy.namespace,
                                    'cluster_status': policy.cluster_status
                                }
                            }) %}
                            {% set _ = grouped_policies[display_key].update({
                                'total_violations': grouped_policies[display_key].total_violations + (policy.cluster_status|length)
                            }) %}
                        {% endif %}
                    {% endfor %}
                {% endfor %}

                {% for display_key, policy_data in grouped_policies.items() %}
                    <div class="policy-card bg-brand-lightblue rounded-lg p-6">
                        <div class="flex justify-between items-start">
                            <div class="flex-grow">
                                <div class="flex items-start justify-between">
                                    <div>
                                        <h4 class="text-lg font-medium text-brand-blue">{{ policy_data.policy_name }}</h4>
                                        <span class="text-sm text-brand-gray">Namespace: {{ policy_data.namespace }}</span>
                                    </div>
                                    <div class="flex flex-wrap gap-2 ml-4">
                                        <span class="px-3 py-1 text-xs font-semibold rounded-full
                                            {{ 'bg-purple-100 text-purple-800' if policy_data.details.remediation_action == 'enforce' 
                                            else 'bg-blue-100 text-blue-800' }}">
                                            {{ policy_data.details.remediation_action|title }}
                                        </span>
                                        <span class="px-3 py-1 text-xs font-semibold rounded-full bg-red-100 text-red-800">
                                            {{ policy_data.total_violations }} violation(s)
                                        </span>
                                    </div>
                                </div>
                                {% if policy_data.details.description %}
                                    <p class="text-sm text-brand-gray mt-2">{{ policy_data.details.description }}</p>
                                {% endif %}
                            </div>
                        </div>

                        <div class="mt-4">
                            <div class="grid grid-cols-1 gap-4">
                                {% for hub_name, hub_data in policy_data.hubs.items() %}
                                    <div class="bg-white rounded-lg p-4 shadow-sm">
                                        <div class="flex items-center justify-between mb-3 pb-2 border-b border-gray-100">
                                            <div>
                                                <span class="font-medium text-brand-blue">Prime Cluster: {{ hub_name }}</span>
                                                <p class="text-sm text-brand-gray mt-1">Namespace: {{ hub_data.namespace }}</p>
                                            </div>
                                            <span class="text-xs bg-brand-lightblue px-2 py-1 rounded-full text-brand-blue">
                                                {{ hub_data.cluster_status|length }} non-compliant cluster(s)
                                            </span>
                                        </div>
                                        <div class="space-y-3">
                                            {% for cluster_name, cluster_data in hub_data.cluster_status.items() %}
                                                <div class="flex items-center justify-between py-2 px-3 bg-gray-50 rounded-lg">
                                                    <span class="text-sm text-brand-gray">{{ cluster_name }}</span>
                                                    {% if cluster_data.console_url %}
                                                        <a href="{{ cluster_data.console_url }}" 
                                                        target="_blank"
                                                        class="text-sm text-brand-blue hover:text-brand-blue/80 flex items-center group">
                                                            View Violation Message
                                                            <svg class="w-4 h-4 ml-1 transition-transform group-hover:translate-x-0.5" 
                                                                fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                                                                    d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"/>
                                                            </svg>
                                                        </a>
                                                    {% endif %}
                                                </div>
                                            {% endfor %}
                                        </div>
                                    </div>
                                {% endfor %}
                            </div>
                        </div>
                    </div>
                {% endfor %}
            </div>
        </div>
    </div>
</div>

<!-- Chart Initialization Script -->
<script>
document.addEventListener('DOMContentLoaded', function() {
// Donut Chart for Policy Compliance Overview
const complianceCtx = document.getElementById('complianceChart').getContext('2d');
new Chart(complianceCtx, {
type: 'doughnut',
data: {
    labels: ['Compliant', 'Non-Compliant', 'Changed'],
    datasets: [{
        data: [
            {{ comparison.summary.compliant|default(0) }},
            {{ comparison.summary.total_noncompliant|default(0) }},
            {{ comparison.summary.changed|default(0) }}
        ],
        backgroundColor: [
            '#22C55E', // green
            '#EF4444', // red
            '#EAB308'  // yellow
        ],
        borderWidth: 0
    }]
},
options: {
    responsive: true,
    maintainAspectRatio: false,
    cutout: '60%',
    plugins: {
        legend: {
            position: 'bottom',
            labels: {
                padding: 20,
                font: {
                    size: 12
                }
            }
        },
        tooltip: {
            callbacks: {
                label: function(context) {
                    return `${context.label}: ${context.raw}`;
                }
            }
        }
    }
}
});

// Bar Chart for Non-Compliant Policies by Prime Cluster
const clusterCtx = document.getElementById('clusterChart').getContext('2d');
new Chart(clusterCtx, {
type: 'bar',
data: {
    labels: {{ comparison.summary.cluster_names|tojson|safe }},
    datasets: [{
        label: 'Non-Compliant Policies',
        data: {{ comparison.summary.cluster_noncompliant|tojson|safe }},
        backgroundColor: '#00658F',
        borderWidth: 0
    }]
},
options: {
    responsive: true,
    maintainAspectRatio: false,
    scales: {
        y: {
            beginAtZero: true,
            ticks: {
                stepSize: 1
            }
        }
    },
    plugins: {
        legend: {
            display: false
        },
        tooltip: {
            callbacks: {
                label: function(context) {
                    return `Non-Compliant Policies: ${context.raw}`;
                }
            }
        }
    }
}
});
});
</script>

</body>
</html>
