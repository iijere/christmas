#!/usr/bin/env python
# This script is run on MKS Nodes, parses log files in
# /var/log/chrony/ up until the oldest date found
# and exports the data to a s3 bucket for FINRA compliance.
# Optimized to reduce S3 API load across large clusters
##

import logging
import logging.handlers as handlers
import os
import sys
import tempfile
import time
import urllib3
import json
import hashlib
import random
from datetime import datetime, timedelta
from threading import Timer
from requests.exceptions import SSLError
from pythonjsonlogger import jsonlogger

# The following imports and config overrides
# are necessary for accessing the ms on-prem ecs3
# https://jive.ms.com/docs/DOC-814168
import requests
requests.packages.urllib3.util.ssl_.DEFAULT_CIPHERS += ':!DH'
import urllib3
urllib3.util.ssl_.DEFAULT_CIPHERS += ':!DH'
import boto3
from botocore.config import Config
from botocore.exceptions import ClientError

# Environment variables
CHRONY_STATS_DIR = os.environ.get("CHRONY_STATS_DIR", "/host/var/log/chrony")
NODE_NAME = os.environ.get("NODE_NAME", "")
CLUSTER_NAME = os.environ.get("CLUSTER_NAME", "")  # Added cluster name for organization
S3_ENDPOINT = os.environ.get("S3_ENDPOINT_URL", "")
S3_SECRET_SECRET_KEY = os.environ.get("S3_SECRET_SECRET_KEY", "")
S3_SECRET_ACCESS_KEY = os.environ.get("S3_SECRET_ACCESS_KEY", "")
BUCKET_NAME = os.environ.get("BUCKET_NAME", "")
# New env variables for optimizations
BATCH_SIZE = int(os.environ.get("BATCH_SIZE", "5"))  # Number of days to process in one batch
JITTER_MAX_SECONDS = int(os.environ.get("JITTER_MAX_SECONDS", "300"))  # Max seconds to delay execution
UPLOAD_FREQUENCY_HOURS = int(os.environ.get("UPLOAD_FREQUENCY_HOURS", "12"))  # How often to upload (hours)
USE_COMPRESSION = os.environ.get("USE_COMPRESSION", "true").lower() == "true"  # Whether to compress logs

LOG_HEADER_BORDER = "======================================================================================================"
LOG_HEADER_COLUMNS = "    Date (UTC) Time      IP Address     Std dev'n Est offset  Offset sd  Diff freq   Est skew  Stress  Ns  Bs  Nr  Asym"
LOG_HEADER = LOG_HEADER_BORDER + '\n' + LOG_HEADER_COLUMNS + '\n' + LOG_HEADER_BORDER + '\n'

# Improved client configuration with exponential backoff
client_config = Config(
    retries = {
        'max_attempts': 5,  # Reduced from 10
        'mode': 'adaptive',  # Changed to adaptive for better backoff strategy
    },
    connect_timeout = 5,
    read_timeout = 10,
    max_pool_connections = 10  # Limit concurrent connections
)

def set_log_handler():
    """ Set up log handler """
    json_handler = logging.StreamHandler(sys.stdout)
    json_handler.setFormatter(jsonlogger.JsonFormatter('%(asctime)s %(message)s %(levelname)s'))
    root_logger = logging.getLogger()
    root_logger.setLevel(os.getenv('LOG_LEVEL', 'INFO'))
    root_logger.handlers = [json_handler]
    
    logger = logging.getLogger(__name__)
    return logger

logger = set_log_handler()

def apply_jitter():
    """Apply a random jitter delay to prevent all nodes from hitting S3 simultaneously
       The jitter is calculated based on the node name to ensure consistent spreading
       Maximum jitter is capped at 25 minutes to ensure we complete within the 30-minute window"""
    # Create deterministic seed from node name
    seed_value = int(hashlib.md5(NODE_NAME.encode()).hexdigest(), 16)
    random.seed(seed_value)
    
    # Generate a jitter between 0-1500 seconds (0-25 minutes)
    # This leaves 5 minutes for execution to complete
    max_jitter = min(JITTER_MAX_SECONDS, 1500)  # Cap at 25 minutes
    jitter = random.randint(0, max_jitter)
    
    logger.info(f"Applying jitter delay of {jitter} seconds")
    time.sleep(jitter)

def compress_content(content):
    """Compress the content if compression is enabled"""
    if USE_COMPRESSION:
        import gzip
        import io
        out = io.BytesIO()
        with gzip.GzipFile(fileobj=out, mode="w") as f:
            f.write(content.encode('utf-8'))
        return out.getvalue()
    else:
        return content.encode('utf-8')

def decompress_content(compressed_content):
    """Decompress gzipped content"""
    if USE_COMPRESSION:
        import gzip
        import io
        with gzip.GzipFile(fileobj=io.BytesIO(compressed_content), mode="r") as f:
            return f.read().decode('utf-8')
    else:
        return compressed_content.decode('utf-8')

