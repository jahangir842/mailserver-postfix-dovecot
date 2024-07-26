# Guide to Installing an Open Source Mail Server with Postfix and Dovecot

This comprehensive guide will walk you through the installation and configuration of an open-source mail server using Postfix and Dovecot on a Linux-based system. We will cover the essential steps needed to set up the server, create users, secure the server with TLS, and integrate it with Mattermost.

## Open and Verify Port 25

1. **Open port 25 (SMTP)**: This step ensures that the SMTP service can communicate through port 25, which is the default port for sending emails.
   ```bash
   sudo ufw allow 25
   ```

2. **Verify port 25 is open**: Use `nmap` to scan your local system and confirm that port 25 is open.
   ```bash
   nmap localhost
   ```

## Create Mail Users

1. **Create users**: Add users to the system who will be receiving emails. The `-m` flag creates the user's home directory, and `-s /sbin/nologin` ensures they cannot log in interactively.
   ```bash
   sudo useradd -m user1 -s /sbin/nologin
   sudo useradd -m user2 -s /sbin/nologin
   ```

2. **Set passwords for users**: Set a password for each user.
   ```bash
   sudo passwd user1
   sudo passwd user2
   ```

## Set the Hostname

1. **Set a Fully Qualified Domain Name (FQDN)**: This command sets the system's hostname, which is crucial for mail server identification.
   ```bash
   sudo hostnamectl set-hostname mailserver.com
   ```

2. **Verify the hostname**: Check the hostname to ensure it has been set correctly.
   ```bash
   hostname
   ```

3. **Edit the `/etc/hosts` file**: Map your FQDN to the local IP address.
   ```bash
   sudo nano /etc/hosts
   ```

   Add your FQDN:
   ```plaintext
   127.0.0.1 localhost mailserver.com mailserver
   ```

4. **Restart the resolved service**: This ensures that the changes to the hostname are applied.
   ```bash
   sudo systemctl restart systemd-resolved.service
   ```

5. **Verify hostname with nslookup**: Check the DNS resolution for your hostname.
   ```bash
   nslookup mailserver.com
   ```

## Install and Configure Postfix

1. **Install Postfix**: Postfix is a powerful and flexible mail transfer agent (MTA).
   - **Debian-based systems**:
     ```bash
     sudo apt install postfix
     ```
   - **RPM-based systems**:
     ```bash
     sudo dnf install postfix
     ```

2. **Configure Postfix**: Edit the main Postfix configuration file to set up the basic parameters for your mail server.
   ```bash
   sudo nano /etc/postfix/main.cf
   ```

   Example configuration:
   ```plaintext
   myhostname = mailserver.com
   mydomain = cmail.com
   inet_interfaces = all
   mydestination = $myhostname, localhost.$mydomain, localhost
   smtpd_sasl_auth_enable = no
   home_mailbox = Maildir/
   ```

3. **Optional: Create virtual aliases**: Map email addresses to local users or scripts.
   ```bash
   sudo nano /etc/postfix/virtual
   ```

   Add mappings:
   ```plaintext
   user1@mailserver.com  user1
   user2@mailserver.com  user2
   support@example.com  /path/to/script.sh
   ```

4. **Apply changes and restart Postfix**: Rebuild the virtual aliases database and restart Postfix to apply changes.
   ```bash
   sudo postmap /etc/postfix/virtual
   sudo systemctl restart postfix
   ```

5. **Test Postfix configuration**: Send a test email to ensure that Postfix is working correctly.
   ```bash
   sudo apt install mailutils  # Debian-based
   sudo dnf install mailx      # RPM-based

   echo "This is a test email" | sendmail user1
   cat /var/mail/user1
   sudo tail -f /var/log/mail.log
   ```

## Install and Configure Dovecot

1. **Install Dovecot**: Dovecot is a popular IMAP and POP3 server for Unix-like operating systems.
   - **Debian-based systems**:
     ```bash
     sudo apt install dovecot-core dovecot-imapd
     ```
   - **RPM-based systems**:
     ```bash
     sudo dnf install dovecot
     ```

2. **Configure Dovecot Main Configuration File**: Edit the main Dovecot configuration file to set up IMAP and authentication mechanisms.
   ```bash
   sudo nano /etc/dovecot/dovecot.conf
   ```

   Example configuration:
   ```plaintext
   protocols = imap
   listen = *
   auth_mechanisms = plain login
   disable_plaintext_auth = no
   mail_location = maildir:~/Maildir
   ```


2. **Configure Dovecot Other Configuration File**: Edit the other Dovecot configuration files.
   ```bash
   sudo nano /etc/dovecot/conf.d/10-mail.conf
   ```

   Example configuration option for mail location as already configured in main configuration file:
   ```plaintext
   mail_location = maildir:~/Maildir
   ```
3. **Test Dovecot configuration**: Verify that the Dovecot configuration is correct.
   ```bash
   sudo dovecot -n
   ```

4. **Start and enable Dovecot**: Ensure Dovecot starts on boot and is currently running.
   ```bash
   sudo systemctl start dovecot
   sudo systemctl enable dovecot
   ```

## Configure Firewall

1. **Debian-based (UFW)**: Configure UFW to allow necessary ports for mail services.
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

