---
title:  "Setup SWAG with Nextcloud AIO"
# mathjax: true
layout: post
# categories: media
---

{:toc}
## Scope
Setup [Nextcloud AIO](https://github.com/nextcloud/all-in-one) with [SWAG](https://hub.docker.com/r/linuxserver/swag) (that use a domain registered on Azure DNS Zone in the showed setup).

### Requirements
- Domain (a domain can be bought on [namecheap](https://www.namecheap.com/) for few $/€)
- A working SWAG setup (check my guide: [azure-swag](https://massimilianorighi.github.io/azure-swag/)).

## SWAG setup
In SWAG we have to setup the reverse proxy configuration. There is a pre-configured configuration from Linuxserver that might help us in the setup.

Steps:
1. If for example the SWAG container name is "swag_data", navigate to the volume (vscode with `docker` extension might help you). From the  volume navigate to: `/config/nginx/proxy-confs/`. Here we want to copy and edit the example configuration: `nextcloud.subdomain.conf.sample`
    ```
    cp nextcloud.subdomain.conf.sample nextcloud.subdomain.conf
    ```
    Then open with editor the file:
    ```
    nano nextcloud.subdomain.conf
    ```
    The configuration should be modified to match the following one (check the sections between `<>`):
    ```
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        server_name <subdomain-name>.*;

        include /config/nginx/ssl.conf;

        client_max_body_size 0;

        location / {
            include /config/nginx/proxy.conf;
            include /config/nginx/resolver.conf;
            set $upstream_app <server_ip-or-container_name>; # If you are using Azure VM server_ip=assigned internal IP of the VM (for example 10.0.0.4)
            set $upstream_port 11000; # apache port of Nextcloud
            set $upstream_proto http;
            proxy_pass $upstream_proto://$upstream_app:$upstream_port;

            # Hide proxy response headers from Nextcloud that conflict with ssl.conf
            # Uncomment the Optional additional headers in SWAG's ssl.conf to pass Nextcloud's security scan
            proxy_hide_header Referrer-Policy;
            proxy_hide_header X-Content-Type-Options;
            proxy_hide_header X-Frame-Options;
            proxy_hide_header X-XSS-Protection;

            # Disable proxy buffering
            proxy_buffering off;

            client_body_buffer_size 512k;

        }
    }
    ```
2. Inside SWAG container, navigate to the volume (vscode with `docker` extension might help you). From the volume navigate to: `/config/nginx/` and open `proxy.conf`:
    ```
    nano /config/nginx/proxy.conf
    ```
    We need to modify the `proxy_read_timeout` variable from `240` to `86400s` (configuration taken from [here](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#nginx-proxy-manager)).
3. Now we are ready to deploy Nextcloud AIO (if you need more details, you can find them [here](https://github.com/nextcloud/all-in-one/blob/main/compose.yaml)). Create a new `docker-compose.yml` file:
    {% highlight yaml %}

    version: "3.8"
    services:
        nextcloud-aio-mastercontainer:
            image: nextcloud/all-in-one:latest
            # init: true
            container_name: nextcloud-aio-mastercontainer # This line is not allowed to be changed as otherwise AIO will not work correctly
            environment:
               - APACHE_PORT=11000 # Is needed when running behind a web server or reverse proxy (like Apache, Nginx, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
                # - APACHE_IP_BINDING=0.0.0.0
                # - SKIP_DOMAIN_VALIDATION=true
               - NEXTCLOUD_DATADIR=/home/<username>/docker/volumes/nextcloudaio # Allows to set the host directory for Nextcloud's datadir. ⚠️⚠️⚠️ Warning: do not set or adjust this value after the initial Nextcloud installation is done! See https://github.com/nextcloud/all-in-one#how-to-change-the-default-location-of-nextclouds-datadir
            volumes:
               - nextcloud_aio_mastercontainer:/mnt/docker-aio-config # This line is not allowed to be changed as otherwise the built-in backup solution will not work
               - /var/run/docker.sock:/var/run/docker.sock:ro # May be changed on macOS, Windows or docker rootless. See the applicable documentation. If adjusting, don't forget to also set 'WATCHTOWER_DOCKER_SOCKET_PATH'!
            ports:
               - 80:80 # Can be removed when running behind a web server or reverse proxy (like Apache, Nginx, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
               - 8080:8080
               - 8443:8443 # Can be removed when running behind a web server or reverse proxy (like Apache, Nginx, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
    # environment: # Is needed when using any of the options below
        restart: unless-stopped

    volumes: # If you want to store the data on a different drive, see https://github.com/nextcloud/all-in-one#how-to-store-the-filesinstallation-on-a-separate-drive
        nextcloud_aio_mastercontainer:
            name: nextcloud_aio_mastercontainer # This line is not allowed to be changed as otherwise the built-in backup solution will not work

    networks:
        default:
            name: proxynet

    {% endhighlight %}
4. Spawn the Nextcloud AIO setup container.
5. To reach the setup page there are two options based on the ports opened in the `docker-compose.yml`:
   1.  Via IP address of the VM/server: `https://localhost:8080` or `https://<server-ip-address>:8080`
   2.  If port `8443` is opens, you can reach the AIO interface with a valid certificate using `https://<your-domain.com>:8443`
![](/assets/images/images_2024-02-10-azure-swag/nextcloud_first_login.png)
6. Click on ‘Open Nextcloud AIO login’ and paste in the password copied in the previous page.
7. Setup the subdomain and website
![](/assets/images/images_2024-02-10-azure-swag/domain_setup_nextcloud.png)
8. If the domain is approved by the domain checker container (`aio-domaincheck`) spawned by the `nextcloud-aio-mastercontainer`, if you point a browser to the selected domain you should see something like this:
![](/assets/images/images_2024-02-10-azure-swag/response_from_subdomain.png)

---
If you see 502 error, please follows all the checks reported [here](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#6-how-to-debug-things).
---
9. After you’ve set it up correctly, it should allow you to pass to the next step where you can configure wanted optional addons.
10. Now complete the configuration of the Nextcloud AIO installation. You can select the Optional containers and change the timezone.
![](/assets/images/images_2024-02-10-azure-swag/optional_containers.png)
11. As last step, select the version of Nextcloud to Install (the last one is usually offered as optional). In this case the last version (**v28**) is installed as example.
12. Click on Download and start containers.
13. The page now freeze in loading (while downloading containers required by the Nextcloud AIO).
![](/assets/images/images_2024-02-10-azure-swag/setup_aio_loading.png)
14. When download is completed the webpage is reloaded and the following window appears (wait for all containers to be ready):
![](/assets/images/images_2024-02-10-azure-swag/containers_status.png)
15. When all containers are up and running the following page is loaded:
![](/assets/images/images_2024-02-10-azure-swag/all_services_running.png)
16. **Now the crucial part**. Click on `Open your nextcloud` and if the reverse proxy (SWAG) is correctly setup the following page should appears, congratulations the setup is completed.
17. ![](/assets/images/images_2024-02-10-azure-swag/nextcloud_login.png)

## Sources
- https://nextcloud.com/blog/how-to-install-the-nextcloud-all-in-one-on-linux/
- https://forum.openmediavault.org/index.php?thread/49357-omv-quick-configuration-guide/
- https://www.reddit.com/r/NextCloud/comments/192epks/is_it_possible_to_use_the_nextcloud_aio_docker/