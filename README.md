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

        <!-- Policy Changes Section (Fixed) -->
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
                    {% set grouped_changes = {} %}
                    {% for display_key, policy in comparison.consolidated_view.policies.items() %}
                        {% if policy.status_change in ['became_noncompliant', 'became_compliant'] %}
                            {% set actual_name = policy.policy_name %}
                            
                            {% if actual_name not in grouped_changes %}
                                {% set _ = grouped_changes.update({
                                    actual_name: {
                                        'instances': [],
                                        'description': policy.details.description,
                                        'remediation_action': policy.details.remediation_action,
                                        'status_change': policy.status_change
                                    }
                                }) %}
                            {% endif %}
                            
                            {% set instance = {
                                'namespace': policy.namespace,
                                'hubs': policy.hubs
                            } %}
                            
                            {% set _ = grouped_changes[actual_name].instances.append(instance) %}
                        {% endif %}
                    {% endfor %}
                    
                    {% for policy_name, policy_data in grouped_changes.items() %}
                        <div class="policy-card {% if policy_data.status_change == 'became_noncompliant' %}bg-red-50{% else %}bg-green-50{% endif %} rounded-lg p-6 mb-4 last:mb-0">
                            <div class="flex justify-between items-start mb-4">
                                <div>
                                    <h4 class="text-lg font-medium {% if policy_data.status_change == 'became_noncompliant' %}text-red-900{% else %}text-green-900{% endif %}">
                                        {{ policy_name }}
                                    </h4>
                                    {% if policy_data.description %}
                                        <p class="text-sm {% if policy_data.status_change == 'became_noncompliant' %}text-red-800{% else %}text-green-800{% endif %} mt-1">
                                            {{ policy_data.description }}
                                        </p>
                                    {% endif %}
                                </div>
                                <div class="flex space-x-2">
                                    <span class="px-3 py-1 text-xs font-semibold rounded-full 
                                        {{ 'bg-purple-100 text-purple-800' if policy_data.remediation_action == 'enforce' 
                                        else 'bg-blue-100 text-blue-800' }}">
                                        {{ policy_data.remediation_action|title }}
                                    </span>
                                    <span class="status-badge {{ 'became-noncompliant' if policy_data.status_change == 'became_noncompliant' else 'became-compliant' }}">
                                        {% if policy_data.status_change == 'became_noncompliant' %}
                                            Became Non-Compliant
                                        {% else %}
                                            Became Compliant
                                        {% endif %}
                                    </span>
                                </div>
                            </div>

                            <div class="space-y-3 divide-y {% if policy_data.status_change == 'became_noncompliant' %}divide-red-100{% else %}divide-green-100{% endif %}">
                                {% for instance in policy_data.instances %}
                                    <div class="pt-3 first:pt-0">
                                        <div class="{% if policy_data.status_change == 'became_noncompliant' %}bg-white/50{% else %}bg-white/50{% endif %} rounded-lg overflow-hidden">
                                            <div class="font-medium {% if policy_data.status_change == 'became_noncompliant' %}text-red-900{% else %}text-green-900{% endif %} p-3">
                                                <span>Namespace: {{ instance.namespace }}</span>
                                            </div>
                                            
                                            <div class="space-y-3 p-3">
                                                {% for hub_name, hub_data in instance.hubs.items() %}
                                                    <div class="bg-white rounded-lg p-3 shadow-sm">
                                                        <div class="flex items-center justify-between pb-2 mb-2 border-b {% if policy_data.status_change == 'became_noncompliant' %}border-red-100{% else %}border-green-100{% endif %}">
                                                            <div>
                                                                <span class="font-medium {% if policy_data.status_change == 'became_noncompliant' %}text-red-700{% else %}text-green-700{% endif %}">
                                                                    Prime Cluster: {{ hub_name }}
                                                                </span>
                                                            </div>
                                                            <div class="text-xs {% if policy_data.status_change == 'became_noncompliant' %}bg-red-100 text-red-800{% else %}bg-green-100 text-green-800{% endif %} px-2 py-1 rounded-full">
                                                                {{ hub_data.current_status }}
                                                            </div>
                                                        </div>
                                                        
                                                        {% if policy_data.status_change == 'became_noncompliant' and hub_data.noncompliant_clusters %}
                                                            <div class="space-y-2">
                                                                {% for cluster in hub_data.noncompliant_clusters %}
                                                                    <div class="flex items-center justify-between py-2 px-3 {% if policy_data.status_change == 'became_noncompliant' %}bg-red-50{% else %}bg-green-50{% endif %} rounded-lg">
                                                                        <span class="text-sm {% if policy_data.status_change == 'became_noncompliant' %}text-red-800{% else %}text-green-800{% endif %}">
                                                                            {{ cluster.cluster }}
                                                                        </span>
                                                                        {% if cluster.console_url %}
                                                                            <a href="{{ cluster.console_url }}" 
                                                                            target="_blank"
                                                                            class="text-sm {% if policy_data.status_change == 'became_noncompliant' %}text-red-800 hover:text-red-600{% else %}text-green-800 hover:text-green-600{% endif %} flex items-center group">
                                                                                View in Console
                                                                                <svg class="w-4 h-4 ml-1 group-hover:translate-x-0.5 transition-transform" fill="none" stroke="currentColor" viewBox="0 0 24 24">
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
                                    </div>
                                {% endfor %}
                            </div>
                        </div>
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

        <!-- All Non-Compliant Policies Section (Fixed) -->
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
                {% set base_name_grouped_policies = {} %}
                {% for hub_name, hub_policies in compliance_state.items() %}
                    {% for policy_key, policy in hub_policies.items() %}
                        {% if policy.overall_compliance == 'NonCompliant' %}
                            {% set key_parts = policy_key.split('/') %}
                            {% set namespace = key_parts[0] %}
                            {% set policy_name = key_parts[1:] | join('/') %}
                            
                            {% if policy_name not in base_name_grouped_policies %}
                                {% set _ = base_name_grouped_policies.update({
                                    policy_name: {
                                        'instances': [],
                                        'total_violations': 0,
                                        'remediation_action': policy.details.remediation_action,
                                        'description': policy.details.description
                                    }
                                }) %}
                            {% endif %}
                            
                            {% set instance = {
                                'hub_name': hub_name,
                                'namespace': namespace,
                                'cluster_status': policy.cluster_status,
                                'violation_count': policy.cluster_status|length
                            } %}
                            
                            {% set _ = base_name_grouped_policies[policy_name].instances.append(instance) %}
                            {% set _ = base_name_grouped_policies[policy_name].update({
                                'total_violations': base_name_grouped_policies[policy_name].total_violations + instance.violation_count
                            }) %}
                        {% endif %}
                    {% endfor %}
                {% endfor %}

                {% for policy_name, policy_data in base_name_grouped_policies.items() %}
                    <div class="policy-card bg-brand-lightblue rounded-lg p-6">
                        <div class="flex justify-between items-start mb-4">
                            <div>
                                <h4 class="text-lg font-medium text-brand-blue">{{ policy_name }}</h4>
                                {% if policy_data.description %}
                                    <p class="text-sm text-brand-gray mt-1">{{ policy_data.description }}</p>
                                {% endif %}
                            </div>
                            <div class="flex space-x-2">
                                <span class="px-3 py-1 text-xs font-semibold rounded-full
                                    {{ 'bg-purple-100 text-purple-800' if policy_data.remediation_action == 'enforce' 
                                    else 'bg-blue-100 text-blue-800' }}">
                                    {{ policy_data.remediation_action|title }}
                                </span>
                                <span class="px-3 py-1 text-xs font-semibold rounded-full bg-red-100 text-red-800">
                                    {{ policy_data.total_violations }} violation(s)
                                </span>
                            </div>
                        </div>

                        <div class="space-y-3 divide-y divide-gray-100">
                            {% for instance in policy_data.instances %}
                                <div class="pt-3 first:pt-0">
                                    <div class="bg-white rounded-lg shadow-sm overflow-hidden">
                                        <div class="flex items-center justify-between p-4 bg-gray-50">
                                            <div>
                                                <div class="flex items-center">
                                                    <span class="font-medium text-brand-blue">{{ instance.hub_name }}</span>
                                                    <span class="mx-2 text-gray-400">•</span>
                                                    <span class="text-sm text-gray-600">{{ instance.namespace }}</span>
                                                </div>
                                            </div>
                                            <span class="text-xs bg-blue-100 text-blue-800 px-2 py-1 rounded-full">
                                                {{ instance.violation_count }} non-compliant cluster(s)
                                            </span>
                                        </div>
                                        
                                        <div class="p-4">
                                            <div class="space-y-3">
                                                {% for cluster_name, cluster_data in instance.cluster_status.items() %}
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
                                    </div>
                                </div>
                            {% endfor %}
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
