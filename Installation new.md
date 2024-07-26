# Guide to Installing an Open Source Mail Server with Postfix and Dovecot

This guide will walk you through the installation and configuration of an open source mail server using Postfix and Dovecot on a Linux-based system. We'll cover the steps necessary to set up the server, configure users and secure the server with TLS.

## Port Setting

1. **Open port 25 (SMTP):**
   ```bash
   sudo ufw allow 25
   ```

2. **Verify port 25 is open:**
   ```bash
   nmap localhost
   ```

## User Creation

1. **Create users:**
   ```bash
   sudo useradd -m user1 -s /sbin/nologin
   sudo useradd -m user2 -s /sbin/nologin
   ```

2. **Set passwords for users:**
   ```bash
   sudo passwd user1
   sudo passwd user2
   ```

## Set Hostname

1. **Set a Fully Qualified Domain Name (FQDN):**
   ```bash
   sudo hostnamectl set-hostname mailserver.com
   ```

2. **Verify the hostname:**
   ```bash
   hostname
   ```

3. **Edit the `/etc/hosts` file:**
   ```bash
   sudo nano /etc/hosts
   ```

   Add your FQDN:
   ```plaintext
   127.0.0.1 localhost mailserver.com mailserver
   ```

4. **Restart the resolved service:**
   ```bash
   sudo systemctl restart systemd-resolved.service
   ```

5. **Verify hostname with nslookup:**
   ```bash
   nslookup mailserver.com
   ```

## Install Postfix

1. **Debian-based:**
   ```bash
   sudo apt install postfix
   ```

2. **RPM-based:**
   ```bash
   sudo dnf install postfix
   ```

3. **Configure Postfix:**
   ```bash
   sudo nano /etc/postfix/main.cf
   ```

   Edit the configuration:
   ```plaintext
   myhostname = mailserver.com
   mydomain = cmail.com
   inet_interfaces = all
   mydestination = $myhostname, localhost.$mydomain, localhost
   smtpd_sasl_auth_enable = no
   home_mailbox = Maildir/
   ```

4. **Optional: Create virtual aliases:**
   ```bash
   sudo nano /etc/postfix/virtual
   ```

   Add mappings:
   ```plaintext
   user1@mailserver.com  user1
   user2@mailserver.com  user2
   support@example.com  /path/to/script.sh
   ```

5. **Apply changes and restart Postfix:**
   ```bash
   sudo postmap /etc/postfix/virtual
   sudo systemctl restart postfix
   ```

6. **Test Postfix configuration:**
   ```bash
   sudo apt install mailutils # Debian-based
   sudo dnf install mailx     # RPM-based

   echo "This is a test email" | sendmail user1
   cat /var/mail/user1
   sudo tail -f /var/log/mail.log
   ```

## Install Dovecot

1. **Debian-based:**
   ```bash
   sudo apt install dovecot-core dovecot-imapd
   ```

2. **RPM-based:**
   ```bash
   sudo dnf install dovecot
   ```

3. **Configure Dovecot:**
   ```bash
   sudo nano /etc/dovecot/dovecot.conf
   ```

   Edit the configuration:
   ```plaintext
   protocols = imap
   listen = *
   auth_mechanisms = plain login
   disable_plaintext_auth = no
   mail_location = Maildir:~/Maildir
   ```

4. **Test Dovecot configuration:**
   ```bash
   sudo dovecot -n
   ```

5. **Start and enable Dovecot:**
   ```bash
   sudo systemctl start dovecot
   sudo systemctl enable dovecot
   ```

## Configure Firewall

1. **Debian-based (UFW):**
   ```bash
   sudo apt update
   sudo apt install ufw
   sudo ufw enable
   sudo ufw allow 80/tcp
   sudo ufw allow smtp
   sudo ufw allow 587/tcp
   sudo ufw allow 465/tcp
   sudo ufw allow imap
   sudo ufw allow imaps
   sudo ufw allow pop3
   sudo ufw allow pop3s
   sudo ufw reload
   ```

