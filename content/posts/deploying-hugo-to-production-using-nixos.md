---
title: "Deploying Hugo to Production Using NixOS"
date: 2025-04-02T16:30:57+09:30
tags:
- nixos
- hugo
---
Now that I have a minimal viable website I need to deploy it to production. For this I wanted to use a simple nginx web server running on NixOS. Using this method is free, easy and reproducible.

To start I created a new NixOS VM and added the following nginx config to my configuration.nix:
{{< highlight nix >}}
# contents of /etc/nixos/configuration.nix

{
  ... <omitted for brevity>

  networking.firewall.allowedTCPPorts = [80];

  services.nginx = {
    enable = true;
    virtualHosts."megacorp.industries" = {
      root = "/var/www/megacorp.industries";
    };
  };
}
{{< /highlight >}}
The above code opens TCP port 80 and sets up the nginx service and listens for requests for the domain "megacorp.industries" and will serve files from the "/var/www/megacorp.industries" folder over HTTP on port 80. Note that this code doesn't do any SSL certificates. For that add the following code to the same config file to have the vm also handle SSL certificates like so:
{{< highlight nix >}}
# contents of /etc/nixos/configuration.nix

{
  ... <omitted for brevity>

  networking.firewall.allowedTCPPorts = [80 443];

  services.nginx = {
    enable = true;
    virtualHosts."megacorp.industries" = {
      forceSSL = true;
      enableACME =true;
      root = "/var/www/megacorp.industries";
    };
  };

  security.acme = {
    acceptTerms = true;
    defaults.email = "replaceme@somedomain.com";
  };
}
{{< /highlight >}}
We have now opened port 443 (for HTTPS) and told nginx to force connections to use SSL (HTTPS) with automatic let's encrypt ACME certificates. Note that the proper DNS and/or port-forwarding must be setup prior or else the ACME service will fail to fetch the SSL certificates and only self-signed certificates will be setup. This is dependent on each setup but for me I had to do the following:
- Add an A record with my public IP to my domain "megacorp.industries"
- Set a port forward on my router/firewall (aka my public IP) to forward all traffic on TCP 80/443 to my vm's private IP
- Explicitly allow HTTP/S traffic inbound from everywhere on my firewall policy settings. I initially configured only Australia traffic to be allowed however the servers issuing the SSL certificates were coming from different countries and thus my vm was failing to receive the certificates.

Now that we've got a functioning web server with SSL certificates we can configure nginx to serve our hugo site. We configured NixOS to serve files from /var/www/megacorp.industries so we need to create this folder.
```
sudo mkdir -p /var/www/megacorp.industries
```
And change the permissions so nginx owns the directory...
```
sudo chown nginx:nginx /var/www/megacorp.industries
```
Now we change to our hugo site directory and run this command which will use hugo to generate our public/ directory, copy the contents to our nginx directory and update the permissions. If our hugo files were on a different machine to our nginx server we'd have to specify a remote location instead of a local one when using the rsync command.
```
hugo && sudo rsync -avz --delete public/ /var/www/megacorp.industries && sudo chown -R nginx:nginx /var/www/megacorp.industries
```
And that's it! If I change anything in my hugo site I just need to run the last command and it will update!
