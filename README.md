# Backup Cisco Devices configuration to remote FTP Server and send E-mail Alert.
Python3 script to backup Cisco Routers and Switches both IOS-XE, Nexus NXOS and IOS devices configurations to a remote FTP Server and also get an email alert for Successfull or failed backup of the each device.
This script is made for the Network Environments where Organizations dose not have any centralized management tool like Cisco DNA or any other tool to backup their Cisco devices configuration.

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
