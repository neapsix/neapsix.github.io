---
title: Make a Container Application Available to Internal and External Users
draft: false
---
To keep your domain name service (DNS) configuration simple and make an application available to people both on your internal network and outside your network, you can host the application on two subdomains, such as:
* `<your application>.apps.example.com`, which is resolved only on the internal network.
* `<your application>.proxy.example.com`, which is resolved over the public internet.

This configuration avoids creating a split-horizon DNS, where one domain name is resolved differently depending on whether a user accesses it from internal network, the internet, or another DNS zone.

>**Why use two different subdomains instead of split-horizon DNS?**
>
>While split-horizon DNS gives users one domain name to use everywhere, separate subdomains can help you avoid issues where a request is routed in an inefficient way. For example, if your computer is on the internal network but is configured to use a public DNS server like Google's DNS server at 8.8.8.8, requests to internal websites might be incorrectly routed out to the internet and back instead of remaining on your internal network.
>
>You can also avoid this issue by making sure your computer prioritizes your internal network's DNS server before public DNS servers. However, some applications, such as Google Chrome, might be set up to prioritize a public DNS server by default.

Complete the steps in the following sections to make a container application available on both subdomains using reverse proxy rules. These steps describe setup you can do with Docker Compose files to run containers in a runtime such as Docker or Podman.

>**Prerequisites:**
>
>The machine hosting your container application must have:
>* A container runtime such as Docker or Podman with bridge networking enabled.
>* A Docker Compose file defining a container for your application and a container for the Traefik reverse proxy package.
>
>You also need a publicly available machine to act as a reverse proxy server, such as a virtual private server (VPS). The reverse proxy server needs a web server package, such as nginx, installed.
>
>Both machines must have Wireguard installed.

## Let Users Access the Application from Your Network
First, make sure the local DNS server on your internal network has an entry that maps the internal URL, `<application.apps.example.com`, to your container host's IP address. You can set up a specific DNS entry for `<application>.apps.example.com` or a wildcard entry for `*.apps.example.com`. Your internal DNS server might be hosted by your router or by a separate machine, such as a Raspberry Pi using `pi-hole`.

Then, configure Traefik on the container host to route requests to `<application>.apps.example.com` and to your container using the container runtime's internal routing:

1. In your Docker Compose file, add the following parameters to the definition for your Traefik container.
    * The `providers` lines enable reverse proxy redirection to other containers using internal routing.
    * The `entrypoints` lines set up redirector websites on ports 80 and 443 for a user to hit when visiting http://apps.example.com and https://apps.example.com.

    ```yaml
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ```

1. Go to the definition for your application's container and add the following metadata. These lines enable integration with Traefik and specify that requests to either entrypoint that match the string `<application>.apps.example.com` should be routed to this container.

    ```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.<your container>.rule=Host(`<your application>.apps.example.com`)"
      - "traefik.http.routers.<your container>.entrypoints=web,websecure"
    ```

1. Run `docker-compose up -d` to start your containers.

>**Note:**
>
>This configuration allows insecure HTTP access from the internal network. For testing purposes and home networks, you might not need to access your internal sites over TLS. However, if you want to do so, you can use Traefik to set up TLS automatically. For instructions, refer to the [Docker & Let's Encrypt](https://doc.traefik.io/traefik/v1.7/user-guide/docker-and-lets-encrypt/) topic in the Traefik documentation.

## Let Users Access the Application from the Public Internet
Complete these tasks to enable external access:
* Set up public DNS records and a TLS certificate for sites on the `proxy.example.com` reverse proxy server.
* Create a tunnel between the container host and the reverse proxy server.
* Add a reverse proxy rule that redirects requests for your application to the container host over that tunnel.

### Set Up Public DNS Records and a TLS Certificate for the Reverse Proxy Server
You need public DNS records for your domain pointing the `*.proxy.example.com` subdomain to the IP address of your reverse proxy server. For instructions to set up DNS records for your domain, refer to the documentation for your DNS provider.

In addition, your reverse proxy server needs a valid TLS certificate for `*.proxy.example.com`. This setup depends on your certificate authority and reverse proxy server package. For example, to get a certificate from Let's Encrypt and use it with nginx, refer to the [Using Free Let’s Encrypt SSL/TLS Certificates with NGINX](https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/) topic in the nginx documentation.

### Set Up a Tunnel with the Reverse Proxy Server
Create a tunnel between the two servers using Wireguard. The Wireguard connection should have two peers: the reverse proxy server and the container host. They should be assigned IP addresses on a subnet that doesn't overlap with anything else on your network. For steps to set up a Wireguard tunnel, refer to the [Quick Start](https://www.wireguard.com/quickstart/) topic in the Wireguard documentation.

### Add a Reverse Proxy Rule
Make your container container available over the Wireguard tunnel, and configure the reverse proxy server to redirect requests for `<application>.proxy.example.com` to it.

In your Docker Compose file, in the definition for your application, publish a port on the host. For example, you might have this line mapping port 58001 on the container host to port 80 on the container:

```yaml
ports:
  - "58001:80"
```

Then, set up the reverse proxy server to redirect `<application>.proxy.example.com` to that port on the container host over the Wireguard tunnel. For example, to use a simple nginx site as the reverse proxy server, you might have the following configuration in `/etc/nginx/sites-enabled/<application>.proxy.example.com`:

```nginx
server_name <application>.proxy.example.com www.<application>.proxy.example.com;
set $upstream <IP of tunneled host>:<Published port of application>;
````
