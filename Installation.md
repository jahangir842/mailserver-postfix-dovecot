###Mail server 
				Port Setting

open port 25 (smtp)
verify it by:
	nmap localhost


				User Creation

Use the useradd command to create the users
	sudo useradd -m user1 -s /sbin/nologin
	sudo useradd -m user2 -s /sbin/nologin
The -m flag creates the user's home directory if it doesn't exist.
Set a password for users:
	sudo passwd user1
	sudo passwd user2


				Set host name

set a FQDN  
Note: An FQDN includes the host name, domain name, and top-level domain (TLD),
providing a complete path to a specific resource on the internet.

	sudo hostnamectl set-hostname mailserver.com
	
verify hostname by:
	hostname
Edit the /etc/hosts file using a text editor like Nano or Vim.

Add your FQDN after localhost. For example:

	sudo nano /etc/hosts
	127.0.0.1 localhost mailserver.com mailserver

Replace "myhost.example.com" with your desired FQDN, and "myhost" with your hostname.
Save the changes and exit the text editor.


	sudo systemctl restart systemd-resolved.service

verify:
	nslookup mailserver.com


				Install postfix

Debian based:
	sudo apt install postfix
RPM based:
	sudo dnf install postfix

During the installation (for debian only), a configuration wizard will appear. Choose the options: Local only:
 
Enter your desired mail domain (e.g., mailserver.com).

Edit the Postfix configuration file /etc/postfix/main.cf using a text editor such as nano:

	sudo nano /etc/postfix/main.cf
	
verify the details as below:
				# Set the hostname of your machine
				myhostname = mailserver.com

				# Set the domain to which mail will be sent
				mydomain = cmail.com

				# Ensure only local mail is delivered
				# inet_interfaces = loopback-only 
				inet_interfaces = all


				# Set the local recipient domains
				mydestination = $myhostname, localhost.$mydomain, localhost

				# Disable SMTP authentication
				smtpd_sasl_auth_enable = no
				
				# Set the mailbox format to Maildir
				home_mailbox = Maildir/
				

				

Optional: You can create virtual aliases to forward emails sent to specific addresses to local users or scripts. Edit the virtual alias file using:

	sudo nano /etc/postfix/virtual

Add your virtual alias mappings in the format alias@example.com destination:

				# Example virtual aliases
				user1@mailserver.com  user1
				user2@mailserver.com  user2
				support@example.com  /path/to/script.sh

After modifying configuration files, apply the changes and restart Postfix:
	sudo postmap /etc/postfix/virtual
	sudo systemctl restart postfix
	
To test if Postfix is configured correctly and sending/receiving mail locally, you can use the sendmail command. For example:

Deb based:	
	sudo apt install mailutils
RPM based:
	sudo dnf install mailx
	
test email:		
	echo "This is a test email" | sendmail user1
	
Replace user1 with the local recipient configured in virtual aliases. Check the recipient's mailbox to verify if the email was delivered.
	cat /var/mail/user1

You can also check the mail log for any errors or to monitor the email delivery process:
	sudo tail -f /var/log/mail.log
	
Optional: To test an email server using Telnet, you can follow these steps.
	telnet smtp.server.com 25
	EHLO example.com

	MAIL FROM: <sender@example.com>
	RCPT TO: <recipient@example.com>
	DATA
	Subject: Test Email

	This is a test email sent using Telnet.
	.
	
	QUIT



				Install dovecot

Debian based:
	sudo apt install dovecot-core dovecot-imapd
RPM based:
	sudo dnf install dovecot

Open the Dovecot configuration file (/etc/dovecot/dovecot.conf) using a text editor like Nano or Vim:
	sudo nano /etc/dovecot/dovecot.conf
	
Uncomment or add the following lines to enable the IMAP protocol and set it to listen only on localhost (127.0.0.1):
	protocols = imap
	#listen = 127.0.0.1
	listen = *

