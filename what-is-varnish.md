# What is Varnish Cache?

```
Varnish Cache is a specialized HTTP caching server software that works as a caching layer located in front of web servers. 
This software handles incoming HTTP requests and stores them in the cache to provide fast responses to these requests. 
As a result, it enables websites to respond faster and reduces server load, HTTP reverse proxy.
```


## How Does Varnish Work?

```
Varnish analyzes incoming requests and if it has already answered similar requests, it retrieves the response from the cache without making a new request to the server. 
This allows web pages to load faster and the server to use its resources more efficiently. Varnish automatically purges or updates cached content after a period of time, so that it continues to serve the most up-to-date data.
In this way, situations such as database operations, backend requests, 3rd party requests that tire the server or take time are reduced.
It directs visitors to cached static pages instead of dynamic pages that are recreated with each request and ensures that the site arrives quickly.
```

```
Suppose we have a page that receives heavy traffic and pulls the data it contains from a database. Each database process runs a process on its own. After a while, this will increase the bandwidth usage on the server side. In parallel with this, RAM and CPU ratios will increase, the response time of the web server will also increase and as a result, it will become unresponsive. Varnish helps us at this point. How? Let our traffic and transaction load continue at the aforementioned intensity. When this page is requested to be displayed; Varnish forwards the request to the server because it is not in memory. The server returns a response. This response is kept in the cache for the specified time. The same request will continue to respond through the cached varnish layer, not through the database, regardless of which user it is for the predetermined period of time. When the time expires, that is, if there is no data in the varnish cache (invalid), it requests the server again.
If the cache time is set to 1 minute on a page that receives 1,000 requests per minute, the server will only deal with 1 request instead of 1,000.
```


## VCL (Varnish Configuration Language)


1. HTTP Requests and Responses Handling: VCL is used to handle incoming HTTP requests and server responses and to specify custom rules and operations for them. This is important for delivering fast and customized responses to clients.

2. Caching Rules: VCL is used to determine which content is cached and for how long, which content is not cached, and which content is automatically purged. This ensures that the cached content is up-to-date and efficient.

3. Routing and Load Balancing: VCL can be used to redirect incoming requests to different servers or perform load balancing. This is important to improve the performance of high-traffic websites.

4. Special Operations: VCL can be used to implement customized functionality such as custom operations, filters or security measures. This is used to meet the special requirements of web applications.

```
VCL makes Varnish Cache a versatile caching tool and is used to improve the performance and security of web servers. VCL can be customized to specific use cases and requirements, so different VCL configurations can be created for different websites.
```

## Varnish Installation

### Register the package repository

```
sudo apt-get update
```

```
sudo apt-get install debian-archive-keyring curl gnupg apt-transport-https
```

```
curl -s -L https://packagecloud.io/varnishcache/varnish60lts/gpgkey | sudo apt-key add -
```

```
. /etc/os-release
sudo tee /etc/apt/sources.list.d/varnishcache_varnish60lts.list > /dev/null <<-EOF
deb https://packagecloud.io/varnishcache/varnish60lts/$ID/ $VERSION_CODENAME main
EOF
sudo tee /etc/apt/preferences.d/varnishcache > /dev/null <<-EOF
Package: varnish varnish-*
Pin: release o=packagecloud.io/varnishcache/*
Pin-Priority: 1000
EOF
```

```
sudo apt-get update
```

#### Installation

```
sudo apt-get install varnish
```

### Config Varnish

#### Systemd Configuration

The Vernishd process is managed by ```systemd`` and has a unit file in ``/lib/systemd/system/varnish.service`` as shown below:

```
[Unit]
Description=Varnish Cache, a high-performance HTTP accelerator
After=network-online.target nss-lookup.target

[Service]
Type=forking
KillMode=process

# Maximum number of open files (for ulimit -n)
LimitNOFILE=131072

# Locked shared memory - should suffice to lock the shared memory log
# (varnishd -l argument)
# Default log size is 80MB vsl + 1M vsm + header -> 82MB
# unit is bytes
LimitMEMLOCK=85983232

# Enable this to avoid "fork failed" on reload.
TasksMax=infinity

