---
title: "Deploying NixOS in AWS using Terraform"
date: 2025-06-09T12:32:56+09:30
---
Recently for my day job I was tasked with assisting an external developer deploy their AI application into our cloud. The developer was using IaC to deploy the entire stack and this prompted me to experiment with IaC on the cloud in my own time. Naturally the first thing I wanted to do was deploy NixOS to the cloud and since my public DNS was hosted on AWS I decided to utilize their virtualisation platform EC2. I also wanted to keep with IaC tradition and deploy everything with code so I also utilized the [terraninx project](https://terranix.org) to keep everything in Nix code. Using Terranix will also allow me to turn my AWS Terraform deployment into a NixOS module making future deployments even easier.

This post will walk through my experience in deploying NixOS to AWS, the problems I encountered and how I overcame them.

My first instinct was to use [nixos-generators](https://github.com/nix-community/nixos-generators) to generate an AMI (Amazon Machine Image) with my NixOS baseline config but after looking at the high-level process I decided against it because it seemed like a lot of work. I instead opted to use the NixOS AMI image built by the NixOS team and available on the AMI marketplace.

Firstly we will need access to the Terranix module which we can achieve by adding it to our flake inputs and passing it down to our hosts configuration.nix file through the use of specialArgs which basically allows us to make whatever variables available in a host's configuration.nix's function parameter's.
{{< highlight nix >}}
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";

    # Add Terranix to our flake inputs
    terraninx = {
      url = "github:terranix/terranix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = {
    nixpkgs,
    terranix,  # <-- Add terranix as outputs parameter to be used by our NixOS hosts
    ...
  }: let
    system = "x86_64-linux";  # <-- Our host platform architecture
    pkgs = nixpkgs.legacyPackages.${system}  # <-- Make the pkgs attribute of nixpkgs easier to access
  in {
    nixosConfigurations.myserver = nixpkgs.lib.nixosSystem {
      specialArgs = {
        inherit terranix system pkgs;  # <-- Here we inherit terranix, system and pkgs so it's available in our configuration.nix file as a function parameter
      };
      modules = [
        ./configuration.nix
      ];
    };
  };
}
{{< /highlight >}}

{{< highlight nix >}}
# configuration.nix
{
  terranix,  # <-- Terranix variable available now because we inherited value through specialArgs
  system,    # <-- Same with system variable...
  pkgs,      # <-- Same with pkgs variable...
  ...
}:

{

  # rest of system configuration...

}
{{< /highlight >}}

We can now use Terranix in our system configuration and have it stored in our NixOS configuration. The idea here is that I'll have a NixOS host that will store my AWS infrastrucure config/state and that same host will have the necessary keys to create those AWS resources.

To keep our infrastrucure config seperate from our NixOS system configuration I'll create a seperate file and import it like so.
{{< highlight nix >}}
# configuration.nix
{
  terranix,
  pkgs,
  system,
  ...
}:

{
  imports = [
    (import ./infra.nix {inherit pkgs terranix system;})
  ];
}
{{< /highlight >}}

{{< highlight nix >}}
# infra.nix
{
  pkgs,
  terranix,
  system
  ...
} let

  # Here is where we'll define our AWS infrastructure

in {

  # Here is where we'll define a systemd unit that will run the terraform commands any time our infrastrucure config changes

}
{{< /highlight >}}

Now we have the following 3x files:
- ./flake.nix
- ./configuration.nix
- ./infra.nix

In our infra file we can start defining the AWS infrastrucure. The following code will make an AWS EC2 instance available in my AUS region and setup the necessary networking, ACL's, keypairs, AMI's and the VM instance.

NOTE: To find the AMI information I went [here](https://nixos.github.io/amis/).

{{< highlight nix >}}
# infra.nix
{
  pkgs,
  terranix,
  system,
  ...
}: let
  terraformConfiguration = terranix.lib.terranixConfiguration {
    inherit system;
    modules = [
      {
        terraform.required_providers.aws.source = "hashicorp/aws";

        provider.aws = {
          shared_credentials_files = ["/home/<user>/.aws/credentials"];  # <-- change
          shared_config_files = ["/home/<user>/.aws/config"];  # <-- change 
          region = "ap-southeast-2";  # <-- change
        };

        data.aws_ami.nixos-x86_64 = {
          owners = ["427812963091"];
          most_recent = true;
          filter = [
            {
              name = "name";
              values = ["nixos/25.05*"];
            }
            {
              name = "architecture";
              values = ["x86_64"];
            }
          ];
        };

        resource = {
          aws_instance.nixos_x86-64 = {
            ami = "\${ data.aws_ami.nixos-x86_64.id }";
            instance_type = "t2.medium";
            subnet_id = "\${ aws_subnet.nix-subnet.id }";
            associate_public_ip_address = true;
            key_name = "\${ aws_key_pair.default.key_name }";
            vpc_security_group_ids = [
              "\${ aws_security_group.default.id }"
            ];
            root_block_device = {
              volume_size = "30";
            };
          };

          aws_key_pair.default = {
            key_name = "default-key";
            public_key = "<ssh-public-key>";  # <-- change
          };

          aws_vpc.nix-vpc = {
            cidr_block = "10.10.0.0/24";
          };

          aws_subnet.nix-subnet = {
            vpc_id = "\${ aws_vpc.nix-vpc.id }";
            cidr_block = "10.10.0.0/25";
            availability_zone = "ap-southeast-2a";  # <-- change
          };

          aws_internet_gateway.nix-gateway = {
            vpc_id = "\${ aws_vpc.nix-vpc.id }";
          };

          aws_route_table.nix-route-tb = {
            vpc_id = "\${ aws_vpc.nix-vpc.id }";
          };

          aws_route_table_association.nix-subnet-route-tb-association = {
            subnet_id = "\${ aws_subnet.nix-subnet.id }";
            route_table_id = "\${ aws_route_table.nix-route-tb.id }";
          };

          aws_route.internet-route = {
            destination_cidr_block = "0.0.0.0/0";
            route_table_id = "\${ aws_route_table.nix-route-tb.id }";
            gateway_id = "\${ aws_internet_gateway.nix-gateway.id }";
          };

          aws_security_group.default = {
            vpc_id = "\${ aws_vpc.nix-vpc.id }";

            ingress = [
              {
                description = "Allow SSH in";
                from_port = 22;
                to_port = 22;
                protocol = "tcp";
                cidr_blocks = ["0.0.0.0/0"];
                ipv6_cidr_blocks = [];
                prefix_list_ids = [];
                security_groups = [];
                self = false;
              }
            ];

            egress = [
              {
                description = "Allow all traffic out";
                from_port = 0;
                to_port = 0;
                protocol = "-1";
                cidr_blocks = ["0.0.0.0/0"];
                ipv6_cidr_blocks = [];
                prefix_list_ids = [];
                security_groups = [];
                self = false;
              }
            ];
          };
        };
      }
    ];
  };
in {
  systemd.services.aws-infra-provisioner = {
    wantedBy = ["multi-user.target"];
    after = ["network.target"];
    path = [pkgs.git];
    serviceConfig.ExecStart = toString (pkgs.writers.writeBash "generate-aws-json-config" ''
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

The above defines an EC2 instance along with it's required networking resources as well as a systemd unit to run terraform against AWS whenever there's a change in the configuration. One thing to note is that I'm using OpenTofu instead of terraform which is a fork of terraform due to Hashicorp making the license more restricted.

The only things you will likely need to change are the region, the location of your AWS credentials/config, and the SSH public key.

Now upon rebuilding the host system with "nixos-rebuild --switch" the systemd unit will activate, generate a JSON config using Terranix, and run OpenTofu (terraform) against AWS with your specified credentials.

To improve this setup, I'll look at using a terraform module ([like this one](https://github.com/terraform-aws-modules/terraform-aws-ec2-instance)) instead of manually defining the AWS resources and also make it a NixOS module.
