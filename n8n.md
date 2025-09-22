# 🗂️ Runbook Overview

- [Step 1: Create a New Virtual Machine on ESXi](#step-1-create-a-new-virtual-machine-on-esxi)
- [Step 2: Install Ubuntu Server on the VM](#step-2-install-ubuntu-server-on-the-vm)
- [Step 3: Update the System and Install Prerequisites](#step-3-update-the-system-and-install-prerequisites)
- [Step 4: Install Docker and Docker Compose](#step-4-install-docker-and-docker-compose)
- [Step 5: Set Up DNS](#step-5-set-up-dns)
- [Step 6: Configure n8n with Docker Compose](#step-6-configure-n8n-with-docker-compose)
- [Step 7: Post-Installation and Best Practices](#step-7-post-installation-and-best-practices)

# Runbook: How to Install n8n on an ESXi VM

## Overview
This runbook provides a step-by-step guide to installing n8n (a workflow automation tool) on a virtual machine (VM) hosted on VMware ESXi. The installation uses Docker Compose as outlined in the official n8n documentation. We assume you have administrative access to the ESXi host and basic familiarity with virtualization and Linux command-line operations.

n8n will be set up with Traefik as a reverse proxy for handling HTTPS and routing. This setup is suitable for production environments and includes automatic SSL certificate generation via Let's Encrypt.

### Assumptions
- ESXi version 7.0 or later.
- Access to the ESXi web interface (VMware Host Client or vSphere Client).
- A static IP address for the VM.
- Domain name and subdomain configured for n8n (e.g., n8n.example.com) with DNS A record pointing to the VM's IP.
- Guest OS: Ubuntu Server 24.04 LTS (recommended for compatibility with Docker).
- Hardware requirements for the VM: At least 2 vCPUs, 4 GB RAM, 20 GB disk space (adjust based on usage).

### Prerequisites
- Download the Ubuntu Server ISO from the official Ubuntu website (e.g., ubuntu-24.04-live-server-amd64.iso).
- Ensure your ESXi host has a datastore with sufficient space.
- Basic networking setup on ESXi (e.g., VM network port group).

## Step 1: Create a New Virtual Machine on ESXi
1. Log in to the ESXi host using the VMware Host Client (https://<ESXi_IP>/ui/) or vSphere Client.
2. Right-click the **Host** in the inventory panel and select **Create/Register VM**.
3. In the wizard:
   - **Select creation type**: Choose **Create a new virtual machine**.
   - **Select a name and guest OS**: Enter a name (e.g., "n8n-VM"), select **Linux** as the guest OS family, and **Ubuntu Linux (64-bit)** as the version.
   - **Select storage**: Choose a datastore for the VM files.
   - **Customize settings**:
     - CPU: 2 cores.
     - Memory: 4 GB.
     - New Hard disk: 20 GB (Thin provisioned).
     - New Network adapter: Connect to your VM network.
     - New CD/DVD Drive: Select **Datastore ISO file** and upload the Ubuntu ISO to the datastore first (via **Datastore browser** > Upload).
   - Review and click **Finish**.
4. Power on the VM and open the console to begin installation.

## Step 2: Install Ubuntu Server on the VM
1. In the VM console, the Ubuntu installer will boot from the ISO.
2. Follow the on-screen prompts:
   - Select your language and keyboard layout.
   - Choose **Install Ubuntu Server**.
   - Network: Configure a static IP if needed (recommended for servers).
   - Proxy: Skip unless required.
   - Mirror: Use default.
   - Storage: Use the entire disk and set up LVM (default).
   - Profile setup: Create a username (e.g., "ubuntu"), password, and hostname (e.g., "n8n-server").
   - SSH: Enable OpenSSH server for remote access.
   - Featured server snaps: None needed initially.
3. Complete the installation and reboot. Remove the ISO from the CD/DVD drive in VM settings to avoid rebooting into the installer.
4. Log in via console or SSH using the credentials you set.

## Step 3: Update the System and Install Prerequisites
1. SSH into the VM (e.g., `ssh ubuntu@<VM_IP>`).
2. Update packages:
   ```
   sudo apt update && sudo apt upgrade -y
   ```
3. Install essential tools:
   ```
   sudo apt install curl gnupg lsb-release -y
   ```

## Step 4: Install Docker and Docker Compose
1. Add Docker's official GPG key:
   ```
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   ```
2. Set up the Docker repository:
   ```
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```
3. Update and install Docker:
   ```
   sudo apt update
   sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
   ```
4. Add your user to the Docker group (to run Docker without sudo):
   ```
   sudo usermod -aG docker $USER
   newgrp docker
   ```
5. Verify installation:
   ```
   docker --version
   docker compose version
   ```

## Step 5: Set Up DNS
1. In your domain registrar's DNS settings, create an A record for your subdomain (e.g., n8n.example.com) pointing to the VM's public IP address.

## Step 6: Configure n8n with Docker Compose
1. Create a project directory:
   ```
   mkdir n8n-compose && cd n8n-compose
   ```
2. Create an `.env` file with your configurations (edit as needed):
   ```
   DOMAIN_NAME=example.com
   SUBDOMAIN=n8n
   GENERIC_TIMEZONE=America/New_York
   SSL_EMAIL=your@email.com
   ```
3. Create a `local-files` directory for file sharing:
   ```
   mkdir local-files
   ```
4. Create a `compose.yaml` file with the following content:
   ```yaml
   services:
     traefik:
       image: "traefik"
       restart: always
       command:
         - "--api.insecure=true"
         - "--providers.docker=true"
         - "--providers.docker.exposedbydefault=false"
         - "--entrypoints.web.address=:80"
         - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
         - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
         - "--entrypoints.websecure.address=:443"
         - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
         - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
         - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
       ports:
         - "80:80"
         - "443:443"
       volumes:
         - traefik_data:/letsencrypt
         - /var/run/docker.sock:/var/run/docker.sock:ro

     n8n:
       image: docker.n8n.io/n8nio/n8n
       restart: always
       ports:
         - "127.0.0.1:5678:5678"
       labels:
         - traefik.enable=true
         - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
         - traefik.http.routers.n8n.tls=true
         - traefik.http.routers.n8n.entrypoints=web,websecure
         - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
         - traefik.http.middlewares.n8n.headers.SSLRedirect=true
         - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
         - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
         - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
         - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
         - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
         - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
         - traefik.http.middlewares.n8n.headers.STSPreload=true
         - traefik.http.routers.n8n.middlewares=n8n@docker
       environment:
         - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
         - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
         - N8N_PORT=5678
         - N8N_PROTOCOL=https
         - N8N_RUNNERS_ENABLED=true
         - NODE_ENV=production
         - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
         - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
         - TZ=${GENERIC_TIMEZONE}
       volumes:
         - n8n_data:/home/node/.n8n
         - ./local-files:/files

   volumes:
     n8n_data:
     traefik_data:
   ```
5. Start the services:
   ```
   docker compose up -d
   ```
6. Verify: Access n8n at https://n8n.example.com (replace with your subdomain). It may take a few minutes for SSL certificates to generate.

## Step 7: Post-Installation and Best Practices
- **Security**: Use a firewall (e.g., UFW on Ubuntu: `sudo ufw allow 80,443/tcp`). Enable automatic updates.
- **Persistence**: Data is stored in Docker volumes (`n8n_data` for database, `traefik_data` for certificates).
- **Updates**: Pull latest images and restart: `docker compose pull && docker compose up -d`.
- **Troubleshooting**: Check logs with `docker compose logs -f`. Ensure ports 80/443 are open and DNS propagates.
- **Scaling**: For production, consider adding queue mode or external database (refer to n8n docs for advanced configs).
- **Backup**: Regularly back up the `n8n_data` volume and `.env` file.

This setup follows the official n8n Docker Compose guide. For customizations, consult the full documentation.
