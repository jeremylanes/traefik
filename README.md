# Traefik Load Balancer

This project provides a robust Traefik setup using Docker Compose, ideal for managing and routing traffic to your containerized applications with automatic SSL certificate generation via Let's Encrypt.

---

## Prerequisites

Before you begin, ensure you have the following installed:

* **Docker**: [Get Docker](https://docs.docker.com/get-docker/)
* **Docker Compose**: Comes with Docker Desktop, or [install standalone](https://docs.docker.com/compose/install/)
* **`htpasswd` utility**: Often available in `apache2-utils` or `httpd-tools` packages (e.g., `sudo apt-get install apache2-utils` on Debian/Ubuntu, `sudo yum install httpd-tools` on CentOS/RHEL).

---

## Quick Start

1.  **Clone the repository:**

    ```bash
    git clone https://github.com/jeremylanes/traefik.git
    cd traefik
    ```

2.  **Generate a hashed password for the Traefik dashboard:**

    Replace `yourusername` and `yourpassword` with your desired credentials. The `htpasswd` command will output a string like `yourusername:$$apr1$$nSIR3PCw$$QAv6OwMGxOOzsn2o5UOHr0`.

    ```bash
    htpasswd -nb yourusername yourpassword
    ```

3.  **Create a `.env` file:**

    Create a file named `.env` in the root of your project directory and populate it with the following, using the hashed password generated in the previous step. **Remember to escape any `$` in the hashed password with `$$` in the `.env` file.**

    ```ini
    DASHBOARD_USERNAME=yourusername
    DASHBOARD_PASSWORD_HASH=$$apr1$$nSIR3PCw$$QAv6OwMGxOOzsn2o5UOHr0 # Example: replace with your generated hash, escaping '$'
    TRAEFIK_DASHBOARD_HOST=traefik.yourdomain.com # Replace with your desired dashboard hostname
    ```

    *Example `.env` content based on your provided info:*

    ```ini
    DASHBOARD_USERNAME=lane
    DASHBOARD_PASSWORD_HASH=$$apr1$$nSIR3PCw$$QAv6OwMGxOOzsn2o5UOHr0
    TRAEFIK_DASHBOARD_HOST=traefik.local
    LETSENCRYPT_EMAIL=email@exemple.com
    ```

    **Note:** For `TRAEFIK_DASHBOARD_HOST`, ensure `traefik.tune.local` (or your chosen hostname) resolves to your Docker host's IP address. For local testing, you might need to add an entry to your `/etc/hosts` file (or equivalent on Windows).

4.  **Create the persistent storage for Let's Encrypt certificates:**

    ```bash
    mkdir -p /opt/traefik
    touch /opt/traefik/acme.json
    chmod 600 /opt/traefik/acme.json
    ```

5.  **Start Traefik:**

    ```bash
    docker compose up -d
    ```

---

## Accessing the Traefik Dashboard

Once Traefik is running, you can access its dashboard at the hostname you defined in your `.env` file (e.g., `https://traefik.tune.local`). You will be prompted for the username and password you set up.

---

## Integrating Your Applications

To integrate your applications with Traefik, add them to the `traefik_proxy` network and configure Traefik labels in their `docker-compose.yml` files.

Here's a basic example for a web service:

```yaml
version: '3.8'

services:
  my-app:
    image: containous/whoami # Or your application's image
    container_name: my-app
    networks:
      - traefik_proxy # Connect to Traefik's external network
    labels:
      # Enable Traefik for this service
      - "traefik.enable=true"

      # HTTP Router: Redirects to HTTPS
      - "traefik.http.routers.my-app-http.entrypoints=web"
      - "traefik.http.routers.my-app-http.rule=Host(`my-app.yourdomain.com`)" # Replace with your application's domain
      - "traefik.http.routers.my-app-http.middlewares=my-app-redirect-to-https@docker"

      # HTTPS Router: Secure access with Let's Encrypt
      - "traefik.http.routers.my-app-https.entrypoints=websecure"
      - "traefik.http.routers.my-app-https.rule=Host(`my-app.yourdomain.com`)" # Replace with your application's domain
      - "traefik.http.routers.my-app-https.tls=true"
      - "traefik.http.routers.my-app-https.tls.certresolver=myresolver" # Use the Let's Encrypt resolver from Traefik

      # Define the target service and its internal port
      - "traefik.http.services.my-app-service.loadbalancer.server.port=80" # The port your application listens on internally

      # Middleware for HTTP to HTTPS redirection
      - "traefik.http.middlewares.my-app-redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.middlewares.my-app-redirect-to-https.redirectscheme.permanent=true"

networks:
  traefik_proxy:
    external: true # This refers to the network created by your main Traefik setup
```

Absolument ! Voici la section complète, mise en forme en Markdown :

---

## Key Features

* **Automatic HTTPS**: Traefik automatically obtains and renews SSL certificates from Let's Encrypt for your services.
* **Docker Integration**: Automatically discovers and configures routes for your Docker containers based on labels.
* **Dashboard**: A web UI for monitoring and managing your Traefik instance.
* **Basic Authentication**: Secure your Traefik dashboard with a username and password.

---

## Configuration Details

### `docker-compose.yml`

#### `traefik` service:

* Uses `traefik:v3.4` image.
* Exposes ports **80** (HTTP) and **443** (HTTPS).
* Mounts `/var/run/docker.sock` for Docker integration.
* Mounts `/opt/traefik/acme.json` for persistent storage of Let's Encrypt certificates.
* Configures HTTP (`web`) and HTTPS (`websecure`) entrypoints.
* Enables the Docker provider, exposing services only if explicitly labeled.
* Configures `myresolver` for Let's Encrypt using HTTP challenge.
* Enables the API dashboard (accessible via a secure router).
* Sets up a router for the dashboard (`traefik-dashboard`) accessible via the `TRAEFIK_DASHBOARD_HOST` environment variable.
* Applies `dashboard-auth` middleware for basic authentication using credentials from the `.env` file.

### `.env` file

* **`DASHBOARD_USERNAME`**: Username for dashboard basic authentication.
* **`DASHBOARD_PASSWORD_HASH`**: Hashed password for dashboard basic authentication (remember to escape `$` with `$$`).
* **`TRAEFIK_DASHBOARD_HOST`**: The hostname you'll use to access the Traefik dashboard.

---

## Troubleshooting

### Dashboard Not Accessible:

* Check your `.env` file for correct `TRAEFIK_DASHBOARD_HOST` and `DASHBOARD_USERNAME`/`DASHBOARD_PASSWORD_HASH`.
* Ensure the `traefik_proxy` network is correctly created and connected.
* Verify DNS resolution for `TRAEFIK_DASHBOARD_HOST`.
* Check Traefik logs: `docker logs traefik`.

### SSL Certificate Issues:

* Ensure port 80 is accessible externally for Let's Encrypt HTTP challenges.
* Check permissions and existence of `/opt/traefik/acme.json`. It should be `chmod 600`.
* Verify the email address configured for Let's Encrypt is valid.
* Review Traefik logs for ACME resolver errors.

---