# Maximum size of the corefile.
LimitCORE=infinity

ExecStart=/usr/sbin/varnishd \
	  -a :80 \
	  -a localhost:8443,PROXY \
	  -p feature=+http2 \
	  -f /etc/varnish/default.vcl \
	  -s malloc,256m
ExecReload=/usr/sbin/varnishreload

[Install]
WantedBy=multi-user.target
```

If you want to override some runtime parameters in the Varnish.service file, you can run the following command:

```
sudo systemctl edit --full varnish
```

An editor will open where you can edit the file. The content of the file comes from ```/lib/systemd/system/varnish.service``.

After making changes, make sure to save the file and exit the editor. As a result, the file ```/etc/systemd/system/system/varnish.service`` will be created containing the modified service file.

NOTE: It is also possible to write the changes directly to ```/etc/systemd/system/system/varnish.service``.

```
systemctl start varnish
systemctl enable varnish
```

### Setting the listened port and cache size

From the *.service file we can specify the port change or the amount of cache size.

```
ExecStart=/usr/sbin/varnishd \
	  -a :6081 \ # ----> "Default is 6081, port change is possible directly from the beginning"
	  -a localhost:8443,PROXY \
	  -p feature=+http2 \
	  -f /etc/varnish/default.vcl \
	  -s malloc,256m # ----> "-s malloc,2g" instead of "-s malloc,256m" increases the cache size to 2 gigabytes.
```

For example;

```
ExecStart=/usr/sbin/varnishd \
	  -a :80 \
	  -a localhost:8443,PROXY \
	  -p feature=+http2 \
	  -f /etc/varnish/default.vcl \
	  -s malloc,2g
```

------

As an additional example:

```/etc/varnish/default.vcl``

```
vcl 4.0;

backend default {
    .host = "127.0.0.0.1";
    .port = "80";
}

sub vcl_recv {
    # You can process incoming requests here
    # For example, you can add some custom cache rules
}

sub vcl_backend_response {
    # You can process server responses here
    # For example, you can set the cache duration
}
```

#### Configure the web server to work with Varnish

Now that Varnish is configured to listen on port 80, your web server will need to be reconfigured and listen on an alternate port. The most common alternate port for HTTP is port 8080.

```
Here are instructions on how to do this for some common web servers such as Apache and Nginx. Note that the instructions here assume that web servers are running with a default installation. If your current web server setup has a custom configuration, the steps outlined may need to be modified.
```

### Apache

If you are using Apache, you change the listening port value in ```/etc/apache2/ports.conf`` from Listen 80 to Listen 8080. You also need to replace ```<VirtualHost *:80> with <VirtualHost *:8080>```. virtual host files.

```
sudo find /etc/apache2 -name '*.conf' -exec sed -r -i 's/\bListen 80\b/Listen 8080/g; s/<VirtualHost ([^:]+):80>/<VirtualHost \1:8080>/g' {} ';'
```

### Nginx

If you are using Nginx, this is just a matter of changing the listening port in the various virtual host configurations.

The following command will replace ```listen 80`` with ``listen 8080`` on all virtual host files:

```
sudo find /etc/nginx -name '*.conf' -exec sed -r -i 's/\blisten ([^:]+:)?80\b([^;]*);/listen \18080\2;/g' {} ';'
```

### VCL backend configuration

Now that we have changed the listening port of the original web server to ```8080``, this change needs to be reflected in the Backend definition of your VCL file.

The default VCL file is located in ```/etc/varnish/default.vcl`` on your system and contains the following Backend definition:

```
vcl 4.1;

backend default {
    .host = "127.0.0.0.1";
    .port = "8080";
}
```

NOTE: DNS resolution of a hostname in a VCL file occurs only once when the VCL file is compiled. Any DNS changes to this hostname will not be noticed by Varnish until the VCL is reloaded.

### Restart services

First of all, host restart is recommended, but it will be enough to throw it to the webcasting tool used.

```
Apache: sudo systemctl restart apache2 varnish

Nginx: sudo systemctl restart nginx varnish
```


https://github.com/ugurbzkrt