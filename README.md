# Arch_AD_Connection
How to connect Arch based distribution to Windows server Active Directory


## 1. Update system
```bash
sudo pacman -Syu
```

## 2. Dependencies
```bash
sudo pacman -S samba smbclient ntp
```

## 3. Configure network
### 3.1.Network Manager 
![image](https://github.com/user-attachments/assets/25078b0a-234b-4d33-9d71-ced9e74cc4dd)

Configure ```Additional DNS servers``` with IP addresses of your DNS servers.

Configure ```Additional search domains``` with your domain.

### 3.2. Resolv.conf

File Location: ```/etc/resolv.conf```

```
nameserver 192.168.1.1
nameserver 192.168.1.2
search internal.domain.com
```
IP addresses are as example, make sure to use IP address of your domain controller.


## 4. NTP
Configure NTP to use your domain controller as NTP servers

File Location: ```/etc/ntp.conf/```

Example: 
```
server server1.internal.domain.tld
server server2.internal.domain.tld


restrict default kod limited nomodify nopeer noquery notrap
restrict 127.0.0.1
restrict ::1

driftfile /var/lib/ntp/ntpd.drift
```

## 5. Kerberos 
Configure Kerberos for ticketing 

File Location: ```/etc/krb5.conf```

Example:
```
[libdefaults]
   default_realm = INTERNAL.DOMAIN.TLD
   dns_lookup_realm = false
   dns_lookup_kdc = true
   default_ccache_name = /run/user/%{uid}/krb5cc

[realms]
   INTERNAL.DOMAIN.TLD = {
      kdc = SERVER1.INTERNAL.DOMAIN.TLD
      default_domain = INTERNAL.DOMAIN.TLD
      admin_server = SERVER1.INTERNAL.DOMAIN.TLD
   }
   INTERNAL = {
      kdc = SERVER1.INTERNAL.DOMAIN.TLD
      default_domain = INTERNAL.DOMAIN.TLD
      admin_server = SERVER1.INTERNAL.DOMAIN.TLD
   }

[domain_realm]
    .internal.domain.tld = INTERNAL.DOMAIN.TLD

[appdefaults]
    pam = {
        ticket_lifetime = 1d
        renew_lifetime = 1d
        forwardable = true
        proxiable = false
        minimum_uid = 1
    }
```

## 6. SAMBA
Create configuration file. File is not created automaticaly.

File Location: ```/etc/samba/smb.conf```

Example:
```
[global]
   workgroup = INTERNAL
   security = ADS
   realm = INTERNAL.DOMAIN.TLD

   winbind refresh tickets = Yes
   vfs objects = acl_xattr
   map acl inherit = Yes
   store dos attributes = Yes

   # Allow a single, unified keytab to store obtained Kerberos tickets
   dedicated keytab file = /etc/krb5.keytab
   kerberos method = secrets and keytab

   # Do not require that login usernames include the default domain
   winbind use default domain = yes

   # UID/GID mapping for local users
   idmap config * : backend = tdb
   idmap config * : range = 3000-7999

   # UID/GID mapping for domain users
   idmap config INTERNAL : backend = rid
   idmap config INTERNAL : range = 10000-999999

   # Template settings for users
   template shell = /bin/bash
   template homedir = /home/%U

   # Allow offline/cached credentials and ticket refresh
   winbind offline logon = yes
   winbind refresh tickets = yes
```
## 7. Join domain

```bash
sudo net ads join -U Administrator
```

## 8. Start and enable services

```bash
sudo systemctl enable smb
sudo systemctl enable nmb
sudo systemctl enable winbind
sudo systemctl start smb
sudo systemctl start nmb
sudo systemctl start winbind
```

## 9. NSS
Configure nsswitch

File Location: ```/etc/nsswitch.conf```

Example: 
```
...
passwd: files winbind mymachines systemd
group: files winbind mymachines systemd
...
```

## 10. Configure PAM 
### 10.1 Configure pam_winbind.conf

File Location: ```/etc/security/pam_winbind.conf```

Example: 
```
[Global]
   debug = no
   debug_state = no
   try_first_pass = yes
   krb5_auth = yes
   krb5_ccache_type = FILE:/run/user/%u/krb5cc
   cached_login = yes
   silent = no
   mkhomedir = yes
```

### 10.2 Configure system-auth
File Location: /etc/pam.d/system-auth

Example: 
```
#%PAM-1.0

auth       required                    pam_faillock.so      preauth
-auth      [success=3 default=ignore]  pam_systemd_home.so
auth       [success=2 default=ignore]  pam_winbind.so
auth       [success=1 default=bad]     pam_unix.so          try_first_pass nullok
auth       [default=die]               pam_faillock.so      authfail
auth       optional                    pam_permit.so
auth       required                    pam_env.so
auth       required                    pam_faillock.so      authsucc

-account   [success=2 default=ignore]  pam_systemd_home.so
account    [success=1 default=ignore]  pam_winbind.so
account    required                    pam_unix.so
account    optional                    pam_permit.so
account    required                    pam_time.so

-password  [success=2 default=ignore]  pam_systemd_home.so
password   [success=1 default=ignore]  pam_winbind.so
password   required                    pam_unix.so          try_first_pass nullok shadow sha512
password   optional                    pam_permit.so

-session   optional                    pam_systemd_home.so
session    required                    pam_mkhomedir.so skel=/etc/skel/ umask=0022
session    required                    pam_limits.so
session    required                    pam_winbind.so
session    required                    pam_unix.so
session    optional                    pam_permit.so
```


