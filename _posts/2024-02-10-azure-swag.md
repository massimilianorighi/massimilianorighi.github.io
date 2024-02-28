---
title:  "Setup SWAG with Azure DNS"
# mathjax: true
layout: post
# categories: media
---

{:toc}
## Scope
Create a reverse proxy using [SWAG](https://hub.docker.com/r/linuxserver/swag) that use a domain registered on Azure DNS Zone.

### Requirements
- Domain (a domain can be bought on [namecheap](https://www.namecheap.com/) for few $/€)
- Azure Account
- Azure VM (not mandatory but highly suggested)

## Azure setup

### Create DNS resource
1. Navigate to the [Azure portal](https://portal.azure.com/) .
2. On the Home screen click `Create a resource` button.
3. In the search box type `dns zone`.
4. Select `DNS zone` and click `Create` button.
5. On the Basic tab:
   - **Resource Group**: click `Create new` and enter the name you like or select existing one.
   - **Name**: enter dns zone name: e.g. `reverseproxynet.xyz`.
   - **Resource group location**: automatically select, or select region closest to you (check [this](https://infrastructuremap.microsoft.com/explore) or [this](https://build5nines.com/map-azure-regions/)).
6. Click `Review + create`
7. When DNS zone resource is created navigate to it. On the **Overview** page you will see list of name savers (1 to 4), that need to be copy/pasted to the your’s domain registrar (e.g: [namecheap](https://www.namecheap.com/)).
![](/assets/images/images_2024-02-10-azure-swag/dns_zone_example.png)


### Create Domain Registar
1. Go to Namecheap and select: `Domain List-> Domain`.
2. In `NAMESERVERS` section, select the `Custom DNS` option from the dropdown menu. Now copy all four name servers from the Azure DNS zone resource created on previous section.
![](/assets/images/images_2024-02-10-azure-swag/namecheap.png)
3. Wait few minutes for nameserver to propagate.
4. You can check if name servers were applied, by calling command below:
```bash
nslookup -type=SOA libertus.dev
```
In the response you should see `origin` that points to one of the Azure DNS name servers we specified above, if it is not just try in a few minutes.

```bash
...
massimilianorighi@KrakaMacBookPro ~ % nslookup -type=SOA reverseproxynet.xyz
Server:		192.168.5.1
Address:	192.168.5.1#53

Non-authoritative answer:
reverseproxynet.xyz
	origin = ns1-XX.azure-dns.com
	mail addr = azuredns-hostmaster.microsoft.com
	serial = 1
	refresh = 3600
	retry = 300
	expire = 2419200
	minimum = 300

Authoritative answers can be found from:
...
```

It is now possible control your domain via Azure portal, automate scripts and/or your application logic.

### Create Subdomanin
1. Obtain the public IP address for you server (for Azure VM this can be found on the VM `overview` section).
2. Go to the domain registrar (Azure DNS Zone) and create `A record`:
   - Click `+ Record set` button.
   - Enter **Name**: e.g. `whoami` and `www`.
   - Select **Type**:: `A - Alias record`.
   - Enter **IP address**: **-> your VM Public IP address <-**.

![](/assets/images/images_2024-02-10-azure-swag/a_record.png)

### Configure DNS-01 challenge
SWAG support http or DNS validation. The DNS challenge is a bit more tricky than HTTP, because it requires access to your DNS provider via API. SWAG needs to access the DNS provider via API, in order to create a DNS TXT record, and then Let’s Encrypt can use it to validate the subdomain ownership.

In this case Azure is the DNS provider. We now need to create an azure application that will be used by SWAG, Traefik or whatelse to get access to the Azure DNS Zone.

SWAG DNS configuration for Azure requires the following inputs:
- dns_azure_sp_client_id
- dns_azure_sp_client_secret
- dns_azure_tenant_id
- dns_azure_environment
- dns_azure_zone1

Let's find out all these info.
#### dns_azure_zone1
From `DNS Zone` page, we need:
- Resource group name: e.g. `test`
- Subscription ID: e.g. `beb6bvvd-d251-45be-a1db-f26b87da91e`
- DNS Zone name: e.g. `reverseproxynet.xyz`

![](/assets/images/images_2024-02-10-azure-swag/subscriptionid.png)

#### dns_azure_tenant_id
On Azure portal search for `Microsoft Entra ID`, then you should see a page like the one below that contains basic information about your tenant.
- Tenant ID: `d8x72dd3-f729-4fe1-0f6e-12dcb6c82d5c`

![](/assets/images/images_2024-02-10-azure-swag/tenantid.png)

#### dns_azure_sp_client_id and dns_azure_sp_client_secret
We need to create an Azure App to have these two information.
Steps:
1. Go to `Microsoft Entra ID`
2. On the left panel, select `App Registrations`
3. Click `New Registration` button.
   - Enter Name
   - Select the longest expiration as possible (**if it stops working all services will be blocked! Keep note of the date, and remember to update it!**)
4. Click `Register` button.
![](/assets/images/images_2024-02-10-azure-swag/register_an_app.png)
5. On the Overview tab of just created app, copy the `Application (client) ID`.
   - dns_azure_sp_client_id = e.g. `689264e4-5e48-172f-a584-dd4124c57c29`
6. Go to the `Certificates & secrets` tab on the left
   - Click New client secret to generate a secret key.
   - Copy new secret `Value` to `dns_azure_sp_client_secret`:
   - dns_azure_sp_client_secret: e.g.`EnG9Q~5tG2Z-_Fi0nZOabmkvaAMxcgBX_YPcOaAo`

#### Assign permission
One more thing is left, we need to assign permissions to just created app.
- Go to your subscription `Overview` page.
- Select `Access Control (IAM)` tab.
- Click `+Add` button.
  - Select `Add role assignment` option.
  - In the search box, type `dns`.
  - Select `DNS Zone Contributor` role.
- Click `Next` button.
  - Click `+ Select members` button.
  - In the search box type app name you created above and select it (in this example is: `reverseproxynet.xyz`).
![](/assets/images/images_2024-02-10-azure-swag/dns_contributor_app.png)
  - Click `Review + Assign` button.
![](/assets/images/images_2024-02-10-azure-swag/app_role_assigned.png)

## SWAG Configuration

1. Deploy the swag container:
    {% highlight yaml %}

    version: "3.8"
    services:
        swag:
            image: lscr.io/linuxserver/swag:latest
            container_name: swag
            cap_add:
               - NET_ADMIN
            environment:
               - PUID=1000
               - PGID=1000
               - TZ=Europe/Rome
               - URL=reverseproxynet.xyz
               - SUBDOMAINS=wildcard
               - VALIDATION=dns
               - DNSPLUGIN=azure
               - EMAIL=<myemail@reverseproxynet.xyz> #optional (doesn't need to be related to domain)
               - DOCKER_MODS=linuxserver/mods:swag-auto-reload
            volumes:
               - swag_data:/config
            ports:
               - 443:443
               - 80:80
            extra_hosts:
               - reverseproxynet.xyz:127.0.0.1
            restart: unless-stopped
    volumes:
        swag_data:

    networks:
        default:
            name: proxynet
            external: true

    {% endhighlight %}
2. Now that the container is running, navigate to the volume "swag_data" (for example vscode with docker extension). From the volume edit `azure.ini`:
```
nano /config/dns-conf/azure.ini
```
Here we need to configure the Azure information created before: `dns_azure_sp_client_id`, `dns_azure_sp_client_secret`, `dns_azure_tenant_id`, `dns_azure_environment`, `dns_azure_zone1`.
For example using all the credentials extracted before:
```
# Instructions: https://certbot-dns-azure.readthedocs.io/en/latest/
# Replace with your values
# dns_azure_environment can be one of the following: AzurePublicCloud, AzureUSGovernmentCloud, AzureChinaCloud, AzureGermanCloud
# Service Principal with Client Secret
dns_azure_sp_client_id = 689264e4-5e48-172f-a584-dd4124c57c29
dns_azure_sp_client_secret = EnG9Q~5tG2Z-_Fi0nZOabmkvaAMxcgBX_YPcOaAo
dns_azure_tenant_id = d8x72dd3-f729-4fe1-0f6e-12dcb6c82d5c
dns_azure_environment = "AzurePublicCloud"
dns_azure_zone1 = reverseproxynet.xyz:/subscriptions/beb6bvvd-d251-45be-a1db-f26b87da91e/resourceGroups/test
```

**NOTE**: dns_azure_zone1 is made in this way: `<domain>/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>`
3. Restart the docker container: `docker restart swag`.
4. If everything is setup correctly and all the above steps has been correctly performed, by reaching the domain with the browser: `https://<YOUR-DOMAIN>` you should visualize the SWAG landing page.
![](/assets/images/images_2024-02-10-azure-swag/swag_landing_page.png)

Final NOTE: If you want to change the aspect of the SWAG landing page (with 404 page or something else), just change the HTML page inside the swag_data volue: `./config/www/index.html`.
Info and example [here](https://libertus.dev/posts/configure-traefik-to-use-custom-domain/part2/).
