#!/bin/bash
#
# Improved NTP Log Archiver Script
# Called by autosys job to start the docker job

# Enable strict mode for better error detection
set -euo pipefail
IFS=$'\n\t'

# Script constants
SCRIPT_NAME="$(basename "$0")"
LOG_PREFIX="[${SCRIPT_NAME}]"

# Function definitions
log() {
    local level="$1"
    local message="$2"
    echo "${LOG_PREFIX} [${level}] ${message}" >&2
}

info() { log "INFO" "$1"; }
error() { log "ERROR" "$1"; }

check_command() {
    if ! command -v "$1" &> /dev/null; then
        error "Required command '$1' not found"
        exit 1
    fi
}

exit_gracefully() {
    local exit_code="${1:-1}"
    # Perform any cleanup here if needed
    exit "${exit_code}"
}

# Trap signals
trap 'error "Script interrupted"; exit_gracefully 1' SIGINT SIGTERM

# Check prerequisites
info "Checking prerequisites..."
check_command docker
check_command sq

# Module load setup for backward compatibility
info "Loading environment configuration..."

# Set up environment variables with proper quoting
export PATH="${PATH:-/ms/dist/sec/PROJ/scv/qa/bin}"
export IMAGE="mshub.webfarm.ms.com/cloudmskube/mks-finra-ntp-a360:2025.01.09-1"
info "Using image: ${IMAGE}"

# Configuration variables with defaults where appropriate
export MONTHS_LOOKBACK="${MONTHS_LOOKBACK:-1}"
export A360_DROPZONE_BASEDIR="/v/global/app1/ilink/era/data/dropzones2/a360_dropzones/EONID_135473"
export A360_SUBMISSION_PATH="${A360_DROPZONE_BASEDIR}/submission"
export A360_RESPONSE_PATH="${A360_DROPZONE_BASEDIR}/response"
export S3_ACCESS_KEY="mks360p"
export S3_ENDPOINT_URL="https://mks-logging-ms-openshift-na.ms3.na.ms.com"
export BUCKET_NAME="ntp-finra"

# CINIT configuration
CINITCNAME="$(ms/dist/sec/PROJ/cinit/dev/bin/cinit.py)"
export CINITCNAME
export DOCKER_TLS_VERIFY=1
export DOCKER_HOST="tcp://${HOSTNAME:-localhost}:2376"
export DOCKER_CERT_PATH="${CINITCNAME}"
export CA_BUNDLE="/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem"

# Define required environment variables array
declare -a required_envvars=(
    "MONTHS_LOOKBACK" 
    "S3_ENDPOINT_URL" 
    "A360_RESPONSE_PATH" 
    "A360_SUBMISSION_PATH" 
    "CLUSTER_ENV" 
    "S3_ACCESS_KEY" 
    "BUCKET_NAME"
)

# Initialize docker parameters
PARAMS=""

# Validate required environment variables
info "Validating environment variables..."
for envvar in "${required_envvars[@]}"; do
    if [[ -z "${!envvar:-}" ]]; then
        error "Missing environment variable: ${envvar}"
        exit_gracefully 1
    else
        # Build docker parameters securely with proper quoting
        PARAMS+=" -e ${envvar}='${!envvar}'"
    fi
done

# Add CLP_VERSION if defined
if [[ -n "${CLP_VERSION:-}" ]]; then
    info "Using CLP_VERSION: ${CLP_VERSION}"
    PARAMS+=" -e CLP_VERSION='${CLP_VERSION}'"
else
    info "No CLP_VERSION provided, auditing without version..."
fi

# Retrieve secret securely
info "Retrieving S3 secret..."
SECRET="$(sq tell ec/storage/object/ecs/users/prod ${S3_ACCESS_KEY})"
if [[ -z "${SECRET}" ]]; then
    error "Failed to retrieve S3 secret"
    exit_gracefully 1
fi

# Construct Docker command with best practices
COMMAND="docker run ${PARAMS} -e S3_SECRET_KEY='${SECRET}' \\
    --user $(id -u):$(id -g) \\
    --network host --cap-drop NET_BIND_SERVICE \\
    --cap-drop SETUID --cap-drop SETGID \\
    --name mks-finra-ntp-a360-archiver \\
    --security-opt=no-new-privileges \\
    --read-only \\
    --tmpfs /tmp:rw,noexec,nosuid \\
    -v ${CA_BUNDLE}:/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem:ro \\
    -v ${A360_DROPZONE_BASEDIR}:${A360_DROPZONE_BASEDIR}:rw \\
    ${IMAGE}"

# Run the container
info "Running NTP log archiver..."
echo "${COMMAND}" | grep -v "S3_SECRET_KEY" # Log command without secret
eval "${COMMAND}"

# Check execution status
if [[ $? -eq 0 ]]; then
    info "Docker container executed successfully"
    exit_gracefully 0
else
    error "Docker container execution failed"
    exit_gracefully 1
fi
