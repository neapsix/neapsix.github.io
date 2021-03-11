---
title: Make Your Container Application Available to Internal and External Users
draft: false
---
To make your web application available to users both on the internal network and over the internet, while keeping domain name service (DNS) configuration simple, you can host the application on two subdomains of the neap.space domain:
* `<your application>.apps.neap.space`, which is resolved only on the internal network.
* `<your application>.proxy.neap.space`, which is resolved over the public internet.

Complete the steps in the following sections to make a container application available to users by adding it to the reverse proxy rules for each subdomain. These steps apply only to applications in a container runtime such as Docker or Podman that you configure using Docker Compose.

>#### Prerequisites:
>
>For both internal and external access:
>* An authorized SSH key and sudo access on the proxy.neap.space server.
>* Wireguard installed on the container host.
>* A container for Traefik on the same host as your application.
> 
>For external access:
>* Wireguard installed on the container host.
>* For your application, bridge networking enabled and ports published on the container host.
>* An authorized SSH key and sudo access on the proxy.neap.space server.

>#### Why use two different subdomains?
>
>The neap.space network uses this configuration to avoid creating a split-horizon DNS, where one domain name is resolved differently depending on whether a user accesses it from internal network, the internet, or another specified DNS zone.
>
>
>While split-horizon DNS gives users one domain name to use everywhere, separate subdomains can help you avoid issues where a request goes to a different DNS server than you expect. For example, applications like Chrome might prioritize a certain public DNS server over the local one, causing internal requests to be resolved as though they were from the public internet.

## Let Users Access the Application from Your Network
First, make sure the local DNS server has an entry that points the application's internal URL to your container host. You might have a specific entry for `<application>.apps.neap.space` or a wildcard entry for `*.apps.neap.space`.

Then, configure Traefik on the container host to match requests to `<application>.apps.neap.space` and route them to your container using the container runtime's internal routing:

1. In your Docker Compose file, add the following parameters to the Traefik container if you haven't already.

    ```yaml
    command:
      # Reverse proxy to docker containers. Each container must have "traefik.enable=true" in labels
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      # Define entry points a user hits when going to http://<domain> and https://<domain>
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ```

1. Go to the definition for your container and add the following metadata. Replace `<application>` with the name of your application.

    ```yaml
    labels:
      # Enable resolution on the internal domain with traefik
      - "traefik.enable=true"
      - "traefik.http.routers.testsite.rule=Host(`<application>.apps.neap.space`)"
      - "traefik.http.routers.testsite.entrypoints=web,websecure"
    ```

1. Run `docker-compose up -d` to start your containers.

Note that this configuration allows insecure HTTP access from the internal network. For testing purposes and small secure networks, you might not need secure HTTPS access. If you want to modify this configuration to use HTTPS, refer to the [Docker & Let's Encrypt](https://doc.traefik.io/traefik/v1.7/user-guide/docker-and-lets-encrypt/) topic in the Traefik documentation.

## Let Users Access the Application from the Public Internet
The public DNS records for the neap.space domain already have an entry pointing the `*.proxy.neap.space` subdomain to a reverse proxy server, and that server has a valid TLS certificate for `*.proxy.neap.space`. Because these steps are done for you, you don't need to do anything to set up name resolution or a TLS certificate for external access to your application.

Complete these tasks to enable external access:
* Create a tunnel between the container host and the proxy.neap.space reverse proxy server.
* Add a reverse proxy rule that redirects requests for your application to the container host and your container's published port over the tunnel you created.

### Set Up a Tunnel with the Reverse Proxy Server
First, create the secure tunnel using Wireguard. The tunnel should have two peers: the reverse proxy server and the container host. They should be assigned IP addresses on a subnet that doesn't overlap with anything else on the network. For steps to set up and enable a Wireguard tunnel, refer to the [Quick Start](https://www.wireguard.com/quickstart/) topic in the Wireguard documentation.

### Add a Rule to the Reverse Proxy Configuration
Next, update the reverse proxy server configuration on the proxy.neap.space host to redirect requests for `<application>.proxy.neap.space` to your container host's IP address and your container's published port.

The proxy.neap.space server runs nginx with a simple configuration to redirect URLs to specific IP addresses. To add your application to the list, you can work from a template nginx configuration file and fill in the details for your container.

1. In your Docker Compose file, make sure that the container publishes ports on the host. For example, you might have this line:

    ```yaml
    ports:
        - "58001:80" # Port 58001 on the host is mapped to port 80 on the container.
    ```

1. Run `ssh <user>@proxy.neap.space` to access the reverse proxy server. Use your authorized key.
1. Make a new copy of the template configuration and add it to the enabled sites directory.

    ```sh
    sudo cp /etc/nginx/sites-available/example.proxy.neap.space.conf \
    /etc/nginx/sites-enabled/<application>.proxy.neap.space.conf
    ```

1. Open your copy with your preferred text editor, and update the following lines. Replace the text in angle brackets with the details for your container application.

    ```nginx
    server_name <application>.proxy.neap.space www.<application>.proxy.neap.space;
    set $upstream <IP of tunneled host>:<Published port of website>;
    ```

1. Restart nginx to deploy your configuration. To do so, run `sudo systemctl restart nginx`.
