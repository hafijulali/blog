---
title: "Cloudflare Tunnel SSH"
date: 2025-08-16
draft: false
---

# Accessing private network ssh using Cloudflare Tunnels

This guide will walk you through the steps to configure a Cloudflare Tunnel to access your device using a public hostname via SSH.

## Prerequisites

- A Cloudflare account
- A domain name managed by Cloudflare
- Cloudflare Tunnel installed on your private network device


## Step 1: Configure the Tunnel 

1. **Navigate to Cloudflare Zero Trust dashboard**

2. **From Networks select Tunnels**

3. **From list of tunnels click on 3dot menu and goto configure**

4. **Add public hostname with details as:**
```
    subdomain: <SUBDOMAIN>
    domain: <YOUR_DOMAIN>
    service: ssh://localhost:22
```


## Step 2: Access Your Device via SSH

Now that your Cloudflare Tunnel is set up, you can access your device using the following SSH command:
```bash
ssh -o "ProxyCommand=cloudflared access ssh --hostname %h" <sshuser>@<SUBDOMAIN>.<YOUR_DOMAIN>
```

Replace `<sshuser>` with your SSH username and `<SUBDOMAIN>.<YOUR_DOMAIN>` with the hostname you configured.

**Note:- This may prompt you `cloudflared: command not found` you need to install cloudflared on the system where you are trying to access from**

## Conclusion

You have successfully configured a Cloudflare Tunnel to access your device using a public hostname via SSH. If you encounter any issues, refer to the Cloudflare documentation or seek assistance from the community.
