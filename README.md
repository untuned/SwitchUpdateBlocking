# Switch Update Blocking

This is a simple guide to setting up a DNS Server to block all communications with Nintendo.

## Setting up Ubuntu Server

For my server I will be installing Ubuntu Server LTS 18.04. Once you have it installed lets make sure that everything is up to date.

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt autoremove
sudo reboot
```

While we are at it lets install all the packages we are going to need.

```bash
sudo apt-get install bind9 bind9utils bind9-doc nginx
```

Next lets setup our firewall. We are going to want to allow access to SSH, Bind9 (DNS Server), and Nginx (HTTP Server).

```bash
sudo ufw allow OpenSSH
sudo ufw allow Bind9
sudo ufw allow "Nginx HTTP"
```

Afterwards we need to enable our firewall.

```bash
sudo ufw enable
```

## Setting up Bind9 (DNS Server)

Lets start off by configuring the options file. Using your favorite text editor open up `/etc/bind/named.conf.options`.

```
options {
        directory "/var/cache/bind";

        allow-transfer { none; };
        allow-update { none; };
        llow-recursion { none; };

        version "none";
        recursion no;

        auth-nxdomain no;
        listen-on-v6 { none; };

        forwarders {
                1.1.1.1;
                1.0.0.1;
        };
};
```

For the forwarders you can use any DNS server you would like, however the ones in the example are Cloudflare's privacy focused DNS Servers. Next lets create our zones. I've gone overboard on the zones to make sure that my Switch does not communicate with Nintendo what-so-ever. Once again using your favorite text editor open up `/etc/bind/named.conf.local`.

```
zone "nintendo.com" {
        type master;
        file "/etc/bind/zones/db.nintendo.com";
};

zone "nintendo.net" {
        type master;
        file "/etc/bind/zones/db.nintendo.net";
};

zone "nintendowifi.net" {
        type master;
        file "/etc/bind/zones/db.nintendowifi.net";
};

zone "wii.com" {
        type master;
        file "/etc/bind/zones/db.wii.com";
};
```

Each one of these are domains we are going to spoof to prevent our Switch from calling home. For `nintendo.com`, `nintendowifi.net`, and `wii.com` they are all similar with their respective domain names changed out so go ahead and start creating the individual zone files for each of them base on the one below:

```
$TTL    604800
@       IN      SOA     nintendo.com. admin.nintendo.com. (
                     2018052901         ; Serial
                           7200         ; Refresh
                            120         ; Retry
                        2419200         ; Expire
                           3600 )       ; Negative Cache TTL
;
        IN      NS      10.0.0.8
                A       127.0.0.1
@       IN      A       127.0.0.1
*       IN      A       127.0.0.1
                AAAA    ::1
@       IN      AAAA    ::1
*       IN      AAAA    ::1
```

For the NS record you are going to want to change that to be the ip address or domain name of your DNS server. Now for `nintendo.net` we have a special record to point `ctest.cdn.nintendo.net` to our web server, which we will setup in the next section.

```
$TTL    604800
@       IN      SOA     nintendo.net. admin.nintendo.net. (
                     2018052901         ; Serial
                           7200         ; Refresh
                            120         ; Retry
                        2419200         ; Expire
                           3600 )       ; Negative Cache TTL
;
        IN      NS      10.0.0.8
                A       127.0.0.1
@       IN      A       127.0.0.1
ctest.cdn.nintendo.net. IN      A       10.0.0.8
*       IN      A       127.0.0.1
                AAAA    ::1
@       IN      AAAA    ::1
*       IN      AAAA    ::1
```

Once again you are going to want to change the `ctest.cdn.nintendo.net.` record to match your server's ip address. That will be our DNS server all setup, you will want to restart and enable Bind9 to auto start if the server ever reboots.

```bash
sudo systemctl restart bind9
sudo systemctl enable bind9
```

## Setting up Nginx (Web Server)

First let create the directory for our web page.

```bash
sudo mkdir -p /var/www/ctest.cdn.nintendo.net/html/
```

Then using your favorite text editor create an `index.txt` file in the folder we just created with the word `ok` in it. Next we want to make sure all of files and folders have the correct permissions.

```bash
sudo chown -R www-data:www-data /var/www/*
sudo chmod -R 755 /var/www/*
```

Lets configure Nginx to host our file now. Using your favorite text editor lets create the file `/etc/nginx/sites-available/ctest.cdn.nintendo.net`.

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/ctest.cdn.nintendo.net/html;
        index index.txt;

        add_header X-Organization Nintendo;

        server_name ctest.cdn.nintendo.net;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

Now that we have our site configured we need to enable it and remove the default site.

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/ctest.cdn.nintendo.net /etc/nginx/sites-enabled/ctest.cdn.nintendo.net
```

With that web server all setup, you will want to restart and enable Nginx to auto start if the server ever reboots.

```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```