By default, Dovecot uses system users for authentication. Ensure that the following lines are present and not commented out:

	auth_mechanisms = plain login
	disable_plaintext_auth = no

Set the location where Dovecot should look for mail. For local testing, you can use a directory like /var/mail/%u where %u represents the username:


Use one of the following option as required

	#mail_location = Maildir:/var/mail/%u   # this saves the incomming mail in /var/mail 
	mail_location = Maildir:~/Maildir       # this saves the incomming mail in /home/<user>   
	
#in Maildir:~/Maildir .....>   first Maildir shows sent item and trash,,,, second Maildir shows inbox
	

Before starting Dovecot, it's crucial to test the configuration for syntax errors. Use the following command:

	sudo dovecot -n

Start the Dovecot service and enable it to start on boot:

	sudo systemctl start dovecot
	sudo systemctl enable dovecot



				Configure Firewall

1. for Debian ufw:
	sudo ufw enable

# Install and enable UFW
	sudo apt update
	sudo apt install ufw
	sudo ufw enable

# Allow HTTP Service
	sudo ufw allow 80/tcp

# Open Ports for Postfix
	sudo ufw allow smtp
	sudo ufw allow 587/tcp  # SMTP Submission
	sudo ufw allow 465/tcp  # SMTPS (optional)

# Open Ports for Dovecot
	sudo ufw allow imap
	sudo ufw allow imaps
	sudo ufw allow pop3
	sudo ufw allow pop3s

# Reload UFW to apply changes
	sudo ufw reload




2. for RPM based Firewalld:
	sudo systemctl status firewalld
	sudo systemctl start firewalld

	sudo firewall-cmd --get-services

Add a New Service (Optional): e.g http
	sudo firewall-cmd --permanent --new-service=http
	sudo firewall-cmd --permanent --service=http --set-description="HTTP Service"
	sudo firewall-cmd --permanent --service=http --add-port=80/tcp
	sudo firewall-cmd --reload

Open Ports for Postfix:

	sudo firewall-cmd --permanent --add-service=smtp
	sudo firewall-cmd --permanent --add-service=smtp-submission
	sudo firewall-cmd --permanent --add-service=smtps   # Optional, if using SMTPS

Open Ports for Dovecot:

	sudo firewall-cmd --permanent --add-service=imap
	sudo firewall-cmd --permanent --add-service=imaps
	sudo firewall-cmd --permanent --add-service=pop3
	sudo firewall-cmd --permanent --add-service=pop3s


	sudo firewall-cmd --reload


				Configure Thunderbird

username: 	bunti
email: 		bunti@mailserver.com

Incomming Server: 
Protocol	IMAP
Hostname	mailserver.com
Port		143
Connection Sec	Auto
Auth method	Auto 
user		bunti@mailserver.com

Incomming Server: 
Hostname	mailserver.com
Port		25
Connection Sec	Auto
Auth method	Auto 
user		bunti@mailserver.com


				Backup of Mail


rsync -av --exclude='admin-user/' /home/ /home/admin-user/mail-server-backup/



			SSL Certificate and key


First, create the necessary directories if they don't already exist:

	sudo mkdir -p /etc/pki/tls/certs
	sudo mkdir /etc/pki/tls/private

Next, generate a self-signed SSL/TLS certificate and key using OpenSSL:

	sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/pki/tls/private/mailserver.key -out /etc/pki/tls/certs/mailserver.crt

verify certificate:

	openssl x509 -in /etc/pki/tls/certs/mailserver.crt -text -noout
	
Configure Postfix for TLS:
Edit the Postfix configuration file (/etc/postfix/main.cf) and add or modify the TLS settings:

	smtpd_tls_cert_file = /etc/pki/tls/certs/mailserver.crt
	smtpd_tls_key_file = /etc/pki/tls/private/mailserver.key
	smtpd_tls_security_level = may
	smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
	smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
	smtpd_tls_loglevel = 1
	smtpd_tls_received_header = yes
	smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
	smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

