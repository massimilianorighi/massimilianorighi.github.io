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

![](/assets/images/images_2024-02-10-azure-swag/subscriptionid.png =200x)

#### dns_azure_tenant_id
On Azure portal search for `Microsoft Entra ID`, then you should see a page like the one below that contains basic information about your tenant.
- Tenant ID: `d8x72dd3-f729-4fe1-0f6e-12dcb6c82d5c`


## SWAG setup

Embed code by putting `{{ "{% highlight language " }}%}` `{{ "{% endhighlight " }}%}` blocks around it. Adding the parameter `linenos` will show source lines besides the code.

{% highlight c %}

static void asyncEnabled(Dict* args, void* vAdmin, String* txid, struct Allocator* requestAlloc)
{
    struct Admin* admin = Identity_check((struct Admin*) vAdmin);
    int64_t enabled = admin->asyncEnabled;
    Dict d = Dict_CONST(String_CONST("asyncEnabled"), Int_OBJ(enabled), NULL);
    Admin_sendMessage(&d, txid, admin);
}

{% endhighlight %}