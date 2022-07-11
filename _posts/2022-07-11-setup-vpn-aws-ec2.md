---
layout: post
title: Install VPN server on AWS EC2
subtitle: 
header-img: img/in-post/2022-07-11/setup-vpn.jpg
header-style: text
catalog: true
tags:
  - cloud
---

# How to access to EC2 with private IP using Pritunl?

## Install Pritunl
Prepare a EC2 machine with security group, key pair to authenticate when SSH, VPC, Public subnet, target group.
Bootstrap script below:

```bash
#!/bin/bash
sudo apt update -y
sudo apt upgrade -y
echo "deb http://repo.pritunl.com/stable/apt bionic main" | sudo tee /etc/apt/sources.list.d/pritunl.list
echo "deb https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.0.list
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv 9DA31620334BD75D9DCB49F368818C72E52529D4
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv 7568D9BB55FF9E5287D586017AE645C0CF8E292A
sudo apt update -y
sudo apt install pritunl mongodb-server -y
sudo systemctl start pritunl mongodb
sudo systemctl enable pritunl mongodb
sudo sh -c 'echo "* hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "* soft nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root soft nofile 64000" >> /etc/security/limits.conf'
```

This script is run on Ubuntu 18.04. If you have any problem with python ($PYTHONHOME or $PYTHONPATH), you can try with script below:

```bash
#!/bin/bash
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get install curl gnupg2 wget unzip -y
curl -fsSL https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
sudo apt-get update
sudo apt-get install mongodb-server -y
sudo systemctl start mongodb
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv E162F504A20CDF15827F718D4B7C549A058F8B6B
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv 7568D9BB55FF9E5287D586017AE645C0CF8E292A
echo "deb http://repo.pritunl.com/stable/apt focal main" | sudo tee /etc/apt/sources.list.d/pritunl.list
sudo apt-get update
sudo apt-get install pritunl -y
sudo sudo systemctl start pritunl
sudo systemctl enable pritunl mongodb
```

## Configure Pritunl server on EC2 instance
After installing Pritunl completely, access Pritunl Web UI via URL: <b>https://PUBLIC_IP_EC2_MACHINE</b>

Connecting to a Pritunl vpn server follow this docs: [Connecting to Pritunl vpn server](https://docs.pritunl.com/docs/connecting)

## Config Load Balancer
If you want to assign domain name for Pritunl vpn server Web UI, you can config load balancer:

```bash
sudo pritunl set app.reverse_proxy true
sudo pritunl set app.redirect_server false
sudo pritunl set app.server_ssl false
sudo pritunl set app.server_port 80
```

Pritunl document: [Load balancing](https://docs.pritunl.com/docs/load-balancing)

## Create VPN user

After set up successfully, create vpn user and import profile to VPN client, such as OpenVPN, Pritunl,... then try it :smile:

## Some notes

Configure Inbound rule of Security Group:

![Configure security group](/img/in-post/2022-07-11/security-group.png)

Remove default server 0.0.0.0/0, replace by your private subnet.

![Config server](/img/in-post/2022-07-11/config-server.png)