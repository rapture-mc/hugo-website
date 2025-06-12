---
title: "Fixing NixOS Guacamole With Overlays"
date: 2025-06-12T19:09:09+09:30
---
In this post I detail how I used overlays to fix a bug with RDP in the nixos guacamole server package where the RDP protocol is no longer functional.

Bug: Whenever you try to access a guacamole client through RDP it times out refusing to connect. Upon checking the guacamole-server logs you find the error:
> 'Support for protocol "rdp" is not installed'

When you google this error along with the term "nixos" you will find [this github issue](https://github.com/NixOS/nixpkgs/issues/395919) where a user reported the bug. Related to the issue is [this pull request](https://github.com/NixOS/nixpkgs/pull/407726) which references [this commit](https://github.com/NixOS/nixpkgs/commit/da303f71c4f9673a7d718396fb23f74679ae4fb0) which contains fixes for the [freerdp](https://github.com/NixOS/nixpkgs/commit/da303f71c4f9673a7d718396fb23f74679ae4fb0#diff-080939340c91a8f001aa4528a8feb3fca424ba1d8d25d95b2a459190a672c9c0) and [guacamole-server](https://github.com/NixOS/nixpkgs/commit/da303f71c4f9673a7d718396fb23f74679ae4fb0#diff-476d095db70babe67a8febd74ed3fb5178e0b0ecb5e1abad8b2ce9d6bdb7e45f) packages. I initially applied this fix for my machines by pinning the commit containing the fix in my nixpkgs inputs like below however I thought this would be a good oppurtuning to learn nix overlays (which I have been putting off learning for a while).
{{< highlight nix >}}

# ./flake.nix
{
  inputs = {
    nixpkgs = {
      type = "github";
      owner = "nixos";
      repo = "nixpkgs";
      ref = "nixos-25.05";
      rev = "da303f71c4f9673a7d718396fb23f74679ae4fb0";
    };
  };

  outputs = { nixpkgs, ... }: {

    # nixos outputs...

  };
}
{{< /highlight >}}

I started learning as much as I could about overlays and I'll admit it took a few days to just grasp the concepts of overlays let alone actually building them. [This article](https://nixos-and-flakes.thiscute.world/nixpkgs/overlays) as well as Google's Gemini helped heaps with understanding how to practically use overlays and Gemini helped with troubleshooting why things weren't working when they inevitably wouldn't.

First I began working on the overlays themselves. The overlay for the guacamole-server is below. All I did was make the same changes in the previously mentioned commit and replicated it in the overlay below.

{{< highlight nix >}}
# ./overlays/guacamole-server.nix
final: prev: {
  guacamole-server = prev.guacamole-server.overrideAttrs (finalAttrs: previousAttrs: {
    src = prev.fetchFromGitHub {
      owner = "apache";
      repo = "guacamole-server";
      rev = "acb69735359d4d4a08f65d6eb0bde2a0da08f751";
      hash = "sha256-rqGSQD9EYlK1E6y/3EzynRmBWJOZBrC324zVvt7c2vM=";
    };

    patches = [];
  });
}
{{< /highlight >}}

In the above overlay I...
- Updated the src repository's rev and hash with the updated values (even though the owner and repo didn't change in the commit we still have to add them because the overlays src attribute will overwrite the existing nixpkgs src attribute)
- Make the patches attribute an empty list. In the commit the whole patches attribute was deleted (including the patches attribute itself) so I just made it an empty list which achieves the same thing

Next I can create the freerdp package overlay.
{{< highlight nix >}}
# ./overlays/freerdp.nix
final: prev: {
  freerdp = prev.freerdp.overrideAttrs (finalAttrs: previousAttrs: {
    patches = [
      (prev.fetchpatch2 {
        url = "https://github.com/FreeRDP/FreeRDP/commit/67fabc34dce7aa3543e152f78cb4ea88ac9d1244.patch";
        hash = "sha256-kYCEjH1kXZJbg2sN6YNhh+y19HTTCaC7neof8DTKZ/8=";
      })
    ];

    postPatch =
      ''
        # skip NIB file generation on darwin
        substituteInPlace "client/Mac/CMakeLists.txt" "client/Mac/cli/CMakeLists.txt" \
          --replace-fail "if(NOT IS_XCODE)" "if(FALSE)"

        substituteInPlace "libfreerdp/freerdp.pc.in" \
          --replace-fail "Requires:" "Requires: @WINPR_PKG_CONFIG_FILENAME@"

        substituteInPlace client/SDL/SDL2/dialogs/{sdl_input.cpp,sdl_select.cpp,sdl_widget.cpp,sdl_widget.hpp} \
          --replace-fail "<SDL_ttf.h>" "<SDL2/SDL_ttf.h>"
      ''
      + prev.lib.optionalString (prev.pcsclite != null) ''
        substituteInPlace "winpr/libwinpr/smartcard/smartcard_pcsc.c" \
          --replace-fail "libpcsclite.so" "${prev.lib.getLib prev.pcsclite}/lib/libpcsclite.so"
      '';

    nativeBuildInputs = previousAttrs.nativeBuildInputs ++ [
      prev.writableTmpDirAsHomeHook
    ];
  });
}
{{< /highlight >}}

The above overlay is very similar to the first overlay. I just applied the same changes in the commit to the overlay.

Now that the overlays are created I can import them under the nixpkgs.overlays option like so.

{{< highlight nix >}}
# ./flake.nix
{
  inputs.nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05"; 

  outputs = { nixpkgs, ... }: let
    system = "x86_64-linux";
    pkgs = import nixpkgs {
      inherit system;
      overlays = [
        (import ./overlays/freerdp.nix)
        (import ./overlays/guacamole-server.nix)
      ];
    };

  in {
    nixosConfigurations.guacamole-server = {
      specialArgs = {
        inherit pkgs;
      };
      modules = [
        ./configuration.nix
      ];
    };
  };
}
{{< /highlight >}}

Above I...
- Set my hosts system architectureto "x86_64-linux"
- Declare the variable **pkgs** which import nixpkgs for my system architecture along with the overlays I defined earlier
- Passed the **pkgs** variable down into the context of my configuration.nix file through the specialArgs option

The final step is to actually apply the overlays to my host configuration. I won't lie this part took a while to figure out but essentially at this point we have our modified packages **freerdp** and **guacamole-server** however we also need to tell our NixOS machine to **use** the packages. This is done by explicitly declaring them (pkgs.freerdp and pkgs.guacamole-server) in our hosts configuration.nix file like so.

{{< highlight nix >}}
# ./configuration.nix

{ config, pkgs, ... }: {
  imports = [
    ./hardware-configuration.nix
  ];

  environment.systemPackages = [
    pkgs.freerdp
  ];

  services.guacamole-server = {
    enable = true;
    package = pkgs.guacamole-server;
    # other guac options...
  };
}
{{< /highlight >}}

And now rebuild your system and your done! Your guacamole instance will now be using your custom overlays with the RDP fix.
