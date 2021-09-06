# Proxy_setup
Hello everyone, before you continue, ensure you meet the following requirements:
* This setup on Ubuntu server 20.04 LTS
* Puttygen on windows client

`datvo@linuxserver:~$ sudo apt-get update && sudo apt-get upragde -y`

## Config your sshd server

1. Add public key \
  `touch ~/.ssh/authorized_keys` \
  `sudo chmod 700 ~/.ssh` \
  `sudo nano ~/.ssh/authorized_keys ` \
  Copy your publickey was generated by puttygen. Save your privatekey!!!! \
  `sudo chmod 600 ~/.ssh/authorized_keys`

2. Secure SSHD \
  `sudo nano /etc/ssh/sshd_config` \
  Add the following in your sshd_config file: 

    ```
    AuthorizedKeysFile /home/datvo(must be eidt)/.ssh/authorized_keys 
    PermitRootLogin no 
    ChallengeResponseAuthentication no 
    PasswordAuthentication no 
    UsePAM no 
    PubkeyAuthentication yes 
    PermitEmptyPasswords no
    ```

3. Install fail2ban \
    `sudo apt install fail2ban` \
    `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local` \
    `sudo nano /etc/fail2ban/jail.local`  \
    Enter your setting in:
    ```
    # "bantime" is the number of seconds that a host is banned.  
    bantime  = 60d 
    # A host is banned if it has generated "maxretry" during the last "findtime" 
    # seconds. 
    findtime  = 60 
    # "maxretry" is the number of failures before a host get banned. 
     maxretry = 3 
    ```
    ```
    [sshd] 
    # To use more aggressive sshd modes set filter parameter "mode" in jail.local: 
    # normal (default), ddos, extra or aggressive (combines all). 
    # See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details. 
    #mode   = normal 
    enabled = true 
    port    = ssh 
    logpath = %(sshd_log)s 
    backend = %(sshd_backend)s 
    bantime = 4w 
    findtime = 1d 
    maxretry = 3
    ```
    
    `sudo systemctl restart fail2ban` \
    `sudo systemctl status fail2ban` \
    
    The output wiil look like this: 
    ```
    ● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2021-09-06 14:15:51 UTC; 2s ago
       Docs: man:fail2ban(1)
    Process: 4882 ExecStartPre=/bin/mkdir -p /run/fail2ban (code=exited, status=0/SUCCESS)
   Main PID: 4883 (f2b/server)
      Tasks: 7 (limit: 1090)
     Memory: 14.0M
     CGroup: /system.slice/fail2ban.service
             └─4883 /usr/bin/python3 /usr/bin/fail2ban-server -xf start
    ```
4. Enable ufw \
    ` sudo ufw enable` \
    `sudo ufw allow ssh`  
6. Block IP using GeoIP (if you need) \
    `sudo apt install snapd` \
    `sudo snap install geoip-lookup` \
    Go to: \
    `touch /usr/local/bin/sshfilter.sh` \
    `sudo nano /usr/local/bin/sshfilter.sh` \
    Copy this script and edit your country: \ 
    
    ```
    #!/bin/bash

    # UPPERCASE space-separated country codes to ACCEPT
    ALLOW_COUNTRIES="VN HK"

    if [ $# -ne 1 ]; then
      echo "Usage:  `basename $0` <ip>" 1>&2
      exit 0 # return true in case of config issue
    fi

    COUNTRY=`/usr/bin/geoiplookup $1 | awk -F ": " '{ print $2 }' | awk -F "," '{ print $1 }' | head -n 1`

    [[ $COUNTRY = "IP Address not found" || $ALLOW_COUNTRIES =~ $COUNTRY ]] && RESPONSE="ALLOW" || RESPONSE="DENY"

    if [ $RESPONSE = "ALLOW" ]
    then
      exit 0
    else
      logger "$RESPONSE sshd connection from $1 ($COUNTRY)"
      exit 1
    fi

    ```
    Save file: \ 
    `sudo chmod +x /usr/local/bin/sshfilter.sh` \
    Apply SSH estrictions using TCP wrappers. \
    `sudo nano /etc/hosts.deny`
    `sshd: ALL`

    Now edit /etc/hosts.allow \
    `sshd: ALL: aclexec /usr/local/bin/sshfilter.sh %a`
    You can see all accessing SSH from an IP address \
    `sudo cat /var/log/syslog | grep 'sshd'`

  ## Config squid server
  1. Install squid \
    `sudo apt install squid`
  2. Configuring Squid server
  
    
    # And finally deny all other access to this proxy
    http_access allow all

    # Squid normally listens to port 3128
    http_port 8888

    cache_mem 256 MB
    # Uncomment and adjust the following to add a disk cache directory.
    cache_dir ufs /var/spool/squid 10000 16 256

    # Leave coredumps in the first cache dir
    coredump_dir /var/spool/squid
    

    if you want to anonymize 
    ```
    forwarded_for off
    via off

    request_header_access Authorization allow all
    request_header_access Proxy-Authorization allow all
    request_header_access Cache-Control allow all
    request_header_access Content-Length allow all
    request_header_access Content-Type allow all
    request_header_access Date allow all
    request_header_access Host allow all
    request_header_access If-Modified-Since allow all
    request_header_access Pragma allow all
    request_header_access Accept allow all
    request_header_access Accept-Charset allow all
    request_header_access Accept-Encoding allow all
    request_header_access Accept-Language allow all
    request_header_access Connection allow all
    request_header_access All deny all

    
3.After config: 
  ` sudo systemctl restart squid` \
  ` sudo systemctl status squid ` \
  ` sudo ufw allow 8888` \
  `sudo status ufw numbered` \
  ```
  Status: active

  To                         Action      From
  --                         ------      ----
  22/tcp                     ALLOW       Anywhere
  8888                       ALLOW       Anywhere
  19999/tcp                  ALLOW       Anywhere
  1194                       DENY        Anywhere
  1994                       DENY        Anywhere
  22/tcp (v6)                ALLOW       Anywhere (v6)
  8888 (v6)                  ALLOW       Anywhere (v6)
  19999/tcp (v6)             ALLOW       Anywhere (v6)
  1194 (v6)                  DENY        Anywhere (v6)
  1994 (v6)                  DENY        Anywhere (v6)

  ```
## Install monitoring:
1. [glances] (https://github.com/nicolargo/glances)
2. [netdata] (https://github.com/netdata/netdata)
  
  
  
  
  
  
  