2. **RPM-based (Firewalld):**
   ```bash
   sudo systemctl start firewalld
   sudo firewall-cmd --permanent --add-service=smtp
   sudo firewall-cmd --permanent --add-service=smtp-submission
   sudo firewall-cmd --permanent --add-service=smtps
   sudo firewall-cmd --permanent --add-service=imap
   sudo firewall-cmd --permanent --add-service=imaps
   sudo firewall-cmd --permanent --add-service=pop3
   sudo firewall-cmd --permanent --add-service=pop3s
   sudo firewall-cmd --reload
   ```

## Configure Thunderbird

1. **User Information:**
   - Username: `bunti`
   - Email: `bunti@mailserver.com`

2. **Incoming Server:**
   - Protocol: `IMAP`
   - Hostname: `mailserver.com`
   - Port: `143`
   - Connection Security: `Auto`
   - Authentication Method: `Auto`
   - User: `bunti@mailserver.com`

3. **Outgoing Server:**
   - Hostname: `mailserver.com`
   - Port: `25`
   - Connection Security: `Auto`
   - Authentication Method: `Auto`
   - User: `bunti@mailserver.com`

## Backup Mail

1. **Backup using rsync:**
   ```bash
   rsync -av --exclude='admin-user/' /home/ /home/admin-user/mail-server-backup/
   ```

## SSL Certificate and Key

1. **Create directories:**
   ```bash
   sudo mkdir -p /etc/pki/tls/certs
   sudo mkdir /etc/pki/tls/private
   ```

2. **Generate self-signed certificate and key:**
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/pki/tls/private/mailserver.key -out /etc/pki/tls/certs/mailserver.crt
   ```

3. **Verify certificate:**
   ```bash
   openssl x509 -in /etc/pki/tls/certs/mailserver.crt -text -noout
   ```

4. **Configure Postfix for TLS:**
   ```bash
   sudo nano /etc/postfix/main.cf
   ```

   Add TLS settings:
   ```plaintext
   smtpd_tls_cert_file = /etc/pki/tls/certs/mailserver.crt
   smtpd_tls_key_file = /etc/pki/tls/private/mailserver.key
   smtpd_tls_security_level = may
   smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
   smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
   smtpd_tls_loglevel = 1
   smtpd_tls_received_header = yes
   smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
   smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
   ```

5. **Restart Postfix:**
   ```bash
   sudo systemctl restart postfix
   ```

6. **Test TLS certificates:**
   ```bash
   openssl s_client -connect hostname:25 -starttls smtp
   ```

## Implement SASL

1. **Configure Postfix for SASL:**
   ```bash
   sudo nano /etc/postfix/main.cf
   ```

   Add settings:
   ```plaintext
   smtpd_sasl_auth_enable = yes
   smtpd_sasl_path = private/auth
   smtpd_sasl_type = dovecot
   smtpd_sasl_local_domain = $myhostname
   smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination
   ```

2. **Configure Dovecot for SASL:**
   ```bash
   sudo nano /etc/dovecot/conf.d/10-auth.conf
   ```

   Edit settings:
   ```plaintext
   disable_plaintext_auth = yes
   auth_mechanisms = plain login
   ```

3. **Configure Dovecot SASL listener:**
   ```bash
   sudo nano /etc/dovecot/conf.d/10-master.conf
   ```

   Add configuration:
   ```plaintext
   service auth {
     unix_listener /var/spool/postfix/private/auth {
       mode = 0666
       user = postfix
       group = postfix
     }
   }
   ```

4. **Restart services:**
   ```bash
   sudo systemctl restart postfix
   sudo systemctl restart dovecot
   ```

5. **Test SASL authentication:**
   - Use a mail client like Thunderbird or Outlook with SMTP authentication.

## Create SASL Password
