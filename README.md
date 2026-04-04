# Fedora Mail Server Project

## 1. Description

This project is a local mail server setup using **Fedora as the server** and **Ubuntu as the client**.

The server side includes:

- Postfix
- Dovecot
- Apache
- MariaDB
- Roundcube

The client side includes:

- Ubuntu
- Thunderbird for mail testing

This project covers:

- installing and configuring the Fedora mail server
- creating local users
- enabling Maildir delivery
- setting up IMAP with Dovecot
- installing and configuring Roundcube
- applying firewall and SELinux fixes
- testing the mail service from an Ubuntu client using Thunderbird

---

## 2. Fedora Part as Server

### 2.1 Update the system

```bash
sudo dnf update -y
sudo dnf install nano -y
```

---

### 2.2 Install Postfix and mailx

```bash
sudo dnf install postfix mailx -y
```

Start and enable Postfix:

```bash
sudo systemctl enable --now postfix
sudo systemctl status postfix
```

---

### 2.3 Configure Postfix

Edit the file:

```bash
sudo nano /etc/postfix/main.cf
```

Put this configuration:

```conf
myhostname = mail.tp.local
mydomain = tp.local
myorigin = $mydomain
inet_interfaces = all
inet_protocols = all
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

home_mailbox = Maildir/

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous

smtpd_recipient_restrictions =
    permit_sasl_authenticated,
    permit_mynetworks,
    reject_unauth_destination
```

Restart Postfix:

```bash
sudo systemctl restart postfix
```

Verify SMTP port 25:

```bash
sudo ss -tlnp | grep :25
```

---

### 2.4 Create local users and test mail

Create test users:

```bash
sudo useradd user1
sudo passwd user1

sudo useradd user2
sudo passwd user2
```

Send a test mail:

```bash
echo "test from postfix" | mail -s "test1" user2
```

Check the logs:

```bash
sudo tail -n 20 /var/log/maillog
```

---

### 2.5 Install Dovecot

```bash
sudo dnf install dovecot -y
```

Start and enable Dovecot:

```bash
sudo systemctl enable --now dovecot
sudo systemctl status dovecot
```

Verify IMAP port 143:

```bash
sudo ss -tlnp | grep :143
```

---

### 2.6 Configure Dovecot

Edit the file:

```bash
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

Put this:

```conf
mail_driver = maildir
mail_path = ~/Maildir
```

Restart Dovecot:

```bash
sudo systemctl restart dovecot
```

Verify configuration:

```bash
sudo doveconf -n
```

Test IMAP locally:

```bash
sudo dnf install telnet -y
telnet localhost 143
```

---

### 2.7 Verify Maildir delivery

```bash
sudo ls /home/user2/Maildir
sudo ls /home/user2/Maildir/new
```

---

### 2.8 Install Apache, PHP, and MariaDB

```bash
sudo dnf install httpd php php-mysqlnd php-intl php-mbstring php-json php-xml mariadb-server -y
```

Start and enable services:

```bash
sudo systemctl enable --now httpd
sudo systemctl enable --now mariadb
```

Check service status:

```bash
systemctl status httpd
systemctl status mariadb
```

Allow HTTP through the firewall:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

---

### 2.9 Install Roundcube

```bash
sudo dnf install roundcubemail -y
```

Verify installation:

```bash
ls /usr/share/roundcubemail
ls /etc/roundcubemail
```

---

### 2.10 Create the Roundcube database

Open MariaDB:

```bash
sudo mysql -u root
```

Run:

```sql
CREATE DATABASE roundcube;
CREATE USER 'roundcube'@'localhost' IDENTIFIED BY 'YOUR_DB_PASSWORD';
GRANT ALL PRIVILEGES ON roundcube.* TO 'roundcube'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Import the initial tables:

```bash
sudo mysql -u roundcube -pYOUR_DB_PASSWORD roundcube < /usr/share/roundcubemail/SQL/mysql.initial.sql
```

---

### 2.11 Configure Dovecot auth socket for Postfix

Edit the file:

```bash
sudo nano /etc/dovecot/conf.d/10-master.conf
```

Make sure this exists inside `service auth`:

```conf
unix_listener /var/spool/postfix/private/auth {
  mode = 0666
  user = postfix
  group = postfix
}
```

Restart services:

```bash
sudo systemctl restart dovecot
sudo systemctl restart postfix
sudo systemctl restart httpd
```

Verify the socket:

```bash
sudo ls -l /var/spool/postfix/private/auth
```

---

### 2.12 Configure Roundcube

Edit the file:

```bash
sudo nano /etc/roundcubemail/config.inc.php
```

Put this configuration:

