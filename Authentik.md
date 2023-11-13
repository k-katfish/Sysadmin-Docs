# Authentick (LDAP++)

Remember all that work you just did to build your very own LDAP server? Well forget all of it because Authentik just dropped and it's awesome.

## Installation

I know, I know, docker in prod bad. But that's their official recommendation for "small-scale production" setups. Besides it's either Docker or Kubernetes, pick your posion.

0. Your server needs to be running in UTC time. You should always do everything in UTC time anyway, but this specifically must be in UTC time, or oauth will fail.

1. Install docker & docker compose

    <https://docs.rockylinux.org/gemstones/docker/>

    ```bash
    sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
    sudo systemctl --now enable docker
    reboot # seriously docker is one of the few stupid things you actually have to rebot for to work
    ```

2. Grab Authentik's docker-compose file

    ```bash
    wget https://goauthentik.io/docker-compose.yml
    ```

3. Generate a password & secret key and write them to the .env file:

    ```bash
    echo "PG_PASS=$(openssl rand -base64 36)" >> .env
    echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 36)" >> .env
    ```

4. Configure email: add these to your .env file

    ```conf
    # SMTP Host Emails are sent to
    AUTHENTIK_EMAIL_HOST=localhost
    AUTHENTIK_EMAIL_PORT=25
    # Optionally authenticate (don't add quotation marks to your password)
    AUTHENTIK_EMAIL_USERNAME=
    AUTHENTIK_EMAIL_PASSWORD=
    # Use StartTLS
    AUTHENTIK_EMAIL_USE_TLS=false
    # Use SSL
    AUTHENTIK_EMAIL_USE_SSL=false
    AUTHENTIK_EMAIL_TIMEOUT=10
    # Email address authentik will send from, should have a correct @domain
    AUTHENTIK_EMAIL_FROM=authentik@localhost
    ```

5. Spin up your authentik instance

    ```bash
    docker compose pull
    docker compose up -d
    ```

6. Configure Nginx as a proxy for port 9000 and 9443:

    ```bash
    # Acquire certificates
    sudo dnf install -y epel-release firewalld
    sudo dnf install -y certbot
    sudo certbot certonly --standalone --non-interactive --agree-tos --email [mail] --server https://acme.sectigo.com/ --eab-kid [your kid] --eab-hmac-key [your-hmac-key] --domain authentik.example.com --cert-name NGINX

    # Configure firewall
    sudo firewall-cmd --permanent --add-service=https
    sudo firewall-cmd --reload

    # Configure Nginx
    sudo dnf update
    sudo dnf install epel-release
    sudo dnf module enable nginx:mainline
    sudo dnf install -y nginx
    sudo vim /etc/nginx/nginx.conf
    ## edit as shown in next block
    # Link conf file:

    sudo nginx -t
    sudo systemctl restart nginx 

    ```

    ```conf
    server {
        listen 443 ssl;
        server_name authentik.example.com;

        ssl_certificate /etc/letsencrypt/live/NGINX/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/NGINX/privkey.pem;

        ssl_prever_server_ciphers on;

        location / {
            proxy_pass http://localhost:9000;

            proxy_set_header  HOST $host;
            proxy_set_header  X-REAL-IP $remote_addr;
            proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header  X-Forwarded-Proto $scheme;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_cache_bypass $http_upgrade;
        }
    }
    ```

## Setup and configuration

Go to <https://your-server/if/flow/initial-setup>

## SELinux tips

If you get a 502 bad gateway when tryna connect to nginx on 443; try the following:

1. Identify the port type for selinux: `semanage port -l | grep 9000`

    ```bash
    http_port_t     tcp     80, 81, 443, 488, 8008, 8009, 8443, 9000
    ```

2. semanage port -a -t http_port_t -p tcp 9000

    something something selinux something something reboot

3. cry

4. fight with chatgpt over who's right about how to use selinux (hint: it's not chatgpt... or you)

5. semanage permissive -a httpd_t ; reboot

## Migrating users over from LDAP

1. In the Authentik Admin console, navigate to Directory -> Federation and Social login.
2. Click "Create".
3. Choose "LDAP source" as your option, next
4. Give it a name and a "slug" (unique name).
5. For absolute integration, check Enabled, Sync users, User password writeback, and Sync groups.
6. ????

## Monitoring tips

Use telegraf :)

inputs.net
inputs.nginx
