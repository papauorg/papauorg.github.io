---
layout: post
title:  "Docker behind authenticating proxy"
description: Pull docker images when running behind a corporate authenticating proxy with docker for windows.
categories: docker network proxy windows
---

I have been trying to pull docker images to my work machine for a long time now. It's such a pain because our desktops do not have direct internet access. Instead we need to use a proxy that requires authentication.

In docker for windows you can easily set a proxy. The problem is, if you want to use a proxy with authentication you need to specify the proxy with your domain user and the password, like so:
```
http://USER:PASSWORD@our-corporate-proxy:8080
```
The user and password are set in clear text which I'm not so comfortable with. I didn't investigate if it's possible in windows, too, but when running docker on a linux machine, you can set the proxy via environment variables ... in which I also don't want to set my password in clear text.

# CNTLM
I'm not aware of a way to get around the above mentioned problem without any additional tool. After a little online search I found the cntlm tool. In short it's a proxy that handles the communication and authentication with an upstream proxy so your tools won't have to.

To be able to authenticate to the upstream proxy you also need to specify the credentials. Other than in the proxy url you can at least specify an NTLM hash and not a clear text password.

## Configuration
After installing cntlm (in my case via `choco install cntlm`) it can be configured with the `cntlm.ini` file in the installation directory. First we need to create the hash that should go to the configuration file. This can be done with the following command in a cmd or powershell.
```powershell
cntlm.exe -H -u DOMAINUSER -d DOMAIN
```
This will prompt you for your password and will then print the NTLM hash you need for configuration.

```
PassLM      XXXXX
PassNT      XXXXX
PassNTLMv2  XXXXX
```
In my case I just needed the last one. So I copied that one and put it in the `cntlm.ini` configuration file at the appropriate location (where the samples are commented). Don't forget to change the user and domain settings. Also make sure the `Auth   NTLMv2` is set. It will look something like this at the beginning of the file:
```cntlm.ini
#
# Cntlm Authentication Proxy Configuration
#
# NOTE: all values are parsed literally, do NOT escape spaces,
# do not quote. Use 0600 perms if you use plaintext password.
#

Username	YOUR_USERNAME
Domain		YOUR_DOMAIN.com

# NOTE: Use plaintext password only at your own risk
# Use hashes instead. You can use a "cntlm -M" and "cntlm -H"
# command sequence to get the right config for your environment.
# See cntlm man page
# Example secure config shown below.
Auth            NTLMv2
PassNTLMv2      XXXXXXXXXX
```
Save the file and check if it works:

```powershell
cntlm.exe -M http://google.com
```
If everything is ok it should print something like
```
Config profile  1/4... OK (HTTP code: 301)
----------------------------[ Profile  0 ]------
Auth            NTLMv2
PassNTLMv2      XXXXXXXXX
------------------------------------------------
```
# Docker
If you start the cntlm windows service you can now use that proxy via `localhost:3128`. In docker for windows you can set it via the UI.

{% include image.html name="docker_proxy_settings.png" caption="Setting proxy in docker UI" %}

Unfortunately after setting this proxy and running e.g. `docker search grafana` I always received the following error.
<blockquote cite="docker error message">
Error response from daemon: Get https://index.docker.io/v1/search?q=grafana&n=25: proxyconnect tcp: dial tcp 10.0.75.1:3128: i/o timeout
</blockquote>

Hm, strange. Why does docker try to use `10.0.75.1:3128` instead of the localhost that I actually specified in the settings? Well I think this is because when using docker for windows, the docker host is actually hosted in a virtual machine - which is running on localhost, but it's not the same as localhost. It's in its own network and has its own IP address. Which can also be seen when doing an `ipconfig -all`.

{% include image.html name="ipconfig.png" caption="ipconfig showing docker NAT ethernet adapter with ip 10.0.75.1" %}

## Troubleshooting
So it seems, that cntlm does not accept connections from other machines, which is a good thing from the security perspective. So the internet connection that uses your credentials can at least only be used from your own machine. But unfortunately docker can't use it. The solution for this is to set the `Gateway` configuration in the `cntlm.ini` file to `yes`. This allows other machines to use the proxy.

```
# Enable to allow access from other computers
#
Gateway	yes
```

To improve security I used the windows firewall and restricted inbound connections to CNTLM (Port 3128) to specifically allow the IPs of the docker NAT (in my case 10.0.75.1-10.0.75.15). So that are the only IPs that can use the proxy. You can also specify allowed IPs in the `cntlm.ini` file but that didn't have any effect when I tried that. I probably did something wrong there. But it works fine with the firewall rules. Additionally I stop cntlm when I'm not actively using it to further reduce the possibility of misuse of this proxy.

# Finally working
After all those changes (and restarting the cntlm service) I can finally search and pull docker images from the docker registry. What a fight ...

{% include image.html name="docker_search_grafana.png" caption="working docker search command" %}