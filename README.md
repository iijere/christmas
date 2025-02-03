<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Multi-Hub RHACM Policy Validation Report</title>
    <!-- Load TailwindCSS and Chart.js from internal S3 -->
    <script src="https://rhacm.internal.s3.com/policies/scripts/tailwind.min.js" onload="console.log('Tailwind loaded')" onerror="console.error('Failed to load Tailwind')"></script>
    <script src="https://rhacm.internal.s3.com/policies/scripts/chart.min.js" onload="console.log('Chart.js loaded')" onerror="console.error('Failed to load Chart.js')"></script>
    <!-- Ensure libraries are loaded before configuration -->
    <script>
        // Wait for Tailwind to be available
        function configureTailwind() {
            if (window.tailwind) {
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
                };
            } else {
                console.error('Tailwind not loaded');
            }
        }

        // Initialize on page load
        window.addEventListener('load', function() {
            configureTailwind();
            if (!window.Chart) {
                console.error('Chart.js not loaded');
                document.querySelectorAll('canvas').forEach(canvas => {
                    const div = document.createElement('div');
                    div.className = 'bg-red-50 text-red-800 p-4 rounded-lg mt-4';
                    div.innerHTML = 'Chart library failed to load. Please check your network connection.';
                    canvas.parentNode.insertBefore(div, canvas);
                    canvas.style.display = 'none';
                });
            }
        });
    </script>
    <!-- Fallback styles in case Tailwind fails to load -->
    <style>
        .bg-red-50 { background-color: #fef2f2; }
        .text-red-800 { color: #991b1b; }
        .p-4 { padding: 1rem; }
        .rounded-lg { border-radius: 0.5rem; }
        .mt-4 { margin-top: 1rem; }
    </style>
</head>
    <style>
        .policy-card {
            transition: all 0.2s ease-in-out;
        }
        .policy-card:hover {
            transform: translateY(-2px);
        }
        .charts-section {
            position: relative;
            height: 250px;
            margin-bottom: 2rem;
            overflow: hidden;
        }
        .chart-container {
            position: absolute;
            top: 0;
            width: 50%;
            height: 100%;
            padding: 1rem;
        }
        .chart-container.left {
            left: 0;
        }
        .chart-container.right {
            right: 0;
        }
        .chart-box {
            background: white;
            border-radius: 0.5rem;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
            height: 100%;
            padding: 1rem;
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
                            Total Hubs: <span class="font-bold">{{ comparison.summary.total_hubs }}</span>
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
                    <h3 class="text-lg font-medium text-ms-blue mb-4">Overall Compliance Status</h3>
                    <canvas id="complianceChart" height="200"></canvas>
                </div>
                
                <!-- Policy Status by Hub -->
                <div class="bg-white rounded-lg shadow p-6">
                    <h3 class="text-lg font-medium text-ms-blue mb-4">Policy Status by Hub</h3>
                    <canvas id="hubStatusChart" height="200"></canvas>
                </div>
            </div>
        </div>

        <!-- Hub Navigation -->
        <div class="max-w-7xl mx-auto px-4 mb-8">
            <div class="bg-white rounded-lg shadow">
                <div class="px-6 py-4 border-b border-gray-200">
                    <h2 class="text-lg font-medium text-ms-blue">Hub Overview</h2>
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
                <!-- Hub Stats -->
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
                                                            {% if cluster.message and cluster.message != '#' %}
                                                                <div class="mt-2">
                                                                    <a href="{{ cluster.message }}" target="_blank" rel="noopener noreferrer" 
                                                                       class="text-ms-blue hover:text-ms-blue/80 underline">
                                                                        View in Console
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
                                <div class="text-ms-gray italic">No policies became non-compliant during this deployment</div>
                            {% endif %}
                        </div>
                    </div>
                </div>

                <!-- Non-Compliant Policies Detail -->
                <div class="mb-8">
                    <div class="bg-white rounded-lg shadow">
                        <div class="px-6 py-4 border-b border-gray-200">
                            <h3 class="text-lg font-medium text-ms-blue flex items-center">
                                <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"/>
                                </svg>
                                Non-Compliant Policies Detail
                            </h3>
                        </div>
                        <div class="p-6">
                            {% set has_violations = false %}
                            {% for policy_name, policy in hub_data.policies.items() %}
                                {% if policy.post_compliance == 'NonCompliant' %}
                                    {% set has_violations = true %}
                                    <div class="policy-card bg-ms-lightblue rounded-lg p-6 mb-4 last:mb-0">
                                        <div class="flex justify-between items-start">
                                            <div>
                                                <h4 class="text-lg font-medium text-ms-blue">{{ policy_name }}</h4>
                                                {% if policy.details.description %}
                                                    <p class="text-sm text-ms-gray mt-1">{{ policy.details.description }}</p>
                                                {% endif %}
                                            </div>
                                            <div class="flex items-center space-x-2">
                                                <span class="px-3 py-1 text-xs font-semibold rounded-full {{ 'bg-purple-100 text-purple-800' if policy.details.remediation_action == 'enforce' else 'bg-blue-100 text-blue-800' }}">
                                                    {{ policy.details.remediation_action|title }}
                                                </span>
                                                <span class="px-3 py-1 text-xs font-semibold rounded-full {{ 'bg-red-100 text-red-800' if policy.compliance_category == 'newly_noncompliant' else 'bg-orange-100 text-orange-800' }}">
                                                    {{ 'Newly Non-Compliant' if policy.compliance_category == 'newly_noncompliant' else 'Still Non-Compliant' }}
                                                </span>
                                            </div>
                                        </div>
                                        {% if policy.noncompliant_clusters %}
                                            <div class="mt-4">
                                                <h5 class="text-sm font-medium text-ms-blue mb-2">Affected Clusters:</h5>
                                                <div class="space-y-3">
                                                    {% for cluster in policy.noncompliant_clusters %}
                                                        <div class="bg-white rounded-lg p-4 shadow-sm">
                                                            <div class="font-medium text-ms-blue">{{ cluster.cluster }}</div>
                                                            {% if cluster.message and cluster.message != '#' %}
                                                                <div class="mt-2">
                                                                    <a href="{{ cluster.message }}" target="_blank" rel="noopener noreferrer" 
                                                                       class="text-ms-blue hover:text-ms-blue/80 underline">
                                                                        View in Console
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
                            {% if not has_violations %}
                                <div class="text-ms-gray italic">No non-compliant policies found</div>
                            {% endif %}
                        </div>
                    </div>
                </div>
            </div>
        </div>
        {% endfor %}
    </div>

    <script>
        function initializeCharts() {
            // Overall Compliance Status Chart
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
                            '#22C55E',  // Green
                            '#EF4444',  // Red
                            '#F59E0B'   // Amber
                        ],
                        borderWidth: 0
                    }]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: {
                        legend: {
                            position: 'bottom'
                        }
                    },
                    cutout: '65%'
                }
            });

            // Policy Status by Hub Chart
            const hubStatusCtx = document.getElementById('hubStatusChart').getContext('2d');
            new Chart(hubStatusCtx, {
                type: 'bar',
                data: {
                    labels: [
                        {% for hub_name in comparison.hubs.keys() %}
                            '{{ hub_name }}',
                        {% endfor %}
                    ],
                    datasets: [
                        {
                            label: 'Compliant',
                            data: [
                                {% for hub_name, hub_data in comparison.hubs.items() %}
                                    {{ hub_data.summary.total_policies - hub_data.summary.currently_noncompliant }},
                                {% endfor %}
                            ],
                            backgroundColor: '#22C55E'
                        },
                        {
                            label: 'Non-Compliant',
                            data: [
                                {% for hub_name, hub_data in comparison.hubs.items() %}
                                    {{ hub_data.summary.currently_noncompliant }},
                                {% endfor %}
                            ],
                            backgroundColor: '#EF4444'
                        },
                        {
                            label: 'Changed',
                            data: [
                                {% for hub_name, hub_data in comparison.hubs.items() %}
                                    {{ hub_data.summary.policies_changed }},
                                {% endfor %}
                            ],
                            backgroundColor: '#F59E0B'
                        }
                    ]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: {
                        legend: {
                            position: 'bottom'
                        }
                    },
                    scales: {
                        x: {
                            stacked: true,
                        },
                        y: {
                            stacked: true,
                            beginAtZero: true,
                            ticks: {
                                precision: 0
                            }
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
            const firstHubButton = document.querySelector('.hub-button');
            if (firstHubButton) {
                firstHubButton.click();
            }
            initializeCharts();
        });
    </script>
</body>
</html>
