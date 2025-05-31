---
title: "Improved Deploying Hugo to Production Using Nixos"
date: 2025-05-31T12:25:20+09:30
---
This article builds on my [first article](https://megacorp.industries/posts/deploying-hugo-to-production-using-nixos/) where I deployed hugo using NixOS. There was still room for improvement with this process so I decided to make it more nix-y and have the deployment steps incorporated into the system configuration so when the machine hosting the website is built the website is rebuilt too.