def batch_dates(dates, batch_size=BATCH_SIZE):
    """Batch dates to reduce number of S3 operations"""
    for i in range(0, len(dates), batch_size):
        yield dates[i:i+batch_size]

def generate_archival_log(target_dates, source_log_content):
    """Parse log contents for multiple dates, save output to file, return filepath"""
    combined_log_content = LOG_HEADER
    logs_found = False
    
    for target_date in target_dates:
        date_str = target_date.strftime('%Y-%m-%d') if isinstance(target_date, datetime) else target_date
        
        for line in source_log_content.splitlines():
            if date_str in line:
                combined_log_content += line + '\n'
                logs_found = True
    
    if not logs_found:
        logger.info(f"No NTP statistics logs found on host {NODE_NAME} to ship for dates {target_dates}")
        return None
    else:
        # Format date range for filename
        start_date = target_dates[0]
        end_date = target_dates[-1]
        if isinstance(start_date, datetime):
            start_date = start_date.strftime('%Y-%m-%d')
            end_date = end_date.strftime('%Y-%m-%d')
            
        date_range = f"{start_date}_to_{end_date}"
        logger.info(f"Logs for date range {date_range} found on host {NODE_NAME}. Saving log...")
        saved_file_path = save_log(combined_log_content, date_range)
        return saved_file_path

def save_log(log_content, date_range):
    """Write content to tmpfs, then save"""
    tmp_file = tempfile.NamedTemporaryFile(prefix=f"{NODE_NAME}_{date_range}_", suffix='.log', delete=False)
    try:
        compressed = USE_COMPRESSION
        file_content = compress_content(log_content) if compressed else log_content.encode('utf-8')
        
        with open(tmp_file.name, "wb") as output_file:
            output_file.write(file_content)
            output_file_name = output_file.name
            
        logger.info(f"Data written to temporary file {output_file_name} "
                  f"({'compressed' if compressed else 'uncompressed'}, {len(file_content)} bytes)")
        return output_file_name
    except Exception as error:
        logger.error(f"Error saving log: {error}")
        raise error

def start_client():
    """Create S3 client with improved connection settings"""
    if (S3_ENDPOINT and S3_SECRET_SECRET_KEY and S3_SECRET_ACCESS_KEY):
        try:
            client = boto3.client(
                service_name='s3',
                endpoint_url=S3_ENDPOINT,
                verify="/etc/pki/tls/certs/ca-bundle.crt",
                aws_secret_access_key=S3_SECRET_SECRET_KEY,
                aws_access_key_id=S3_SECRET_ACCESS_KEY,
                config=client_config
            )
            return client
        except Exception as error:
            logger.error(f"Error starting s3 client: {error}")
            raise error
    else:
        logger.error(f"Missing environment variables necessary for client startup. "
                    f"S3_ENDPOINT: [{S3_ENDPOINT}] S3_SECRET_SECRET_KEY: [***] "
                    f"S3_SECRET_ACCESS_KEY: [***]")
        return None

def get_s3_key_prefix():
    """Generate a consistent S3 key prefix structure"""
    return f"{CLUSTER_NAME}/{NODE_NAME}" if CLUSTER_NAME else NODE_NAME

def upload_file_to_bucket(client, file_path, bucket_name, key_name):
    """Upload file with exponential backoff retry logic"""
    try:
        logger.info(f"Uploading log to bucket [{bucket_name}] with key: [{key_name}]")
        
        # Determine content type and encoding based on compression
        content_type = 'application/gzip' if USE_COMPRESSION else 'text/plain'
        content_encoding = 'gzip' if USE_COMPRESSION else None
        
        extra_args = {
            'ContentType': content_type
        }
        
        if content_encoding:
            extra_args['ContentEncoding'] = content_encoding
            
        # Add metadata to help with management
        extra_args['Metadata'] = {
            'cluster': CLUSTER_NAME if CLUSTER_NAME else 'unknown',
            'node': NODE_NAME,
            'compressed': str(USE_COMPRESSION).lower(),
            'created': datetime.now().isoformat()
        }
            
        upload_response = client.upload_file(
            file_path,
            bucket_name,
            key_name,
            ExtraArgs=extra_args
        )
        
        logger.info("Upload successful")
        return upload_response
    except Exception as error:
        logger.error(f"Error uploading file to s3: {error}")
        raise error

