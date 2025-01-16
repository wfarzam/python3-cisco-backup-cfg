# Backup Cisco Devices configuration to remote FTP Server and send E-mail Alert.
Python3 script to backup Cisco Routers and Switches both IOS-XE, Nexus NXOS and IOS devices configurations to a remote FTP Server and also get an email alert for Successfull or failed backup of the each device.
This script is made for the Network Environments where Organizations dose not have any centralized management tool like Cisco DNA or any other tool to backup their Cisco devices configuration.

Please change the Device IP addresses, usersname, passwords, ftp server and email settings according to your environment settings and how things are setup.

Intial, steps are as following
## 1. First, lets make the python3 envirnment ready.
```sh
pip install netmiko    
pip install logging
pip install datetime
pip install colorama
```

## 2. All the python import commands
```py
import os
import logging
import getpass
from ftplib import FTP, error_perm
from netmiko import ConnectHandler
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor, as_completed
import time
from colorama import Fore, Style, init
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.utils import formataddr
```

## Initialize colorama
```py
init(autoreset=True)
```

## Your Remote FTP Server Configurations details
```py
ftp_servers = ["192.168.11.123", "192.168.22.123"]
ftp_username = "ftpusername"
ftp_password = "ftpuserid password"
```

## Email Alert Configuration details to prepare the script to send email.
```py
email_sender = 'DevicesBackup@WhatEverYourDomainNameIs or Your desired email'
email_receivers = ['NetworkTeam@@WhatEverYourDomainNameIs or Your desired email distritbuion' ]
email_subject = 'Device Backup Status'
email_smtp_server = 'Your Organization Email Server SMTP Address'
email_smtp_port = 587
email_username = 'username which can be used to be able to send email'
email_password = 'email password'
```

## Device List
in this example we use the following devices IP as our test devices and make sure SSH is enabled and can be access with the username and password.
```py
devices = [
    {"ip": "192.168.10.200", "vendor": "cisco_xe"},
    {"ip": "192.168.10.201", "vendor": "cisco_nxos"},

]
```
## Device Credentials
```py
device_username = "your ssh user id"
device_password = "your ssh password"
```
## The rest of the script

