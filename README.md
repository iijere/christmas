#!/usr/bin/env python
# This script is run on MKS Nodes, parses log files in
# /var/log/chrony/ up until the oldest date found
# and exports the data to a s3 bucket for FINRA compliance.
##

# import ms.version
import logging
import logging.handlers as handlers
import os
import shutil
import sys
import tempfile
import time
import urllib3
import json
from datetime import datetime, timedelta
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

CHRONY_STATS_DIR = os.environ.get("CHRONY_STATS_DIR", "/host/var/log/chrony")
NODE_NAME = os.environ.get("NODE_NAME", "")
S3_ENDPOINT = os.environ.get("S3_ENDPOINT_URL", "")
S3_SECRET_SECRET_KEY = os.environ.get("S3_SECRET_SECRET_KEY", "")
S3_SECRET_ACCESS_KEY = os.environ.get("S3_SECRET_ACCESS_KEY", "")
BUCKET_NAME = os.environ.get("BUCKET_NAME", "")
MAX_DAYS_BACK = int(os.environ.get("MAX_DAYS_BACK", "30"))  # Default to 30 days

LOG_HEADER_BORDER = "======================================================================================================"
LOG_HEADER_COLUMNS = "    Date (UTC) Time      IP Address     Std dev'n Est offset  Offset sd  Diff freq   Est skew  Stress  Ns  Bs  Nr  Asym"
LOG_HEADER = LOG_HEADER_BORDER + '\n' + LOG_HEADER_COLUMNS + '\n' + LOG_HEADER_BORDER + '\n'