Restart Postfix:

	sudo systemctl restart postfix
	
To test TLS certificates 
	
	openssl s_client -connect hostname:25 -starttls smtp
	
	

				implement SASL


To implement SASL (Simple Authentication and Security Layer) in Postfix and Dovecot on Ubuntu,
you need to configure both services to work together for authentication. Here's a step-by-step guide:

Configure Postfix:

sudo nano /etc/postfix/main.cf

Add or modify the following settings to enable SASL authentication:

    smtpd_sasl_auth_enable = yes
    smtpd_sasl_path = private/auth
    smtpd_sasl_type = dovecot
    smtpd_sasl_local_domain = $myhostname
    smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination

Save and close the file.

Configure Dovecot:

Edit the Dovecot authentication configuration file:

	sudo nano /etc/dovecot/conf.d/10-auth.conf

Ensure the following settings are enabled or set:

    disable_plaintext_auth = yes
    auth_mechanisms = plain login

Save and close the file.

Configure Dovecot SASL:

Edit the Dovecot SASL configuration file:

	sudo nano /etc/dovecot/conf.d/10-master.conf

Add or modify the following service configuration:

    service auth {
      unix_listener /var/spool/postfix/private/auth {
        mode = 0666
        user = postfix
        group = postfix
      }
    }

Save and close the file.

Restart Services:

    sudo systemctl restart postfix
    sudo systemctl restart dovecot

Testing:

Test SASL authentication by sending an email using a client like Thunderbird or Outlook with SMTP authentication enabled.
Use a valid email account on your server and ensure that the credentials are accepted.



			   Create SASL Password


Create SASL Password File:
Create a SASL password file that will store the SMTP credentials (username and password) for authentication.
You can use the saslpasswd2 command to add entries to this file.

	sudo saslpasswd2 -c -u yourdomain.com username

Replace yourdomain.com with your actual domain and username with the desired SMTP username for authentication. You will be prompted to enter the password for the username.

Set Permissions for SASL Password File:
Ensure that the SASL password file has the correct permissions to be accessed by Postfix.

	sudo chown postfix:saslauth /etc/sasldb2
	sudo chmod 640 /etc/sasldb2

Restart Postfix Service:
After making these changes, restart the Postfix service to apply the configuration changes.

	sudo systemctl restart postfix


			Integrate Mail with Mattermost


integrate your own Postfix and Dovecot email server with Mattermost.

login to the Mattermost system console as an administrator.

Set Up Email Server:
Navigate to "System Console" > "Email" > "SMTP" in the Mattermost system console. Here, you'll configure Mattermost to use your Postfix email server for sending emails.

    SMTP Server: 			Enter the hostname or IP address of your Postfix server.
    SMTP Port: 				Typically, SMTP uses port 25. If you're using a different port, specify it here.
    SMTP Username and Password: 	If your SMTP server requires authentication, enter the credentials here.
    Connection Security: 		Choose the appropriate security option based on your server's configuration (e.g., TLS, STARTTLS).
    Sender Email Address: 		Enter the email address that Mattermost should use as the sender for emails (e.g., noreply@yourdomain.com).

Configure LDAP or SSO (Optional):
If you want to enable single sign-on (SSO) or LDAP authentication, you can configure these settings in the Mattermost system console under "Authentication" or "AD/LDAP" sections. This step is optional but can streamline user authentication if you already have LDAP or SSO set up.

Enable Email Verification (Optional):
To ensure security and verify user emails, you can enable email verification in the Mattermost system console under "Email" > "Email Settings." This step is recommended for added security.

Test Email Configuration:
After configuring the email settings, you can send a test email from the Mattermost system console to ensure that the integration with your Postfix server is working correctly.

User Registration:
Users can now register on Mattermost using their email IDs. They will receive an email with a verification link to complete the registration process.


















