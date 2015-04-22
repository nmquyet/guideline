## vsftpd

```
sudo yum install vsftpd
sudo yum install ftp
```

```
sudo vi /etc/vsftpd/vsftpd.conf
```

One primary change you need to make is to change the Anonymous_enable to No:

```
anonymous_enable=NO
```

Uncomment the local_enable option, changing it to yes.

```
local_enable=YES
```

Uncomment command to chroot_local_user. When this line is set to Yes, all the local users will be jailed within their chroot and will be denied access to any other part of the server.

```
chroot_local_user=YES
```


Finish up by restarting vsftpd:

```
sudo service vsftpd restart
chkconfig vsftpd on
```

### Virtual User Configurations 

http://www.unixmen.com/install-vsftp-with-virtual-users-on-centos-rhel-scientific-linux-6-4/

## IPTABLES

#### Default rule

```
*filter

#  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT -d 127.0.0.0/8 -j REJECT

#  Accept all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

#  Allow all outbound traffic - you can modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

#  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

#  Allow SSH connections
#
#  The -dport number should be the same port number you set in sshd_config
#
-A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT

#  Allow ping
-A INPUT -p icmp -j ACCEPT

#  Log iptables denied calls
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

#  Drop all other inbound - default deny unless explicitly allowed policy
-A INPUT -j DROP
-A FORWARD -j DROP

COMMIT
```

#### Save rules on start up

##### Ubuntu/Debian 

Create file `/etc/network/if-pre-up.d/firewall` with following content

```
#!/bin/sh 
/sbin/iptables-restore < /etc/iptables.firewall.rules
```

Change script permission

```
sudo chmod +x /etc/network/if-pre-up.d/firewall
```


##### CENTOS/Fedora

If you are using CentOS 6.2 or 6.5, save your current iptables rules with the following command:

```
/sbin/service iptables save 
```

If you are using CentOS 7 or Fedora 20, the base image does not include iptables-services. You will need to install it before your firewall is persistent through boots:

```
yum install -y iptables-services
systemctl enable iptables
systemctl start iptables
/usr/libexec/iptables/iptables.init save
```

## SSH

##### Login Public Key

To allow *public key* login, add your public key to `$USER_HOME/.ssh/authorized_keys`

##### Secure SSH server

Fine tune your `/etc/ssh/sshd_config` to protect your server


Disable password login

```
PasswordAuthentication no
```

Edit `/etc/ssh/sshd_config` to disable root login

```
PermitRootLogin no
```

## NGINX 

#### Config PHP-FPM 

Socket configuration for every virtual host in `/etc/php-fpm.d/`

```
[your-site-name]

;; Socket directory
listen = /var/run/php-fpm/your-site-socket.socket
listen.backlog = -1
listen.owner = nginx
listen.group = www-data
listen.mode=0660

; Unix user/group of processes
user = nginx
group = www-data

; Choose how the process manager will control the number of child processes.
pm = dynamic
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 2
pm.max_spare_servers = 5
pm.max_requests = 10

; Pass environment variables
env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp

; host-specific php ini settings here
; php_admin_value[open_basedir] = /var/www/DOMAINNAME/htdocs:/tmp

; enable display of errors
php_flag[display_errors] = on
php_flag[display_startup_errors] = on
```

#### Foward php script to PHP-FPM

Add following lines inside `server` directive block

```
    location ~ \.php$ {
        root           /your-web-root;

        # Basic authentication
        # auth_basic     "Basic Auth";
        # auth_basic_user_file /etc/nginx/.htpasswd;

        # fastcgi_pass   127.0.0.1:9000;
        # Connect to php-fpm through unix socket
        fastcgi_pass   unix:/var/run/php-fpm/your-php-fpm-socket.socket;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
```


##### CENTOS

Create nginx repo in '/etc/yum.repo.d/nginx.repo' directory with following content

```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$version/$basearch/
gpgcheck=0
enabled=1
```

*Note: Replace $version with appropriate version (5,6,7...)*


## REDIS 

##### CENTOS 7

```
wget -r --no-parent -A 'epel-release-*.rpm' http://dl.fedoraproject.org/pub/epel/7/x86_64/e/
rpm -Uvh dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-*.rpm
yum install redis
systemctl start redis.service
redis-cli ping
```


Configs:

```
/etc/redis.conf
/etc/redis-sentinel.conf
```

## Memcached

Start up command

```
memcached -d -u nobody -m 128 -p 11211 127.0.0.1
```

## PostgreSQL

##### Init db

```
postgresql-setup initdb
su - postgres
psql
```

##### Allow remote connection

Find `pg_hba.conf` location following PSQL statement

```
show hba_file;
```

Add to `pg_hba.conf`

```
host    all             all             0.0.0.0/0               md5
```

Edit `postgresql.conf` to listen to all ip address

```
listen_addresses = '*'
```
