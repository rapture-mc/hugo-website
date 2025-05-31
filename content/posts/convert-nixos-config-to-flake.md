---
title: "Convert NixOS Config to Flake"
date: 2025-04-01T10:16:13+09:30
tags:
- nixos
---
# EDIT: With NixOS 25.05 flakes is now stable and this article is only applicable pre 25.05...
When you first install NixOS from an ISO image you start out with a configuration.nix file and a hardware-configuration.nix file both of which are auto-generated for you. To modify the system configuration you change these files accordingly depending on what you want to do. For example to install the git application you would modify the configuration.nix file like so...
> Run "nano /etc/nixos/configuration.nix"

```
{
  # omitted for brevity

  environment.systemPackages = with pkgs; [
    nano
    git  <-- Add this line here to install git
  ];
}
```
> Then run "nixos-rebuild switch"

After the above finishes you will then have git installed on your system. A lot of magic goes on under the hood to make this happen but the main takeaway is that it uses a thing called "nix channels" which is equivalent to a linux package repository. It's what controls which package versions you receive when you decide to install a program (like git) among other things (like installing a service). Nix channels however aren't reproducible because they're configured on a per-system basis through the terminal and therefore have no way to "pin" channels consistently across systems. That's where flakes come in, a new experimental feature of Nix that has been widely embraced by the community. Flakes make it a breeze to handle nix dependecies and package versions in a reproducible way.

To enable flakes:
> Run "nano /etc/nixos/configuration.nix"
```
{
  # omitted for brevity

  nix.settings.experimental-features = [ "nix-command" "flakes" ];  <-- Add this line

}
```
> Then run "nixos-rebuild switch"

This both enables the flakes feature and also unlocks new terminal commands such as "nix shell" and "nix profile" (in contrast to the older "nix-shell" and "nix-env" commands). Where possible you should priortize learning the newer command syntax over the older ones. 

Now that flakes is enabled we can create our flake.nix file in our /etc/nixos directory.
> Run "nano /etc/nixos/configuration.nix"
```
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-24.11";
  };

  outputs = { nixpkgs, ... }:

  {
    nixosConfigurations = {
      myNixMachine = nixpkgs.lib.nixosSystem {
        modules = [
          ./configuration.nix
          ./hardware-configuration.nix
        ]; 
      };  
    };
  };
}
```
> Then run "nixos-rebuild switch"

Nix will then rebuild your system using the flake file.
