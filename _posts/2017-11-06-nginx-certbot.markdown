---
layout: post
title:  NGINX and Certbot
date:   2017-11-06
categories: tutorials
---
### What is NGINX?

[NGINX](https://nginx.com) is a Linux web load balancer that is used to serve files for websites, such as this blog and my [website](https://sethgower.com). You can read more about Load Balancing [here](https://en.wikipedia.org/wiki/Load_balancing_(computing)).

### What is Certbot?

[Certbot](https://certbot.eff.org/) is a piece of software that allows you to easily install SSL certificates via [Let's Encrypt](https://letsencrypt.org/) and the [Electronic Frontier Foundation](https://www.eff.org/). SSL is a very important part of making a website because it ensures that any traffic sent between your website (possibly passwords, credit cards, etc) are protected and encrypted through HTTPS.

## NGINX

##### Installing NGINX
These instructions are for Linux based systems, since Windows doesn't support some of these features.

First, you must install NGINX, this can be done through the following terminal commands, depending on the distro you are running.

- For Debian and Ubuntu based systems, use <br>
    `sudo apt-get update` <br>
    `sudo apt-get install nginx`
- For Fedora and Red Hat based systems, you must install the EPEL repository. <br>
    `sudo yum install epel-release`<br>
    Now Install the packages <br>
    `sudo yum update`

From here on, I will be discussing installation in terms of a Debian system, but it should be fairly universal.

##### Configuring NGINX

This section assumes that you have already configured your DNS provider to point your domain to your server. This can be done by creating an `A` record for `@` to point to `<public ip of your server>`.

Once you have NGINX installed, navigate to `/etc/nginx/sites-available/` and create a file (I would suggest naming it the fully qualified domain name, ie `blog.sethgower.com` for this website).

Inside of that file (in order to edit the file, you will need `sudo` or root access) put the following code.
```nginx
server {
    listen 80;

    server_name yourdomain.com www.yourdomain.com;
    root /path/to/your/html;
    index index.html index.htm;
    location / {
        try_files $uri $uri/ =404
    }
}
```
Now let's walk through that line by line (starting side of the `server` tag)

1. On Line 2, `listen 80;` tells NGINX to listen on port 80, which is the default TCP port for HTTP traffic.
2. `server_name` tells NGINX to look for traffic looking for `yourdomain.com` and `www.yourdomain.com` and tells it to follow the instructions in this file for traffic matching that.
3. `root` is the location where your website is stored, ie where your `index.html`, `main.css`, etc are located, the convention in Debian systems is `/var/www/folder-for-website`.
4. `index` just tells NGINX what the main page is called in that root folder
5. `location /` tells NGINX where to send any traffic looking for `(www.)yourdomain.com/`, you can use this functionality to do things like `/blog` and it will direct those requests as specified in the subsequent block surrounded by `{}`.
6. `try_files $uri $uri/ =404` says "Send all traffic looking for `/` to the files specified in `root`, and if what they are looking for can't be found, send them to the `404` page".

You can Do other things in the `location /` block, such as, as I mention in my tutorial on [GitHub webhooks]({{ site.baseurl }}{% post_url 2017-11-05-webhook-puller %}), you can send traffic to another IP or port on your local host, if you have an app running by using the `proxy_pass` flag followed by the IP along with the protocol, for example `http://localhost:12345`.

The final step before your server will work is to make a symlink between this file and itself in `/etc/nginx/sites-enabled/`. You can do this by running
```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/yourdomain.com
```

Now, after you populate your `/var/www/yourdomain.com/` with HTML, then you should have a working website. You can use this sample HTML for now
```html
<!DOCTYPE html>
<html>
    <body>
        <h1>Hello World</h1>
    </body>
<html>
```
And then reload NGINX by executing
```bash
sudo systemctl reload nginx.service
```

## Certbot

Now that you can reach your website, from the internet, it is time to secure your website with HTTPS. We do this by using Certbot. Certbot will do the majority of the work for you, so all you have to do is run the following commands.

```bash
sudo certbot --nginx -d yourdomain.com
```

After you run this, you will get this output, if your DNS record is properly setup and your NGINX is properly setup.

```bash
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Obtaining a new certificate
Performing the following challenges:
tls-sni-01 challenge for yourdomain.com
Generating key (1024 bits): /var/lib/letsencrypt/snakeoil/0015_key.pem
Waiting for verification...
Cleaning up challenges
Generating key (2048 bits): /etc/letsencrypt/keys/0016_key-certbot.pem
Creating CSR: /etc/letsencrypt/csr/0016_csr-certbot.pem
Generating key (1024 bits): /var/lib/letsencrypt/snakeoil/0016_key.pem
Deployed Certificate to VirtualHost /etc/nginx/sites-enabled/test for set(['yourdomain.com'])

Please choose whether HTTPS access is required or optional.
-------------------------------------------------------------------------------
1: Easy - Allow both HTTP and HTTPS access to these sites
2: Secure - Make all requests redirect to secure HTTPS access
-------------------------------------------------------------------------------
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

If you select `2`, then ***all*** traffic coming to your server will be encrypted, which is ***highly*** suggested.

After you finish the previous step, you should see this

```bash
Redirecting all traffic on port 80 to ssl in /etc/nginx/sites-enabled/yourdomain.com

-------------------------------------------------------------------------------
Congratulations! You have successfully enabled https://yourdomain.com

You should test your configuration at:
https://www.ssllabs.com/ssltest/analyze.html?d=yourdomain.com
-------------------------------------------------------------------------------
```

Now if you look into your `/etc/nginx/sites-available/yourdomain.com` you should now see this.


```nginx
server {
        listen 80;

        server_name yourdomain.com;
        root /var/www/yourdomain.com;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }

        listen 443 ssl; # managed by Certbot

ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem; # managed by Certbot
ssl_session_cache shared:le_nginx_SSL:1m; # managed by Certbot
ssl_session_timeout 1440m; # managed by Certbot

ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # managed by Certbot
ssl_prefer_server_ciphers on; # managed by Certbot

#Don't worry, I am not pulling an adobe, this is inactive
ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES128-SHA ECDHE-ECDSA-AES256-SHA ECDHE-ECDSA-AES128-SHA256 ECDHE-ECDSA-AES256-SHA384 ECDHE-RSA-AES128-GCM-SHA256 ECDHE-RSA-AES256-GCM-SHA384 ECDHE-RSA-AES128-SHA ECDHE-RSA-AES128-SHA256 ECDHE-RSA-AES256-SHA384 DHE-RSA-AES128-GCM-SHA256 DHE-RSA-AES256-GCM-SHA384 DHE-RSA-AES128-SHA DHE-RSA-AES256-SHA DHE-RSA-AES128-SHA256 DHE-RSA-AES256-SHA256 EDH-RSA-DES-CBC3-SHA"; # managed by Certbot

    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    } # managed by Certbot

}
```

Now, if you were to test your website [here](https://www.ssllabs.com/ssltest/) you won't get a full A+. There are still some tweaks you need to do. First, you need to run

```bash
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```
and then add the following code to your `/etc/nginx/sites-available/yourdomain.com`


```nginx
ssl_dhparam /etc/ssl/certs/dhparam.pem;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

This adds a [DH Parameter](https://security.stackexchange.com/questions/94390/whats-the-purpose-of-dh-parameters) to you SSL and enables HSTS (HTTPS Strict Transport Security). Which guarantees that the client will connect via HTTPS.
