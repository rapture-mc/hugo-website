---
title: "Whonix on NixOS"
date: 2025-04-17T18:54:35+09:30
tags:
- nixos
- whonix
---
[Whonix](https://www.whonix.org/) is an operating system that is built for privacy. Out of all the privacy based operating systems it's considered the most superior but also the most difficult to use. Some competence of Linux/Virtual Machine administration and use of anonymizing technooliges like PGP and Tor are required to operate Whonix effectively. 

Using NixOS the work involved with installing Whonix can be automated and reproduced thanks to NixOS's declarative nature. With a few scripts and some Nix utilities we can make setting up Whonix a straight forward process.

The typical process for installing Whonix involves:
- Installing a hypervisor (VirtualBox or QEMU)
- Downloading and importing the Whonix virtual disk
- Connecting the Whonix Gateway to the Tor network
- Applying latest patches Whonix Gateway/Workstation

Instead of doing this manually we can automate it with some bash scripts and NixOS magic.

To install Virtualbox on NixOS it's as simple as including this one line in your configuration.nix file:
{{< highlight nix >}}
{
  # omitted for brevity

  virtualisation.virtualbox.host.enable = true;
}
{{< /highlight >}}

To download and import NixOS we can create a Bash script that checks for the existence of Whonix and if it doesn't exist download and import the virtual disk image. We can only make this command available on our system by utilizing "pkgs.writeShellScriptBin".
{{< highlight nix >}}
{ pkgs, ... }: let

whonixVersion = "17.2.3.7";

installWhonix = pkgs.writeShellScriptBin "installWhonix" ''
  echo -e "This script will check for the existence of Whonix and if not found download Whonix from the internet and import it into VirtualBox"

  while true; do
    read -p "Continue? (y/n)" response
    case $response in
      [Yy]* )
        if ! VBoxManage list vms | grep -q "Whonix" && [ -e /tmp/Whonix-Xfce-${whonixVersion}.ova ]; then
          echo "Whonix VMs don't exist, importing..."

          VBoxManage import /tmp/Whonix-Xfce-${whonixVersion}.ova --vsys 0 --eula accept --vsys 1 --eula accept

        elif ! VBoxManage list vms | grep -q "Whonix" && [ ! -e /tmp/Whonix-Xfce-${whonixVersion}.ova ]; then
          echo "Whonix VMs don't exist and Whonix OVA file doesn't exist, downloading OVA file..."

          wget https://download.whonix.org/ova/${whonixVersion}/Whonix-Xfce-${whonixVersion}.ova -O /tmp/Whonix-Xfce-${whonixVersion}.ova

          VBoxManage import /tmp/Whonix-Xfce-${whonixVersion}.ova --vsys 0 --eula accept --vsys 1 --eula accept

          echo -e "Import successful!\n Cleaning up OVA file from /tmp folder..."

          rm /tmp/Whonix-Xfce-${whonixVersion}.ova

        else
          echo "Whonix VMs already exist, skipping..."

          exit 1

        fi

        echo  "Done!"; break;;
      [Nn]* ) exit;;
      * ) echo "Please answer y or n.";;
    esac
  done
'';
in {
  environment.systemPackages = [
    installWhonix
  ];
}
{{< /highlight >}}

This will make the command "installWhonix" available on the CLI and ensure Whonix is installed and install it if not.

We can also streamline launching Whonix making it more user friendly by taking advantage of a Linux feature called [desktop entries](https://wiki.archlinux.org/title/Desktop_entries). This will enable us to create a searchable application that can run any program (or script) that we want. NixOS's popular [home-manager](https://github.com/nix-community/home-manager) project has an option called "xdg.desktopEntries" which will let us create our own desktop entries the Nix way.

To begin we first need to define the programs that the desktop entries will call when launched.

{{< highlight nix >}}
{ pkgs, ... }: let
  startWhonix = pkgs.writeShellScriptBin "startWhonix" ''
    VBoxManage startvm Whonix-Gateway-Xfce --type headless

    sleep 1

    VBoxManage startvm Whonix-Workstation-Xfce
  '';

  stopWhonix = pkgs.writeShellScriptBin "stopWhonix" ''
    VBoxManage controlvm Whonix-Workstation-Xfce poweroff

    VBoxManage controlvm Whonix-Gateway-Xfce poweroff
  '';
in {
  environment.systemPackages = [
    startWhonix
    stopWhonix
  ];
}
{{< /highlight >}}

Above we once again utilize the "writeShellScriptBin" function to create a custom script which calls the VirtualBox cli tool to start the Whonix Gateway headless (no GUI) and allow it to boot for one second before calling the Whonix Workstation. We also create the corresponding script to stop both virtual machines and then add them to our global system packages.

Finally we then create desktop entries using home-manger that calls these scripts so the user doesn't have to open up a terminal and call the scripts manually.
{{< highlight nix >}}
_:  # <-- Shorthand for "{ ... }:"

{
  xdg.desktopEntries = {
    StartWhonix = {
      name = "Start Whonix";
      exec = "startWhonix";
      terminal = false;
    };

    StopWhonix = {
      name = "Stop Whonix";
      exec = "stopWhonix";
      terminal = false;
    };
  };
}
{{< /highlight >}}

After rebuilding the system users can then search for "Start Whonix" and "Stop Whonix" in the desktop's search bar instead of launching VirtualBox and starting/stopping the 2x virtual machines!
