---
title: "Declarative VMs With NixOS"
date: 2025-05-12T13:58:59+09:30
---
One thing that I've struggled with since adopting NixOS over a year ago was somehow integrating my virtual machine infrastructure into my NixOS deployments. I wanted to be able to define the virtual machine infrastructure in Nix code exactly how you would define a NixOS system in Nix code too so when you would rebuild the NixOS system it would also create/update any virtual machine infrastructure as well.

In researching how this could be achieved I came across a few existing soltuions.
- [NixVirt](https://github.com/AshleyYakeley/NixVirt)
- [MicroVM](https://github.com/astro/microvm.nix)
- [vms](https://github.com/Nekroze/vms.nix/tree/master)


While the aforementioned solutions are mature and battle-tested they did not work for my circumstances. Instead I opted to create a solution that utilized [Terraform](https://terraform.io) technology combined with Nix.

## Preparing the host/hypervisor

### Installing virtualisation packages

The first thing I needed to do is install the KVM/QEMU/Libvirt stack on a machine that will run the virtualisation infrastructure. Using Nix this can be achieved by including the following code in our system configuration.
{{< highlight nix >}}
# configuration.nix
{ pkgs, ... }:

{
  virtualisation.libvirtd = {
    enable = true;
    onBoot = "start";
    onShutdown = "shutdown";
    qemu.ovmf.enable = true;
  };

  environment.sessionVariables = {
    LIBVIRT_DEFAULT_URI = "qemu:///system";
  };

  environment.systemPackages = with pkgs; [
    libxslt
    opentofu
    virt-manager
  ];

  # rest of system configuration code...
}
{{< /highlight >}}

The above code will install and configure the KVM kernel module, QEMU emulator software and Libvirt API (all of which together provide a full virtualisation solution). It also installs some extra libvirt packages as well as the open-source fork of Terraform "OpenTofu". We use OpenTofu instead of Terraform because Terraform's license is now marked as unfree in nixpkgs.

### Setting up networking

Next I wanted to have my virtual machines appear as their own standalone hosts on my private subnet (no host-to-vm NAT) which means I needed to turn one of my physical nterfaces on the hypervisor into a bridge. Since my hypervisor was just a refurbished desktop I only had one NIC available to me so management and VM traffic would be on the same interface. Ideally this would be seperated but for a home-lab environment this will have to suffice.

{{< highlight nix >}}
# configuration.nix
{ pkgs, lib, ... }:

{
  networking = {
    networkmanager.enable = lib.mkForce false;
    domain = "megacorp.industries";  # <--- Change this
    useDHCP = false;
    useNetworkd = true;
  };

  services.resolved = {
    llmnr = "false";
  };

  systemd.network = {
    enable = true;

    netdevs.br0 = {
      enable = true;
      netdevConfig = {
        Kind = "bridge";
        Name = "br0";
      };
    };

    networks = {
      br0-lan = {
        enable = true;
        matchConfig = {
          Name = ["br0"];
        };
        networkConfig = {
          Bridge = "br0";
        };
      };

      br0-lan-bridge = {
        enable = true;
        matchConfig = {
          Name = "br0";
        };

        networkConfig = {
          DHCP = "no";
          Address = "192.168.1.15/24";  # <--- Change this
          DNS = ["192.168.1.45"];  # <--- Change this
          Domains = "megacorp.industries";  # <--- Change this
        };

        routes = [
          {
            Destination = "0.0.0.0/0";
            Gateway = "192.168.1.99";  # <--- Change this
          }
        ];
      };

      enp6s0 = {
        matchConfig = {
          Name = "enp6s0";  # <--- Change this
        };

        networkConfig = {
          Bridge = "br0";
          DHCP = "no";
          DNS = ["192.168.1.45"];  # <--- Change this
          Domains = "megacorp.industries";  # <--- Change this
        };
      };
    };
  };
}
{{< /highlight >}}

The above configuration disables DHCP and NetworkManager and uses systemd networking to create a bridge interface "br0" and enslave my physical interface "enp6s0" to it. If adopting this configuration be sure to change the IP and DNS info accordingly as indicated by the comments.

I also had to disable LLMNR due to local name resolution bugs.

**Note:** If connected to the hypervisor over SSH you will temporarily lose connectivity since you're changing the interface you're currently connected to.

## Creating Terraform config in Nix

The next step is to somehow describe the libvirt infrastructure in code in a way that will be tied together with the NixOS hypervisor system configuration. I managed to get this working in 2 steps.

Firstly I was able to use Duncan Mac-Vicar's [terraform libvirt module](https://github.com/dmacvicar/terraform-provider-libvirt) to declaratively describe libvirt infrastructure using Terraform code.

Secondly I utilizied the Nix project [Terranix](https://terranix.org) which provides the ability to write Terraform code in Nix.

Combining these 2x technologies (with a little systemd magic) enables the ability to declare libvirt infrastructure in my NixOS system configuration and have the infrastructure update (if there are any changes) with system rebuilds.

Since I'm using flakes I only need to add the Terranix repo to my inputs...
{{< highlight nix >}}
# flake.nix
{
  inputs = {
    nixpkgs = "github:nixos/nixpkgs/nixos-24.11";

    terranix = {
      url = "github:terranix/terranix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };
}
{{< /highlight >}}

I then add it as a function argument to my flakes outputs and add it to specialArgs as well as my system architecture variable...
{{< highlight nix >}}
# flake.nix
{
  outputs = { self, nixpkgs, terranix, ... }: let
    system = "x86_64-linux";
  in {
    nixosConfigurations = nixpkgs.lib.nixosSystem {
      specialArgs = {
        inherit terranix system;
      };
      modules = [
        ./configuration.nix
      ];
    };
  };
}
{{< /highlight >}}

And pass the Terranix module + system variable to my machines configuration file as function arguments...
{{< highlight nix >}}
# configuration.nix
{ pkgs, terranix, system, ... }:

{

}
{{< /highlight >}}

Now that Terranix is ready to use in the system configuration I can begin implementing the logic. We need to ensure that when I rebuild the NixOS system any changes made to our Terraform (Terranix) code also update when NixOS is rebuilt.

First I'll define the Terranix code in a "let in" block.

**Note:** I'm using a modified version of [this terraform module](https://github.com/MonolithProjects/terraform-libvirt-vm) which abstracts much of the options in the [libvirt terraform provider](https://github.com/dmacvicar/terraform-provider-libvirt) but you can also easily substitute my terraform module decleration with the options in the aforementioned provider.
{{< highlight nix >}}
# configuration.nix
{ pkgs, terranix, system, ... }: let
  terraformConfiguration = terranix.lib.terranixConfiguration {
    inherit system;
    modules = [
      {
        terraform.required_providers.libvirt.source = "dmacvicar/libvirt";

        provider.libvirt.uri = "qemu:///system";

        module = {
          testvm = {
            source = "https://github.com/rapture-mc/terraform-libvirt-module";
            vm_hostname_prefix = "testvm";
            uefi_enabled = false;
            autostart = true;
            vm_count = 1;
            memory = "2048";
            vcpu = 1;
            system_volume = 100;
            bridge = "br0";
            dhcp = true;
          };
        };
      }
    ];
  };
in {

}
{{< /highlight >}}

Then add the systemd service which will create a directory to store out infrastructure state at /terraform-state and run "opentofu init && opentofu apply -auto-approve" whenever there's a change in our terranix code.
{{< highlight nix >}}
# configuration.nix
{ pkgs, terranix, system, ... }: let
  terraformConfiguration = terranix.lib.terranixConfiguration {
    inherit system;
    modules = [
      {
        terraform.required_providers.libvirt.source = "dmacvicar/libvirt";

        provider.libvirt.uri = "qemu:///system";

        module = {
          testvm = {
            source = "https://github.com/rapture-mc/terraform-libvirt-module";
            vm_hostname_prefix = "testvm";
            uefi_enabled = false;
            autostart = true;
            vm_count = 1;
            memory = "2048";
            vcpu = 1;
            system_volume = 100;
            bridge = "br0";
            dhcp = true;
          };
        };
      }
    ];
  };
in {
  systemd.services.libvirt-infra = {
    wantedBy = ["multi-user.target"];
    after = ["network.target"];
    path = [pkgs.git];
    serviceConfig.ExecStart = toString (pkgs.writers.writeBash "generate-terranix-config" ''
      if [ -d /terraform-state ]; then
        echo "/terraform-state directory exists... Skipping creating dir"
      else
        echo "/terraform-state directory doesn't exist... Creating dir"
        mkdir /terraform-state
      fi

      cd /terraform-state

      if [[ -e config.tf.json ]]; then
        rm -f config.tf.json;
      fi
      cp ${terraformConfiguration} config.tf.json \
        && ${pkgs.opentofu}/bin/tofu init \
        && ${pkgs.opentofu}/bin/tofu apply -auto-approve
    '');
  };
}
{{< /highlight >}}
Now we can rebuild our system and our libvirt infrastructure will be built. We can check whether our virtual machines are active by running "virsh list" in a terminal on the hypervisor host. If we need to troubleshoot and check logs we can run "systemctl status libvirt-infra" or "journalctl -u libvirt-infra".

As of 2025-05-14 I haven't been able to get UEFI working in the virtual machine. Whenever I try to reference the OVMF code file at "/run/libvirt/nix-ovmf/OVMF_CODE.fd" I get the following error:
> *Error: error creating libvirt domain: operation failed: unable to find any master var store for loader: /run/libvirt/nix-ovmf/OVMF_CODE.fd*

Despite the file "/run/libvirt/nix-ovmf/OVMF_CODE.fd" existing on the hypervisor. Strangely enough I am able to reference that OVMF file when manually creating a libvirt machine using virt-manager.
