---
layout: post
title: "2019-03-14-deploy-react-app-on-AWS-EC2"
date: 2019-03-14
categories:Note
---

1. `sudo apt-get update && sudo apt-get upgrade -y`
2. `sudo apt-get install nginx -y`
3. `sudo systemctl start nginx`
4. `sudo systemctl enable nginx`
5. `sudo apt-get install build-essential libssl-dev`
6. ```bash
cd ~
curl -sL https://raw.githubusercontent.com/creationix/nvm/v0.33.6/install.sh -o install_nvm.sh
bash install_nvm.sh
source ~/.profile
nvm install 8.15.0
nvm use 8.15.0
```
7. `sudo vim /etc/nginx/sites-available/default`
8. in server block
```
location / {
 root /home/ubuntu/myReactApp/build;
 index index.html index.htm index.nginx-debian.html;
 }
 ```
 9. `sudo service nginx restart`