def get_log_keys_for_date_range(client, start_date, end_date):
    """Get log keys for a date range"""
    prefix = get_s3_key_prefix()
    
    # Convert dates to strings if they're not already
    start_date_str = start_date.strftime('%Y-%m-%d') if hasattr(start_date, 'strftime') else start_date
    end_date_str = end_date.strftime('%Y-%m-%d') if hasattr(end_date, 'strftime') else end_date
    
    logger.info(f"Getting log keys with prefix {prefix} for date range {start_date} to {end_date}")
    
    try:
        # Use pagination to handle large results more efficiently
        paginator = client.get_paginator('list_objects_v2')
        pages = paginator.paginate(
            Bucket=BUCKET_NAME,
            Prefix=prefix
        )
        
        keys = []
        for page in pages:
            if 'Contents' in page:
                for key_data in page['Contents']:
                    key = key_data['Key']
                    # Check if the key is within our date range
                    if is_key_in_date_range(key, start_date, end_date):
                        keys.append(key)
        
        logger.debug(f"Found {len(keys)} log keys in S3")
        return keys
    except Exception as error:
        logger.error(f"Error getting log keys: {error}")
        raise error

def is_key_in_date_range(key, start_date, end_date):
    """Check if a S3 key name contains a date that falls within the given range"""
    # Example key format: clustername/nodename-YYYY-MM-DD-timestamp
    parts = key.split('-')
    if len(parts) < 2:
        return False
    
    try:
        # Try to extract the date part (assuming format YYYY-MM-DD)
        date_part = parts[-3] if len(parts) >= 3 else None
        if date_part and len(date_part) == 10:  # YYYY-MM-DD is 10 chars
            key_date = datetime.strptime(date_part, '%Y-%m-%d').date()
            start = datetime.strptime(start_date, '%Y-%m-%d').date() if isinstance(start_date, str) else start_date.date()
            end = datetime.strptime(end_date, '%Y-%m-%d').date() if isinstance(end_date, str) else end_date.date()
            return start <= key_date <= end
    except (ValueError, IndexError):
        pass
    
    return False

def get_log_content_from_s3(client, keys):
    """Get and combine log content from multiple S3 keys"""
    if not keys:
        return ""
        
    united_content = ""
    for key in keys:
        try:
            response = client.get_object(Bucket=BUCKET_NAME, Key=key)
            content = response['Body'].read()
            
            # Check if content is compressed (either by Content-Encoding or our own logic)
            is_compressed = (response.get('ContentEncoding') == 'gzip' or 
                            response.get('ContentType') == 'application/gzip' or
                            key.endswith('.gz'))
            
            if is_compressed:
                decoded_content = decompress_content(content)
            else:
                decoded_content = content.decode('utf-8')
                
            united_content += decoded_content
            
        except Exception as error:
            logger.warning(f"Error retrieving S3 object {key}: {error}")
            continue
            
    return united_content

def get_host_logs_content():
    """Get all log content from host"""
    combined_content = ""
    for filename in os.listdir(CHRONY_STATS_DIR):
        if filename.startswith("."):
            continue  # Skip hidden files
            
        full_path = os.path.join(CHRONY_STATS_DIR, filename)
        try:
            with open(full_path, 'r') as f:
                combined_content += f.read()
        except Exception as error:
            logger.warning(f"Error reading log file {full_path}: {error}")
            
    return combined_content

def get_deduped_host_logs(united_s3_logs):
    """
    Get logs from host that don't exist in S3 yet
    Modified to read whole files at once for better performance
    """
    # If we have no S3 logs yet, return all host logs
    if not united_s3_logs:
        return get_host_logs_content()
    
    # Create a set of lines from S3 for faster lookups
    s3_lines = set(united_s3_logs.splitlines())
    
    deduped_lines = []
    for filename in os.listdir(CHRONY_STATS_DIR):
        if filename.startswith("."):
            continue  # Skip hidden files
            
        full_path = os.path.join(CHRONY_STATS_DIR, filename)
        try:
            with open(full_path, 'r') as f:
                for line in f:
                    # Only add lines that aren't already in S3
                    clean_line = line.rstrip('\n')
                    if clean_line and clean_line not in s3_lines:
                        deduped_lines.append(clean_line)
        except Exception as error:
            logger.warning(f"Error reading log file {full_path}: {error}")
    
    return '\n'.join(deduped_lines)

def get_date_range_to_process():
    """Determine the date range that needs to be processed"""
    try:
        # Get the oldest date in local logs
        oldest_date = get_oldest_log_line_date()
        if not oldest_date:
            logger.warning("No log dates found on host")
            return []
            
        # Calculate all dates from oldest to today
        today = datetime.today().date()
        date_range = []
        current = oldest_date.date()
        
        while current <= today:
            date_range.append(current)
            current += timedelta(days=1)
            
        return date_range
    except Exception as error:
        logger.error(f"Error determining date range: {error}")
        return []