```php
<?php
$config = array();
$config['db_dsnw'] = 'mysql://roundcube:YOUR_DB_PASSWORD@localhost/roundcube';
$config['default_host'] = '127.0.0.1';
$config['default_port'] = 143;
$config['smtp_server'] = 'tls://127.0.0.1';
$config['smtp_port'] = 587;
$config['smtp_user'] = '%u';
$config['smtp_pass'] = '%p';
$config['smtp_timeout'] = 10;
$config['smtp_helo_host'] = 'mail.tp.local';
$config['smtp_conn_options'] = array(
  'ssl' => array(
    'verify_peer' => false,
    'verify_peer_name' => false,
    'allow_self_signed' => true,
  ),
);
$config['username_domain'] = '';
$config['mail_domain'] = 'tp.local';
$config['skin'] = 'elastic';
$config['plugins'] = array();
```

Clear cache and restart Apache:

```bash
sudo rm -f /var/lib/roundcubemail/temp/*
sudo rm -f /var/lib/roundcubemail/cache/*
sudo systemctl restart httpd
```

---

### 2.13 Optional checks

You may also check these files if needed:

```bash
sudo nano /etc/postfix/master.cf
sudo nano /etc/httpd/conf.d/roundcubemail.conf
```

Test Roundcube locally:

```bash
curl -I http://127.0.0.1/roundcubemail
```

---

### 2.14 SELinux and permissions fixes

Apply SELinux settings:

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_connect_db 1
sudo setsebool -P httpd_can_sendmail 1
sudo restorecon -Rv /usr/share/roundcubemail
sudo restorecon -Rv /etc/roundcubemail
```

Fix permissions:

```bash
sudo chown root:apache /etc/roundcubemail/config.inc.php
sudo chmod 640 /etc/roundcubemail/config.inc.php
```

If needed, re-check the database user:

```bash
sudo mysql -u root -p
SELECT user, host FROM mysql.user WHERE user='roundcube';
ALTER USER 'roundcube'@'localhost' IDENTIFIED BY 'YOUR_DB_PASSWORD';
GRANT ALL PRIVILEGES ON roundcube.* TO 'roundcube'@'localhost';
FLUSH PRIVILEGES;
exit;
```

---

### 2.15 Access Roundcube in browser

Restart services:

```bash
sudo systemctl restart httpd
sudo systemctl restart postfix
```

Then open:

```text
http://SERVER_IP/roundcubemail/
```

---

### 2.16 Open firewall ports

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=smtp
sudo firewall-cmd --permanent --add-service=imap
sudo firewall-cmd --reload
```

---

### 2.17 Create more test users

```bash
sudo useradd test1
sudo passwd test1

sudo useradd test2
sudo passwd test2

sudo useradd test3
sudo passwd test3
```

---

### 2.18 Enable simple login auth in Dovecot

Edit the file:

```bash
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

Put or verify:

```conf
auth_mechanisms = plain login
#disable_plaintext_auth = no
```

Restart Dovecot:

```bash
sudo systemctl restart dovecot
```

---

## 3. Ubuntu Part as Client

### 3.1 Install Thunderbird on Ubuntu

```bash
sudo apt update
sudo apt install thunderbird
```

---

### 3.2 Configure Thunderbird manually

Use these settings:

#### Incoming Server
- Protocol: `IMAP`
- Hostname: `SERVER_IP`
- Port: `143`
- Connection security: `STARTTLS`
- Authentication method: `Normal password`

#### Outgoing Server
- Hostname: `SERVER_IP`
- Port: `25`
- Connection security: `None`
- Authentication method: `Normal password`

#### Username
Try:

```text
test3@tp.local
```

If that does not work, try:

```text
test3
```

Then:
- click **Advanced config**
- confirm the security exception if Thunderbird asks

---

## 4. Troubleshooting

### Postfix

```bash
sudo systemctl status postfix
sudo ss -tlnp | grep :25
sudo tail -n 20 /var/log/maillog
```

### Dovecot

```bash
sudo systemctl status dovecot
sudo ss -tlnp | grep :143
sudo doveconf -n
```

### Apache / Roundcube

```bash
sudo systemctl status httpd
curl -I http://127.0.0.1/roundcubemail
```

### Roundcube cache

```bash
sudo rm -f /var/lib/roundcubemail/temp/*
sudo rm -f /var/lib/roundcubemail/cache/*
sudo systemctl restart httpd
```

### Firewall

```bash
sudo firewall-cmd --list-all
```

---

## 5. Notes

- Replace `YOUR_DB_PASSWORD` with your own password.
- Replace `SERVER_IP` with your Fedora server IP.
- If Thunderbird login fails with `test3@tp.local`, try just `test3`.
- If Roundcube does not open, check Apache, firewall, SELinux, and cache.
- If mail is not delivered, check Postfix logs and Maildir.

---

## 6. Final Result

At the end of this project, you should have:

- Fedora working as a mail server
- Ubuntu working as a mail client
- Postfix working for SMTP
- Dovecot working for IMAP
- Mail delivered to Maildir
- Roundcube working in browser
- Thunderbird able to send and receive test mail
