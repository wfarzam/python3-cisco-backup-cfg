# python3-cisco-backup-cfg
Backup Cisco Routers and Switches both IOS-XE, Nexus NXOS and IOS devices configurations to a remote SFTP Server and also get an email alert for Successfull or failed backup of the each device.

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