def parse_date_from_filename(filename):
    """Parse date from a log filename"""
    fn_s = filename.split('-')
    is_rotation_log = len(fn_s) > 1
    if is_rotation_log:
        date = fn_s[1]
        if len(date) >= 8:  # Make sure we have enough characters
            try:
                month = date[4:6]
                day = date[6:8]
                year = date[0:4]
                return datetime.strptime(month + '-' + day + '-' + year, '%m-%d-%Y')
            except ValueError:
                pass
    return None

def get_oldest_log_file():
    """Returns the path to the oldest log file"""
    log_files = [f for f in os.listdir(CHRONY_STATS_DIR) if not f.startswith('.')]
    if not log_files:
        logger.error("No ntp logs found on host!")
        return None
    if len(log_files) == 1:
        return os.path.join(CHRONY_STATS_DIR, log_files[0])
    
    oldest = None
    full_fp = None
    for log in log_files:
        file_date = parse_date_from_filename(log)
        if file_date:
            if oldest is None:
                oldest = log
                full_fp = os.path.join(CHRONY_STATS_DIR, oldest)
            elif file_date < parse_date_from_filename(oldest):
                oldest = log
                full_fp = os.path.join(CHRONY_STATS_DIR, oldest)
    return full_fp

def get_oldest_log_line_date():
    """Find the oldest date within log files"""
    full_fp = get_oldest_log_file()
    oldest_date_found = None
    if full_fp:
        try:
            with open(full_fp) as f:
                for line in f:
                    try:
                        if line.strip() and line.strip() != LOG_HEADER_BORDER and line.strip() != LOG_HEADER_COLUMNS.strip():
                            split = line.split()
                            if len(split) > 0:
                                # Try to parse the date (first column)
                                try:
                                    log_line_date = datetime.strptime(split[0], '%Y-%m-%d')
                                    if oldest_date_found is None or log_line_date < oldest_date_found:
                                        oldest_date_found = log_line_date
                                except ValueError:
                                    # Not a date in expected format, skip
                                    pass
                    except Exception as inner_error:
                        logger.debug(f"Error parsing line in log file: {inner_error}")
                        continue
        except Exception as error:
            logger.error(f"Error reading log file {full_fp}: {error}")
    
    if not oldest_date_found:
        # Fallback to 30 days ago if we couldn't find a date
        oldest_date_found = datetime.now() - timedelta(days=30)
        logger.warning(f"Could not determine oldest date, using fallback: {oldest_date_found}")
    else:
        logger.info(f"Oldest date found in NTP statistics log: {oldest_date_found}")
        
    return oldest_date_found

def run():
    """Main execution with optimizations to reduce S3 API load"""
    # Apply random jitter to spread out requests
    apply_jitter()
    
    # Determine date range to process
    all_dates = get_date_range_to_process()
    if not all_dates:
        logger.warning("No dates to process, exiting")
        return
        
    logger.info(f"Processing {len(all_dates)} days of logs from {all_dates[0]} to {all_dates[-1]}")
    
    # Start S3 client
    client = start_client()
    if not client:
        logger.error("Failed to initialize S3 client")
        return
        
    # Process dates in batches
    for date_batch in batch_dates(all_dates):
        start_date = date_batch[0]
        end_date = date_batch[-1]
        
        # Format dates for logging
        start_str = start_date.strftime('%Y-%m-%d') if isinstance(start_date, datetime) else start_date
        end_str = end_date.strftime('%Y-%m-%d') if isinstance(end_date, datetime) else end_date
        
        logger.info(f"Processing batch from {start_str} to {end_str}")
        
        # Get existing S3 logs for this date range
        log_keys = get_log_keys_for_date_range(client, start_str, end_str)
        
        # Get content from S3 logs
        s3_log_content = get_log_content_from_s3(client, log_keys)
        
        # Get new content from host logs (deduped against S3)
        host_logs_to_ship = get_deduped_host_logs(s3_log_content)
        
        if not host_logs_to_ship:
            logger.info(f"No new logs to ship for date range {start_str} to {end_str}")
            continue
            
        # Generate archive log for the batch
        output_fp = generate_archival_log(date_batch, host_logs_to_ship)
        
        if output_fp:
            # Create a unique key with date range and timestamp
            timestamp = int(datetime.now().timestamp())
            s3_key = f"{get_s3_key_prefix()}/{start_str}_to_{end_str}_{timestamp}"
            if USE_COMPRESSION:
                s3_key += ".gz"
                
            # Upload to S3
            try:
                upload_file_to_bucket(client, output_fp, BUCKET_NAME, s3_key)
                # Clean up temp file
                os.unlink(output_fp)
            except Exception as error:
                logger.error(f"Failed to upload batch {start_str} to {end_str}: {error}")
                # Continue with next batch rather than failing entirely
                continue

if __name__ == "__main__":
    try:
        run()
    except Exception as e:
        logger.error(f"Unhandled exception in main execution: {e}")
        sys.exit(1)
