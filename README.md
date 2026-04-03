# Fedora Local Mail Server Setup with Postfix, Dovecot, and Roundcube

This project shows how to build a **local mail server on Fedora** using:

- **Postfix** for SMTP mail delivery
- **Dovecot** for IMAP access
- **Apache + PHP + MariaDB** for webmail support
- **Roundcube** as the webmail client
- **Thunderbird** for desktop mail testing

The original lab includes:
- Fedora package installation
- Postfix configuration
- Dovecot configuration
- Apache/PHP/MariaDB setup
- Roundcube installation and database setup
- SELinux and permission fixes
- Browser and Thunderbird testing :contentReference[oaicite:0]{index=0}

---

## Project Goal

The goal of this lab is to create a working **local email environment** where local users can:

- send mail with Postfix
- receive mail in `Maildir`
- read mail through IMAP with Dovecot
- access mail using Roundcube webmail
- test mail accounts using Thunderbird :contentReference[oaicite:1]{index=1}

---

## Environment

- **OS:** Fedora
- **Mail domain:** `tp.local`
- **Mail host:** `mail.tp.local`
- **Webmail URL example:** `http://SERVER_IP/roundcubemail/` :contentReference[oaicite:2]{index=2}

---

## Packages Used

### Mail server
- postfix
- mailx
- dovecot

### Web stack
- httpd
- php
- php-mysqlnd
- php-intl
- php-mbstring
- php-json
- php-xml
- mariadb-server

### Webmail
- roundcubemail

### Testing
- telnet
- thunderbird :contentReference[oaicite:3]{index=3}

---

## 1. Update the system

```bash
sudo dnf update -y
sudo dnf install nano -y