client_config = Config(
    retries = {
        'max_attempts': 10,
        'mode': 'standard'
    }
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

def generate_archival_log(target_date, source_log_content):
    """ Parse log contents, save output to file, return filepath """
    log_content = parse_source_log(target_date, source_log_content)
    if log_content == LOG_HEADER:
        logger.info(f"No NTP statistics logs found on host {NODE_NAME} to ship for date {target_date}")
        return None
    else:
        logger.info(f"Logs for date {target_date} found on host {NODE_NAME} found. Saving log...")
        saved_file_path = save_log(log_content, target_date)
        return saved_file_path

def parse_source_log(target_date, source_log_content):
    """ Parse the mounted ntp log for records of the target_date, formatted with proper log headers """
    log_content = LOG_HEADER
    
    for line in source_log_content.splitlines():
        if target_date in line:
            log_content += line+'\n'
    
    if log_content == LOG_HEADER:
        logger.info("No logs found for " + target_date)
    else:
        logger.info("Logs found for " + target_date)
    return log_content

def save_log(log_content, target_date):
    """ Write content to tmpfs, then save """
    tmp_file = tempfile.NamedTemporaryFile(prefix=NODE_NAME+target_date, suffix='.log')
    try:
        with open(tmp_file.name, "w+") as output_file:
            output_file.write(log_content)
            output_file.seek(0)
            output_file_name = output_file.name
            logger.info("Data written to temporary file. It is ready to be shipped to s3 as file: [{}]".format(output_file_name))
            return output_file_name
    except Exception as error:
        logger.error("Error saving log: [{}]".format(error))
        raise error

def start_client():
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
            logger.error("Error starting s3 client: [{}]".format(error))
            raise error
    else:
        logger.error("Missing environment variables necessary for client startup. S3_ENDPOINT: [{}] S3_SECRET_SECRET_KEY: [{}]  S3_SECRET_ACCESS_KEY: [{}]".format(S3_ENDPOINT,S3_SECRET_SECRET_KEY,S3_SECRET_ACCESS_KEY))

def upload_file_to_bucket(client, file_path, bucket_name, key_name):
    """ Attempt upload """
    try:
        logger.info("Attempting to upload log to bucket [{}] with key: [{}]".format(bucket_name, key_name))
        upload_response = client.upload_file(
            file_path,
            bucket_name,
            key_name
        )
        logger.info("Response received from s3: [{}]".format(upload_response))
        return upload_response
    except Exception as error:
        logger.error("Error uploading file to s3: [{}]".format(error))
        raise error

def try_get_object(client, object_key):
    if client and object_key:
        try:
            get_obj_response = client.get_object(
                Bucket=BUCKET_NAME,
                Key=object_key
            )
            return get_obj_response
        except Exception as error:
            logger.error("Error getting object from bucket: [{}]".format(error))
            raise error

# Original function - kept for backward compatibility
def get_log_keys(client, target_date):
    """ Get logs shipped from this cluster-node """
    logger.info("Getting log keys...")
    try:
        response = client.list_objects_v2(
            Bucket=BUCKET_NAME,
            Prefix=NODE_NAME+'-'+target_date
        )
        logger.debug("S3 log keys response: [{}]".format(response))
        keys = []
        for key_data in response.get("Contents", []):
            keys.append(key_data["Key"])
        return keys
    except Exception as error:
        logger.error("Error getting cluster-node ntps logs: [{}]".format(error))
        raise error

# New optimized function for batch retrieval of keys
def get_log_keys_batch(client, target_dates):
    """
    Get logs shipped from this cluster-node for multiple dates in fewer API calls
    Uses prefixes and pagination to reduce API calls
    """
    logger.info("Getting log keys for multiple dates...")
    all_keys = {}
    
    # Group dates by month to reduce number of API calls
    date_prefixes = {}
    for date in target_dates:
        formatted_date = datetime.strftime(date, '%Y-%m-%d')
        month_year = formatted_date[:7]  # Get YYYY-MM portion
        if month_year not in date_prefixes:
            date_prefixes[month_year] = []
        date_prefixes[month_year].append(formatted_date)
    
    # For each month, make one API call instead of one per day
    for month_year, dates in date_prefixes.items():
        try:
            # Use month prefix for query instead of specific day
            prefix = f"{NODE_NAME}-{month_year}"
            paginator = client.get_paginator('list_objects_v2')
            
            # Use pagination to handle large response sets
            for page in paginator.paginate(Bucket=BUCKET_NAME, Prefix=prefix):
                if 'Contents' not in page:
                    continue
                    
                for key_data in page.get('Contents', []):
                    key = key_data["Key"]
                    # Filter keys by specific dates after retrieving them
                    for date in dates:
                        if f"{NODE_NAME}-{date}" in key:
                            if date not in all_keys:
                                all_keys[date] = []
                            all_keys[date].append(key)
            
        except Exception as error:
            logger.error(f"Error getting cluster-node ntps logs for {month_year}: {error}")
            raise error
    
    return all_keys

# New function to efficiently get metadata
def get_log_metadata(client, all_keys):
    """
    Instead of downloading all logs, get metadata and track what we already have
    Returns a set of unique log entry identifiers (date + IP + timestamp)
    """
    existing_log_entries = set()
    download_count = 0
    max_downloads = 10  # Limit number of files to download for sampling
    
    # Sample approach: only download a subset of files to determine patterns
    for date, keys in all_keys.items():
        for key in keys:
            if download_count >= max_downloads:
                break
                
            try:
                log_s3_response = client.get_object(Bucket=BUCKET_NAME, Key=key)
                log_content = log_s3_response.get("Body").read().decode('utf-8')
                
                # Extract unique identifiers from each log line
                for line in log_content.splitlines():
                    if line and not line.startswith('=') and "IP Address" not in line:
                        # Parse the line to get a unique identifier
                        parts = line.split()
                        if len(parts) >= 3:
                            # Use date + time + IP as unique identifier
                            entry_id = f"{parts[0]}_{parts[1]}_{parts[2]}"
                            existing_log_entries.add(entry_id)
                
                download_count += 1
            except Exception as error:
                logger.error(f"Error processing S3 log {key}: {error}")
                continue
    
    logger.info(f"Identified {len(existing_log_entries)} unique log entries from {download_count} sample files")
    return existing_log_entries

# Original function - will be replaced
def get_united_s3_logs(client, log_keys):
    """
    Get all logs from S3 using keys from log_keys
    Collect them and return them
    """
    united_s3_logs = ""
    for log_key_date in log_keys:
        if log_key_date:
            for log_key in log_key_date:
                log_s3_response = try_get_object(client, log_key)
                united_s3_logs += log_s3_response.get("Body").read().decode('utf-8')
    return united_s3_logs

# Original function - will be replaced
def get_deduped_host_logs(united_s3_logs):
    """
    Collect all NTP stat logs from the Chrony stats dir,
    go through each line in each log,
    collect lines that are not in s3 and return them.
    """
    united_host_logs = ''
    for fp in os.listdir(CHRONY_STATS_DIR):
        full_fp = os.path.join(CHRONY_STATS_DIR, fp)
        with open(full_fp) as f:
            f.seek(0)
            for line in f:
                if line not in united_s3_logs:
                    united_host_logs += line
    return united_host_logs

# New optimized function for deduplication
def get_deduped_host_logs_efficient(existing_entries):
    """
    More efficient deduplication using set operations instead of string contains
    """
    deduped_logs = []
    entry_count = 0
    duplicate_count = 0
    
    # Add header to output
    deduped_logs.append(LOG_HEADER.strip())
    
    for fp in os.listdir(CHRONY_STATS_DIR):
        full_fp = os.path.join(CHRONY_STATS_DIR, fp)
        with open(full_fp) as f:
            for line in f:
                # Skip header lines
                if line.strip() and not line.startswith('=') and "IP Address" not in line:
                    parts = line.split()
                    if len(parts) >= 3:
                        entry_id = f"{parts[0]}_{parts[1]}_{parts[2]}"
                        entry_count += 1
                        
                        # If entry not in our existing set, add it to output
                        if entry_id not in existing_entries:
                            deduped_logs.append(line.strip())
                        else:
                            duplicate_count += 1
    
    logger.info(f"Processed {entry_count} log entries, found {duplicate_count} duplicates")
    return '\n'.join(deduped_logs)

def parse_date_from_filename(filename):
    fn_s = filename.split('-')
    is_rotation_log = len(fn_s) > 1
    if is_rotation_log:
        date = fn_s[1]
        month = date[4:6]
        day = date[6:8]
        year = date[0:4]
        return datetime.strptime(month + '-' + day + '-' + year, '%m-%d-%Y')
    else:
        return None

def get_oldest_log_file():
    """ Returns None if there are no log files.
    Finds and returns the path to the
    log with the oldest date within
    """
    log_files = [f for f in os.listdir(CHRONY_STATS_DIR)]
    if len(log_files) == 0:
        logger.error("No ntp logs found on host!")
        return None
    if len(log_files) == 1:
        return os.path.join(CHRONY_STATS_DIR, log_files[0])
    
    oldest = None
    full_fp = None
    for log in log_files:
        file_date = parse_date_from_filename(log)
        if file_date:
            if oldest == None:
                oldest = log
                full_fp = os.path.join(CHRONY_STATS_DIR, oldest)
            else:
                # We have to compare
                if file_date < parse_date_from_filename(oldest):
                    oldest = log
                    full_fp = os.path.join(CHRONY_STATS_DIR, oldest)
    return full_fp

def get_oldest_log_line_date():
    """ Find the oldest log file and then find the oldest date recorded within """
    full_fp = get_oldest_log_file()
    oldest_date_found = None
    if full_fp:
        with open(full_fp) as f:
            for line in f.readlines():
                # Skip empty lines and header lines
                line = line.strip()
                if not line or line.startswith('=') or "IP Address" in line or "Date" in line and "Time" in line:
                    continue
                
                try:
                    # Try to extract and parse the date from the beginning of the line
                    split = line.split()
                    if len(split) >= 1:
                        # Only try to parse if the first item looks like a date (YYYY-MM-DD)
                        date_str = split[0]
                        if len(date_str) == 10 and date_str[4] == '-' and date_str[7] == '-':
                            try:
                                log_line_date = datetime.strptime(date_str, '%Y-%m-%d')
                                
                                # Compare with current oldest date
                                if oldest_date_found is None or log_line_date < oldest_date_found:
                                    oldest_date_found = log_line_date
                                    logger.debug(f"Found date: {log_line_date} in line: {line[:30]}...")
                            except ValueError:
                                # Skip lines that don't parse as dates but don't fail the whole process
                                logger.debug(f"Skipping non-date line: {line[:30]}...")
                                continue
                except Exception as e:
                    # Log the error but continue processing other lines
                    logger.warning(f"Error processing line: {line[:30]}... - {str(e)}")
                    continue
                    
        if oldest_date_found is None:
            # If no valid dates found, default to recent date to avoid processing too much history
            oldest_date_found = datetime.now() - timedelta(days=7)
            logger.warning(f"No valid dates found in logs, defaulting to {oldest_date_found}")
    else:
        logger.error(f"No logs files to parse for oldest date on host {NODE_NAME}")
        # Default to 7 days ago to avoid errors in subsequent code
        oldest_date_found = datetime.now() - timedelta(days=7)
        
    logger.info(f"Oldest date found in NTP statistics log on host {NODE_NAME}: [{oldest_date_found}]")
    return oldest_date_found

# Original run function - replaced by optimized version
def run_original():
    """ Parses necessary source logs and ships to s3 """
    oldest_log_line_date = get_oldest_log_line_date()
    default_target_date = datetime.now()
    target_dates = []
    today = datetime.today()
    target_dates = [oldest_log_line_date + timedelta(days=x) for x in range((default_target_date-oldest_log_line_date).days)]
    target_dates.append(today.date())
    logger.info("Target dates: [{}]".format(target_dates))
    
    # Start client, generate log, and ship
    client = start_client()
    
    if client:
        logger.info("Client started.")
        log_keys = []
        # Query s3 for logs relevant to this cluster-node for each target date
        for date in target_dates:
            formatted_date = datetime.strftime(date, '%Y-%m-%d')
            log_keys.append(get_log_keys(client, formatted_date))
        logger.debug("Log keys retrieved from s3: [{}]".format(log_keys))
        
        if len(log_keys) == 0:
            logger.info("No keys found.")
        
        united_s3_logs = get_united_s3_logs(client, log_keys)
        host_logs_to_ship = get_deduped_host_logs(united_s3_logs)
        
        # united_host_logs should be records from host that dont exist on s3
        if host_logs_to_ship == "":
            logger.info(f"{NODE_NAME} has no NTP stats logs to ship.")
        else:
            logger.info(f"{NODE_NAME} has logs to ship.")
            logger.debug("Host logs to ship: [{}]".format(host_logs_to_ship))
        
        now = datetime.now().timestamp()
        for date in target_dates:
            formatted_date = datetime.strftime(date, '%Y-%m-%d')
            output_fp = generate_archival_log(formatted_date, host_logs_to_ship)
            
            if output_fp:
                upload_file_to_bucket(
                    client,
                    output_fp,
                    BUCKET_NAME,
                    NODE_NAME+'-'+formatted_date+'-'+str(now)
                )
    else:
        logger.error("Error with s3 client.")
        raise Exception("Client does not exist.")

# New optimized run function
def run():
    """ Optimized version that reduces S3 API calls """
    oldest_log_line_date = get_oldest_log_line_date()
    default_target_date = datetime.now()
    
    # Optimization: Consider limiting the date range
    days_difference = (default_target_date - oldest_log_line_date).days
    
    if days_difference > MAX_DAYS_BACK:
        logger.info(f"Limiting processing to last {MAX_DAYS_BACK} days instead of {days_difference} days")
        oldest_log_line_date = default_target_date - timedelta(days=MAX_DAYS_BACK)
    
    # Generate target dates
    target_dates = [oldest_log_line_date + timedelta(days=x) for x in range((default_target_date-oldest_log_line_date).days)]
    target_dates.append(default_target_date.date())
    logger.info(f"Processing {len(target_dates)} target dates")
    
    # Start client
    client = start_client()
    
    if client:
        logger.info("Client started.")
        
        # Batch retrieve keys for all dates (fewer API calls)
        all_keys = get_log_keys_batch(client, target_dates)
        
        # Get metadata instead of downloading all logs
        existing_entries = get_log_metadata(client, all_keys)
        
        # More efficient deduplication
        host_logs_to_ship = get_deduped_host_logs_efficient(existing_entries)
        
        if not host_logs_to_ship or host_logs_to_ship == LOG_HEADER.strip():
            logger.info(f"{NODE_NAME} has no new NTP stats logs to ship.")
        else:
            logger.info(f"{NODE_NAME} has logs to ship.")
            
            # Process and upload logs
            now = datetime.now().timestamp()
            for date in target_dates:
                formatted_date = datetime.strftime(date, '%Y-%m-%d')
                output_fp = generate_archival_log(formatted_date, host_logs_to_ship)
                
                if output_fp:
                    upload_file_to_bucket(
                        client,
                        output_fp,
                        BUCKET_NAME,
                        f"{NODE_NAME}-{formatted_date}-{now}"
                    )
    else:
        logger.error("Error with s3 client.")
        raise Exception("Client does not exist.")

if __name__ == "__main__":
    # Set up logging
    run()
