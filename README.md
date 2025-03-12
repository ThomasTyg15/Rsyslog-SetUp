# Rsyslog-SetUp
This repository guides you through setting up a secure remote logging system using rsyslog with TLS encryption. It ensures secure log transmission from a Linux client to a centralized server. The README provides step-by-step instructions, and a detailed PDF is available for an in-depth explanation of the project.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Setting Up Virtual Machines](#setting-up-virtual-machines)
4. [Configuring rsyslog](#configuring-rsyslog)
5. [Securing with TLS](#securing-with-tls)
6. [Testing the Setup](#testing-the-setup)
7. [Conclusion](#conclusion)


## directory structure
remote-logging-system/
│
├── README.md                  # Detailed README file with setup instructions
├── remote-logging-setup.pdf   # Detailed PDF explaining the project and steps
│
├── client/                    # Directory for client configuration files
│   ├── rsyslog.conf          # Main rsyslog configuration file for the client
│   └── 50-default.conf       # Additional rsyslog configuration for the client
│
└── server/                   # Directory for server configuration files
    ├── rsyslog.conf          # Main rsyslog configuration file for the server
    └── 50-default.conf        # Additional rsyslog configuration for the server

## Introduction

In modern IT environments, logging and monitoring are essential for maintaining system security and performance. This project focuses on setting up a secure remote logging system using rsyslog, which centralizes logs from multiple machines and ensures secure transmission using TLS encryption.

## Prerequisites

- Two virtual machines (VMs) running Ubuntu 2022.
- Basic knowledge of Linux command line.
- VirtualBox or any other virtualization software.

## Setting Up Virtual Machines

### Client VM

1. **Install Ubuntu 2022** on the client VM.
2. **Configure Networking**: Assign a static IP address (e.g., `192.168.1.2`).

### Server VM

1. **Install Ubuntu 2022** on the server VM.
2. **Configure Networking**: Assign a static IP address (e.g., `192.168.1.1`).

Ensure both VMs are on the same internal network to facilitate communication.

## Configuring rsyslog

### On the Server (Log Receiver)

1. **Install rsyslog**:
   ```bash
   sudo apt update && sudo apt install rsyslog -y
   ```

2. **Edit rsyslog Configuration**:
   - Open `/etc/rsyslog.conf` and enable UDP and TCP log reception by uncommenting the following lines:
     ```plaintext
     module(load="imudp")
     input(type="imudp" port="514")

     module(load="imtcp")
     input(type="imtcp" port="6514")
     ```

3. **Restart rsyslog**:
   ```bash
   sudo systemctl restart rsyslog
   sudo systemctl status rsyslog
   ```

4. **Optional**: To store client logs separately, edit `/etc/rsyslog.d/50-default.conf`:
   ```plaintext
   $template RemoteLogs,"/var/log/remote/%HOSTNAME%.log"
   *.* ?RemoteLogs
   ```

5. **Set Permissions**:
   ```bash
   sudo chown syslog:adm /var/log/remote/
   sudo chmod 755 /var/log/remote/
   ```

### On the Client (Log Sender)

1. **Install rsyslog**:
   ```bash
   sudo apt update && sudo apt install rsyslog -y
   ```

2. **Edit rsyslog Configuration**:
   - Open `/etc/rsyslog.d/50-default.conf` and comment out the local log storage line:
     ```plaintext
     # *.*;auth,authpriv.none -/var/log/syslog
     ```

   - Add the following line to forward logs to the server:
     ```plaintext
     *.* @@192.168.1.1
     ```

3. **Restart rsyslog**:
   ```bash
   sudo systemctl restart rsyslog
   sudo systemctl status rsyslog
   ```

## Securing with TLS

### Installation

1. **Install Required Tools**:
   ```bash
   sudo apt install -y rsyslog-gnutls gnutls-bin gnutls-doc
   ```

### Generating Secure Keys

#### On the Server

1. **Generate CA Private Key**:
   ```bash
   certtool --generate-privkey --outfile ca-key.pem --sec-param High
   ```

2. **Generate Self-Signed CA Certificate**:
   ```bash
   certtool --generate-self-signed --load-privkey ca-key.pem --outfile ca.pem
   ```

3. **Generate Server Private Key**:
   ```bash
   certtool --generate-privkey --outfile rsyslog-server-key.pem --sec-param High
   ```

4. **Create Certificate Signing Request (CSR)**:
   ```bash
   certtool --generate-request --load-privkey rsyslog-server-key.pem --outfile rsyslog-server-certificate-request.pem
   ```

5. **Generate Signed Server Certificate**:
   ```bash
   certtool --generate-certificate --load-request rsyslog-server-certificate-request.pem --outfile rsyslog-server-certificate.pem --load-ca-certificate ca.pem --load-ca-privkey ca-key.pem
   ```

#### On the Client

1. **Generate Client Private Key**:
   ```bash
   certtool --generate-privkey --outfile rsyslog-client-key.pem --sec-param High
   ```

2. **Create CSR**:
   ```bash
   certtool --generate-request --load-privkey rsyslog-client-key.pem --outfile rsyslog-client-certificate-request.pem
   ```

3. **Send CSR to Server**:
   ```bash
   scp ./rsyslog-client-certificate-request.pem root@192.168.1.1:/etc/ssl/certs/rsyslog/
   ```

4. **Generate Signed Client Certificate on Server**:
   ```bash
   certtool --generate-certificate --load-request rsyslog-client-certificate-request.pem --outfile rsyslog-client-certificate.pem --load-ca-certificate ca.pem --load-ca-privkey ca-key.pem
   ```

5. **Send Signed Certificate and CA Certificate to Client**:
   ```bash
   scp ./rsyslog-client-certificate.pem root@192.168.1.2:/etc/ssl/certs/rsyslog/
   scp ./ca.pem root@192.168.1.2:/etc/ssl/certs/rsyslog/
   ```

### Configuring TLS

#### On the Server

1. **Edit `/etc/rsyslog.conf`**:
   ```plaintext
   module(
     load="imtcp"
     StreamDriver.Name="gtls"
     StreamDriver.Mode="1"
     StreamDriver.Authmode="x509/name"
     PermittedPeer="192.168.1.2"
   )

   input(type="imtcp" port="6514")

   global(
     DefaultNetstreamDriver="gtls"
     DefaultNetstreamDriverCAFile="/etc/ssl/certs/rsyslog/ca.pem"
     DefaultNetstreamDriverCertFile="/etc/ssl/certs/rsyslog/rsyslog-server-certificate.pem"
     DefaultNetstreamDriverKeyFile="/etc/ssl/certs/rsyslog/rsyslog-server-key.pem"
   )
   ```

2. **Restart rsyslog**:
   ```bash
   sudo systemctl restart rsyslog
   ```

#### On the Client

1. **Edit `/etc/rsyslog.conf`**:
   ```plaintext
   global(
     DefaultNetstreamDriver="gtls"
     DefaultNetstreamDriverCAFile="/etc/ssl/certs/rsyslog/ca.pem"
     DefaultNetstreamDriverCertFile="/etc/ssl/certs/rsyslog/rsyslog-client-certificate.pem"
     DefaultNetstreamDriverKeyFile="/etc/ssl/certs/rsyslog/rsyslog-client-key.pem"
   )
   ```

2. **Restart rsyslog**:
   ```bash
   sudo systemctl restart rsyslog
   ```

## Testing the Setup

1. **Generate a Test Log on the Client**:
   ```bash
   logger -p local0.info "Test log message"
   ```

2. **Check Logs on the Server**:
   ```bash
   tail -f /var/log/remote/client-hostname.log
   ```

## Conclusion

This setup provides a secure and centralized logging system using rsyslog with TLS encryption. It ensures that logs are transmitted securely and are available for analysis and troubleshooting.

---