```py
# Set up logging
logging.basicConfig(filename='backup_log.txt', level=logging.INFO,
                    format='%(asctime)s - %(levelname)s - %(message)s')

# Function to log and print messages with color coding
def log_and_print(message, level='info'):
    if level == 'info':
        logging.info(message)
        print(Fore.YELLOW + message)
    elif level == 'success':
        logging.info(message)
        print(Fore.GREEN + Style.BRIGHT + message)
    elif level == 'warning':
        logging.warning(message)
        print(Fore.YELLOW + message)
    elif level == 'error':
        logging.error(message)
        print(Fore.RED + Style.BRIGHT + message)

# Function to send email
def send_email(subject, body):
    try:
        msg = MIMEMultipart()
        msg['From'] = formataddr(('Network Operations Backup Team', email_sender))
        msg['To'] = ", ".join(email_receivers)
        msg['Subject'] = subject

        msg.attach(MIMEText(body, 'plain'))

        server = smtplib.SMTP(email_smtp_server, email_smtp_port)
        server.set_debuglevel(1)  # Enable debug output for the SMTP connection
        server.ehlo()
        server.starttls()
        server.ehlo()
        server.login(email_username, email_password)
        text = msg.as_string()
        server.sendmail(email_sender, email_receivers, text)
        server.quit()

        log_and_print("Email sent successfully.", level='success')
    except smtplib.SMTPException as e:
        log_and_print(f"Failed to send email: {e}", level='error')
    except Exception as e:
        log_and_print(f"An error occurred while sending email: {e}", level='error')

# Function to fetch configuration and hostname
def fetch_device_config(device):
    try:
        # Connect to the device
        connection = ConnectHandler(
            device_type=device["vendor"],
            host=device["ip"],
            username=device_username,
            password=device_password,
        )
        log_and_print(f"Connected to {device['ip']}", level='info')

        # Get hostname
        if device["vendor"] == "cisco_xe":
            hostname_command = "show running-config | include hostname | exce"
        elif device["vendor"] == "cisco_nxos":
            hostname_command = 'show run | sec hostname | exclude "logging origin-id"'
        else:
            raise ValueError("Unsupported vendor")

        hostname_output = connection.send_command(hostname_command)
        hostname = (
            hostname_output.split()[-1] if "hostname" in hostname_output else "unknown"
        )

        # Fetch configuration
        config_command = "show running-config"
        config_output = connection.send_command(config_command)

        connection.disconnect()

        # Return hostname and configuration
        return hostname, config_output

    except Exception as e:
        log_and_print(f"Failed to fetch config from {device['ip']}: {e}", level='error')
        return None, None

# Function to upload to FTP with retries and exponential backoff
def upload_to_ftp(hostname, config, backup_filename):
    success_count = 0
    for ftp_host in ftp_servers:
        for attempt in range(3):  # Retry 3 times
            try:
                ftp = FTP(ftp_host, timeout=60)
                ftp.login(user=ftp_username, passwd=ftp_password)
                log_and_print(f"Connected to FTP server: {ftp_host}", level='info')

                # Create a folder for the hostname
                remote_directory = f"/{hostname}"
                try:
                    ftp.cwd(remote_directory)
                except error_perm:
                    log_and_print(f"Creating directory {remote_directory} on {ftp_host}", level='info')
                    ftp.mkd(remote_directory)
                    ftp.cwd(remote_directory)

                # Save configuration to a local file
                with open(backup_filename, "wb") as file:
                    file.write(config.encode("utf-8"))

                # Upload the configuration file
                with open(backup_filename, "rb") as file:
                    ftp.storbinary(f"STOR {backup_filename}", file)
                    log_and_print(f"Uploaded {backup_filename} to FTP server {ftp_host}", level='success')

                # Verify if the file was uploaded successfully
                ftp.cwd(remote_directory)
                files = ftp.nlst()
                if backup_filename in files:
                    log_and_print(f"File {backup_filename} successfully uploaded to {ftp_host}", level='success')
                    success_count += 1
                    ftp.quit()
                    break  # Exit retry loop on success
                else:
                    log_and_print(f"File {backup_filename} upload verification failed on {ftp_host}", level='warning')
                    ftp.quit()
                    time.sleep(2 ** attempt)  # Exponential backoff

            except Exception as e:
                log_and_print(f"Attempt {attempt + 1} failed for {ftp_host}: {e}", level='error')
                time.sleep(2 ** attempt)  # Exponential backoff

    if success_count == len(ftp_servers):
        return True  # Successful upload to all FTP servers
    else:
        log_and_print(f"Not all FTP servers succeeded for {backup_filename}.", level='error')
        return False

# Main function to process devices in parallel
def backup_devices():
    results = []
    with ThreadPoolExecutor(max_workers=len(devices)) as executor:
        futures = []
        for device in devices:
            futures.append(executor.submit(process_device, device, results))

        for future in as_completed(futures):
            try:
                future.result()
            except Exception as e:
                log_and_print(f"Error in backup process: {e}", level='error')

    # Send email summary
    email_body = "\n".join(results)
    send_email(email_subject, email_body)

# Function to process a single device
def process_device(device, results):
    hostname, config = fetch_device_config(device)
    if not hostname or not config:
        log_and_print(f"Skipping backup for {device['ip']} due to errors.", level='error')
        results.append(f"Backup failed for {device['ip']}: Unable to fetch configuration.")
        return

    # Generate a timestamped filename
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_filename = f"{hostname}_config_{timestamp}.txt"

    # Upload the configuration to the FTP servers
    if not upload_to_ftp(hostname, config, backup_filename):
        log_and_print(f"Backup failed for {hostname} ({device['ip']})", level='error')
        results.append(f"Backup failed for {hostname} ({device['ip']})")
    else:
        results.append(f"Backup successful for {hostname} ({device['ip']})")

# Execute the script
if __name__ == "__main__":
    backup_devices()
```
