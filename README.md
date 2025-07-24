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
    TRAEFIK_DASHBOARD_HOST=traefik.tune.local
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
    image: containous/whoami
    container_name: my-app
    networks:
      - traefik_proxy # Connect to Traefik's network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.my-app.rule=Host(`my-app.yourdomain.com`)" # Your application's domain
      - "traefik.http.routers.my-app.entrypoints=websecure"
      - "traefik.http.routers.my-app.tls.certresolver=myresolver" # Use Let's Encrypt
      - "traefik.http.services.my-app.loadbalancer.server.port=80" # Port your application listens on

networks:
  traefik_proxy:
    external: true # Reference the external network created by Traefik