Install Hexo
============

```bash
sudo npm install -g hexo-cli
npm install
```

Serve
=====

```bash
hexo server
```

Admin Panel
===========

Open [http://127.0.0.1:4000/admin](http://127.0.0.1:4000/admin)

Generate
========

```bash
hexo generate
```

Nginx Config
============

First create an nginx configuration, assuming
code is cloned as `/usr/share/nginx/ucoder.ir`

```
server {
  listen 80;
  server_name ucoder.ir;
  server_name www.ucoder.ir;

  root /usr/share/nginx/ucoder.ir/public;
}
```

Now install certbot to enable SSL

```bash
yum -y install yum-utils
yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional
sudo yum install certbot-nginx
```

Enable CertBot on our Nginx Configuration
```bash
sudo certbot --nginx
```

**OR IF YOU RUN INTO BUG PROBLEM**
```bash
sudo certbot --authenticator standalone --installer nginx --pre-hook "systemctl stop nginx" --post-hook "systemctl stop nginx"
```

Test if renewal procedure succeeds
```bash
sudo certbot renew --dry-run
```

Enable CronTab job to renew our certificates automatically
```bash
sudo crontab -e
```
```
0 0,12 * * * python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew 
```

# Allow Nginx to read user content

```bash
setsebool -P httpd_read_user_content 1
```
 