2. **RPM-based (Firewalld)**: Configure Firewalld to allow necessary ports for mail services.
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

## Configure Thunderbird Mail Client

1. **User Information**:
   - Username: `bunti`
   - Email: `bunti@mailserver.com`

2. **Incoming Server Configuration**:
   - Protocol: `IMAP`
   - Hostname: `mailserver.com`
   - Port: `143`
   - Connection Security: `Auto`
   - Authentication Method: `Auto`
   - User: `bunti@mailserver.com`

3. **Outgoing Server Configuration**:
   - Hostname: `mailserver.com`
   - Port: `25`
   - Connection Security: `Auto`
   - Authentication Method: `Auto`
   - User: `bunti@mailserver.com`

## Backup Mail

1. **Backup using rsync**: Create a backup of the mail directories using `rsync`.
   ```bash
   rsync -av --exclude='admin-user/' /home/ /home/admin-user/mail-server-backup/
   ```

## SSL Certificate and Key

1. **Create directories**: Ensure the necessary directories for storing SSL certificates and keys exist.
   ```bash
   sudo mkdir -p /etc/pki/tls/certs
   sudo mkdir /etc/pki/tls/private
   ```

2. **Generate self-signed certificate and key**: Use OpenSSL to generate a self-signed certificate and key for your mail server.
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/pki/tls/private/mailserver.key -out /etc/pki/tls/certs/mailserver.crt
   ```

3. **Verify the certificate**: Check the details of the generated certificate.
   ```bash
   openssl x509 -in /etc/pki/tls/certs/mailserver.crt -text -noout
   ```

4. **Configure Postfix for TLS**: Edit the Postfix configuration file to enable TLS encryption.
   ```bash
   sudo nano /etc/postfix/main.cf
   ```

   Add the following TLS settings:
   ```plaintext
   smtpd_tls_cert_file = /etc/pki/tls/certs/mailserver.crt
   smtpd_tls_key_file = /etc/pki/tls/private/mailserver.key
   smtpd_tls_security_level = may
   smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
   smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
   smtpd_tls_loglevel = 1
   smtpd_tls_received_header = yes
   smtpd_tls_session_cache_database = btree:${data

_directory}/smtpd_scache
   smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
   tls_random_source = dev:/dev/urandom
   ```

5. **Configure Dovecot for TLS**: Edit the Dovecot configuration file to enable TLS encryption.
   ```bash
   sudo nano /etc/dovecot/conf.d/10-ssl.conf
   ```

   Add the following TLS settings:
   ```plaintext
   ssl = yes
   ssl_cert = </etc/pki/tls/certs/mailserver.crt
   ssl_key = </etc/pki/tls/private/mailserver.key
   ```

6. **Restart Postfix and Dovecot**: Apply the changes by restarting both services.
   ```bash
   sudo systemctl restart postfix
   sudo systemctl restart dovecot
   ```

## Secure Mail with SASL Authentication

1. **Install Cyrus SASL**: Ensure that Cyrus SASL is installed on your system.
   - **Debian-based systems**:
     ```bash
     sudo apt install sasl2-bin
     ```
   - **RPM-based systems**:
     ```bash
     sudo dnf install cyrus-sasl
     ```

2. **Configure Postfix for SASL**: Edit the Postfix configuration file to enable SASL authentication.
   ```bash
   sudo nano /etc/postfix/main.cf
   ```

   Add the following SASL settings:
   ```plaintext
   smtpd_sasl_auth_enable = yes
   smtpd_sasl_security_options = noanonymous
   smtpd_sasl_local_domain = $myhostname
   broken_sasl_auth_clients = yes
   smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination
   ```

3. **Create SASL password file**: Add users to the SASL password file.
   ```bash
   sudo saslpasswd2 -c -u yourdomain.com username
   ```

4. **Set permissions for SASL password file**: Ensure the correct permissions are set for the SASL password file.
   ```bash
   sudo chown postfix:saslauth /etc/sasldb2
   sudo chmod 640 /etc/sasldb2
   ```

5. **Restart Postfix service**: Apply the changes by restarting the Postfix service.
   ```bash
   sudo systemctl restart postfix
   ```

## Integrate Mail with Mattermost

1. **Login to Mattermost system console**: Access the Mattermost system console as an administrator.

2. **Set up email server**: Navigate to "System Console" > "Email" > "SMTP" and configure the email server settings:
   - **SMTP Server**: `mailserver.com`
   - **SMTP Port**: `25`
   - **SMTP Username and Password**
   - **Connection Security**: `TLS/STARTTLS`
   - **Sender Email Address**: `noreply@yourdomain.com`

3. **Optional: Configure LDAP or SSO**: Set up LDAP or Single Sign-On (SSO) in the Mattermost system console if needed.

4. **Optional: Enable email verification**: Navigate to "Email" > "Email Settings" and enable email verification for new user registrations.

5. **Test email configuration**: Send a test email from the Mattermost system console to ensure the email server is configured correctly.

6. **User registration on Mattermost**: Enable and verify email verification to allow users to register and receive confirmation emails on Mattermost.

This guide covers the essential steps to set up a robust mail server using Postfix and Dovecot. Follow these instructions to ensure a functional and secure mail service.
