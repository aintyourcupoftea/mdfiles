# Deploying a Docker Container on Ubuntu VPS with NGINX Reverse Proxy

This guide walks you through setting up a production-ready Docker container deployment on an Ubuntu server, complete with NGINX reverse proxy and SSL certificates.

## Prerequisites

Before starting, ensure you have an Ubuntu server instance ready with a public IP address and domain name configured.

## Initial Server Access

### SSH Key Setup

First, connect to your server using SSH:

```bash
ssh root@your-server-ip
```

If this is your first time connecting, you'll need to set up SSH key authentication for secure, password-free access.

Navigate to the SSH directory:

```bash
cd ~/.ssh
```

Generate a new SSH key pair:

```bash
ssh-keygen -t rsa -b 4096
```

Add your **local machine's public key** to the server's `authorized_keys` file:

```bash
nano ~/.ssh/authorized_keys
```

Paste your local machine's public key (the content from your local `~/.ssh/id_rsa.pub` file) into this file, save, and exit.

Now you can SSH into the server from your local machine without a password:

```bash
ssh username@your-server-ip
```

## System Updates and Docker Installation

### Update System Packages

Update the package lists to ensure you're installing the latest versions:

```bash
sudo apt-get update
```

### Install Docker

Install Docker Engine on Ubuntu:

```bash
sudo apt install docker.io -y
```

### Install Docker Compose v2

Install the Docker Compose plugin (v2), which uses the `docker compose` syntax instead of the legacy `docker-compose` command:

```bash
sudo apt install docker-compose-plugin -y
```

Alternatively, for the latest version from the official repository:

```bash
mkdir -p ~/.docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
```

Verify the installation:

```bash
docker compose version
```

If you see a version number, you're ready to proceed.

## Deploy Your Docker Container

Run your Docker container with the appropriate configuration:

```bash
docker run -d \
  --restart=always \
  -p 3000:3000 \
  -v app-data:/app/data \
  --name my-app \
  username/my-app:latest
```

Replace the following placeholders:
- `3000:3000` - Change both ports to match your application's exposed port
- `app-data` - Choose a meaningful volume name for persistent data
- `my-app` - Use a descriptive container name
- `username/my-app:latest` - Use your actual Docker image name and tag

This command runs the container in detached mode with automatic restart, maps the specified port, and creates a persistent volume for data storage.

## Firewall Configuration

### Install UFW

Install the Uncomplicated Firewall (UFW) to manage server security:

```bash
sudo apt install ufw -y
```

### Configure Firewall Rules

Allow essential services through the firewall:

```bash
# Allow SSH (important - don't lock yourself out!)
sudo ufw allow 22/tcp

# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable the firewall
sudo ufw enable
```

Check the firewall status:

```bash
sudo ufw status
```

**Note:** Docker containers bypass UFW by default because Docker manipulates iptables directly. Since we're using NGINX as a reverse proxy, the Docker container ports don't need to be directly exposed to the internet, which improves security.

## NGINX Configuration

### Install NGINX

Install NGINX web server to act as a reverse proxy:

```bash
sudo apt install nginx -y
```

### Create NGINX Configuration File

Create a new site configuration file (replace `myapp` with your service name):

```bash
sudo nano /etc/nginx/sites-available/myapp
```

Add the following configuration:

```nginx
server {
    listen 80;
    server_name myapp.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Replace the following:
- `myapp` - Your service/application name
- `myapp.yourdomain.com` - Your actual subdomain
- `localhost:3000` - Match the port your Docker container exposes

## DNS Configuration

Before proceeding, configure your DNS settings in Cloudflare (or your DNS provider):

1. Log into Cloudflare
2. Select your domain
3. Navigate to DNS settings
4. Add an **A record**:
   - **Name:** myapp (your chosen subdomain)
   - **IPv4 address:** Your VPS public IP
   - **Proxy status:** DNS only (disable the orange cloud)

Wait a few minutes for DNS propagation to complete.

## Enable NGINX Site and SSL

### Enable the Site Configuration

Create a symbolic link to enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
```

### Test NGINX Configuration

Verify the NGINX configuration syntax:

```bash
sudo nginx -t
```

If the test passes, reload NGINX to apply changes:

```bash
sudo systemctl reload nginx
```

### Install SSL Certificate

Install Certbot to obtain a free SSL certificate from Let's Encrypt:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Obtain and install the SSL certificate:

```bash
sudo certbot --nginx -d myapp.yourdomain.com
```

Follow the prompts to complete the SSL setup. Certbot will automatically configure NGINX to use HTTPS and redirect HTTP traffic to HTTPS.

## Verification

Your Docker application is now live and accessible at `https://myapp.yourdomain.com`.

To verify everything is working:

```bash
# Check Docker container status
docker ps

# Check NGINX status
sudo systemctl status nginx

# Check firewall status
sudo ufw status
```

## Security Best Practices

- Keep your system updated: `sudo apt update && sudo apt upgrade -y`
- Use strong SSH keys and disable password authentication
- Regularly update Docker images: `docker pull imagename:tag`
- Monitor logs: `docker logs container-name` and `sudo tail -f /var/log/nginx/access.log`
- Enable automatic security updates
- Use Cloudflare's proxy (orange cloud) for additional DDoS protection after SSL is configured

## Troubleshooting

### Container won't start
```bash
docker logs my-app
```

### NGINX configuration errors
```bash
sudo nginx -t
sudo tail -f /var/log/nginx/error.log
```

### Firewall issues
```bash
sudo ufw status verbose
sudo ufw allow port/tcp
```

### SSL certificate issues
```bash
sudo certbot renew --dry-run
sudo certbot certificates
```

## Additional Resources

- Docker Documentation: https://docs.docker.com/
- NGINX Documentation: https://nginx.org/en/docs/
- Certbot Documentation: https://certbot.eff.org/
- UFW Guide: https://help.ubuntu.com/community/UFW

---

**Last Updated:** October 2025
