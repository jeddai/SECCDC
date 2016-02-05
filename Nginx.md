### Links
- [Nginx server](#nginx)
    - [Server block](#server_block)
        - [Location block](#location_block)
        - [New block](#new_block)
    - [Hosts](#hosts)
    - [PHP](#php5-fpm)
    - [Wrap up](#wrapup)

<a name="nginx"></a>
## Nginx Server (172.31.251.72)
I have configured an Nginx server instead of apache on the DNS server to act as a web hub for the CCT sites. Nginx has the capability to host multiple web sites, with multiple different domain names, all on one server. Its secret is in its sites-available folder. In this folder, when you first install Nginx, is nothing more than a `default` file. The key is that you can make more files which allow you to have the Nginx server show the user a different folder based on which url they are accessing the server from. Here is an example of the `default` file.
```
server {
	listen 80;
	root /var/www/html;
	index index.php index.html index.htm;
	server_name cct.lipscomb.edu;

	location / {
		root /var/www/html;
		try_files $uri $uri/ =404;
	}

	location ~ \.php$ {
		fastcgi_split_path_info ^(.+\.php)(/.+)$;
		fastcgi_pass 127.0.0.1:9000;
		fastcgi_index index.php;
		fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
		include fastcgi_params;
	}
}
```
---
<a name="server_block"></a>
### Server block
Let's break it down, shall we? First we have the declaration of the server block, and the listen, root, index, and server_name lines.
```
server {
	listen 80;
	root /var/www/html;
	index index.php index.html index.htm;
	server_name cct.lipscomb.edu;
```
The `listen` line tells the Nginx server to listen on port 80. When we acquire an SSL certificate then we will have it redirect port 80 to port 443, but that is irrelevant at the moment. The `root` line tells the server where exactly the root directory for this server is, i.e: where the index file is. The `index` line is telling the server what the index file should be named. The `server_name` line is the most important line here. It tells the server what the domain we are looking for is. If it reads that the url is cct.lipscomb.edu, then it will show the user this site.

---
<a name="location_block"></a>
#### Location block
Next is the location block. These two blocks tell the server what to do exactly. First is `location /`.
```
location / {
  root /var/www/html;
  try_files $uri $uri/ =404;
}
```
This block is the default block. It declares a new `root` and it tries for files at the url, and if there are no files then it checks the other location blocks, and then returns a 404 page if nothing is returned.

Next is the `location ~ \.php$` block. This block checks for a php index file and allows the nginx server to work with php5-fpm.
```
location ~ \.php$ {
  fastcgi_split_path_info ^(.+\.php)(/.+)$;
  fastcgi_pass 127.0.0.1:9000;
  fastcgi_index index.php;
  fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
  include fastcgi_params;
}
```
The `fastcgi_split_path_info` line is just there. Not sure why. The `fastcgi_pass` line tells the server that this is the PHP port we will be using for this server block. The `fastcgi_index` line tells the server what the php index file should be called. the `fastcgi_param` line tells the server where the index file should be located. Then the `include` line actually tells the server to include all that info you just put above the `include` line.

---
<a name="new_block"></a>
#### New block
Next is an example of another file. This file is just like the one above, but with a different `server_name`.
```
server {
    listen 80;

    root /var/www/webdev.cct.lipscomb.edu;

    server_name webdev.cct.lipscomb.edu *.webdev.cct.lipscomb.edu;

    location /static {
	    index index.html;
    }
    location / {
            root /var/www/webdev.cct.lipscomb.edu;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass unix:/var/run/php5-fpm-webdev-cct.sock;
            fastcgi_index universihttp.php;
            fastcgi_param SCRIPT_FILENAME $document_root/script/universihttp.php;
            include fastcgi_params;
    }

    #server_name webdev.cct.lipscomb.edu *.webdev.cct.lipscomb.edu;
    #return 301 https://$server_name$request_uri;
    #^^ These two lines would normally redirect all port 80 requests to port 443, aka https. We don't have an SSL certificate yet so we can't quite do that just yet.
}
```
You will notice that the `server_name` line has an entry for webdev.cct.lipscomb.edu. It makes it so if that URL is the one being accessed, it then uses this server as the one the user sees. When you make a new server block file, the last thing you need to do is symbolically link the file in the `sites-available` folder with another file in the `sites-enabled` folder by running the command `sudo ln -s /etc/nginx/sites-available/SITENAME /etc/nginx/sites-enabled/SITENAME`.

---
<a name="hosts"></a>
### Hosts file
After making the server block files there are a few more things you need to do. First, add the URL into the `/etc/hosts` file. You do this by running `sudo nano /etc/hosts`. This is what the /etc/hosts file should look like once you have added URLs.
```
127.0.0.1       localhost
127.0.1.1       cctadmin-dns
127.0.0.1       help.cct.lipscomb.edu
127.0.0.1       webdev.cct.lipscomb.edu
127.0.0.1       cacti.cct.lipscomb.edu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

---
<a name="php5-fpm"></a>
### php5-fpm
The last thing we have to configure is php5-fpm. We only need to configure this if the server block we created needs php to run. First we create a new group and a new user.
```
sudo groupadd site1
sudo useradd -g site1 site1
```
This makes a group and user for our site. Next we add a `.conf` file for it in the `pool.d` php directory. The following file would be called/located in `/etc/php5/fpm/pool.d/site1.conf`.
```
[site1]
user = site1
group = site1
listen = /var/run/php5-fpm-site1.sock
listen.owner = www-data
listen.group = www-data
php_admin_value[disable_functions] = exec,passthru,shell_exec,system
php_admin_flag[allow_url_fopen] = off
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
chdir = /
```
The `listen` line there is what goes in the `fastcgi_pass unix:/var/run/php5-fpm-site1.sock;` line in the `~ \.php$` location block in the server block file we created earlier.

---
<a name="wrapup"></a>
### Wrap up
The last thing you need to do is run `sudo service php5-fpm restart && sudo service nginx restart` and you are good to go!
