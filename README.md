<img src="https://1.tilyanpristka.id/images/tP-logo-rounded.png" height="111" align="right">

# <img src="https://www.haproxy.org/img/HAProxyCommunityEdition_60px.png" height="40" align="left"> Ubuntu 22.04 - HAProxy - HTTP3 - QUIC


```bash
ufw allow 80/tcp
ufw allow 234/tcp #for haproxy stats
ufw allow 443/tcp
ufw allow 443/udp
ufw allow 8080/tcp
ufw enable
ufw reload
```

```bash
apt-get install build-essential
```

```bash
cd /usr/local/src/
git clone https://github.com/quictls/openssl
cd openssl
mkdir -p /opt/quictls/ssl
./Configure --libdir=lib --prefix=/opt/quictls
make -j $(nproc)
make install
```

```bash
apt install lua5.4 libpcre3-dev libreadline-dev libpcre3-dev libssl-dev zlib1g-dev libsystemd-dev -y
```

```bash
cd /usr/local/src/
curl -R -O http://www.lua.org/ftp/lua-5.4.6.tar.gz
tar xvzf lua-5.4.6.tar.gz
cd lua-5.4.6/
make linux test
sudo make install
```


> open `https://www.haproxy.org/` choose which version you want.
```bash
cd /usr/local/src/
wget http://www.haproxy.org/download/2.6/src/haproxy-2.6.14.tar.gz
tar xvzf haproxy-2.6.14.tar.gz
cd haproxy-2.6.14/
make -j $(nproc) TARGET=linux-glibc USE_LUA=1 USE_PCRE=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_GZIP=1 USE_PROMEX=1 USE_QUIC=1 USE_OPENSSL=1 SSL_INC=/opt/quictls/include SSL_LIB=/opt/quictls/lib LDFLAGS="-Wl,-rpath,/opt/quictls/lib"
make install-bin
```

```
groupadd -g 188 haproxy
useradd -g 188 -u 188 -d /var/lib/haproxy -s /sbin/nologin -c haproxy haproxy

mkdir /run/haproxy/
mkdir -p /etc/haproxy
touch /etc/haproxy/haproxy.cfg
chown -R haproxy:haproxy /etc/haproxy/
mkdir -p /var/lib/haproxy
touch /var/lib/haproxy/stats
chown -R haproxy:haproxy /var/lib/haproxy/
ln -s /usr/local/sbin/haproxy /usr/sbin/haproxy
```

```
nano /etc/systemd/system/haproxy.service
----------------------------------------------------------------------------------------------------------
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
Documentation=file:/usr/share/doc/haproxy/configuration.txt.gz
After=network-online.target rsyslog.service
Wants=network-online.target

[Service]
EnvironmentFile=-/etc/default/haproxy
EnvironmentFile=-/etc/sysconfig/haproxy
BindReadOnlyPaths=/dev/log:/var/lib/haproxy/dev/log
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "PIDFILE=/run/haproxy.pid" "EXTRAOPTS=-S /run/haproxy-master.sock"
ExecStart=/usr/sbin/haproxy -Ws -f $CONFIG -p $PIDFILE $EXTRAOPTS
ExecReload=/usr/sbin/haproxy -Ws -f $CONFIG -c -q $EXTRAOPTS
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed
Restart=always
SuccessExitStatus=143
Type=notify

[Install]
WantedBy=multi-user.target
----------------------------------------------------------------------------------------------------------
```

```
nano /etc/haproxy/haproxy.cfg
----------------------------------------------------------------------------------------------------------
global
 daemon
 log 127.0.0.1 local0
 log 127.0.0.1 local1 notice

 maxconn 4096

 tune.ssl.default-dh-param 2048
 tune.bufsize 32678
 tune.maxrewrite 4096
 tune.http.maxhdr 202

 stats socket /run/admin.sock mode 660 level admin
 stats timeout 120s
 user haproxy
 group haproxy

# ----------------------------------------------------------------
defaults
 log global
 mode http
 maxconn 4096

 option redispatch
 option dontlognull
 option dontlog-normal
 option http-server-close
 option log-health-checks

 retries 3
 timeout connect 600s
 timeout client 600s
 timeout server 600s
 timeout queue 600s
 timeout check 600s
 timeout client-fin 60s
 timeout server-fin 60s
 timeout http-request 120s
 timeout http-keep-alive 1s
 timeout tunnel 3h

 errorfile 400 /etc/haproxy/errors/400.http
 errorfile 403 /etc/haproxy/errors/403.http
 errorfile 408 /etc/haproxy/errors/408.http
 errorfile 500 /etc/haproxy/errors/500.http
 errorfile 502 /etc/haproxy/errors/502.http
 errorfile 503 /etc/haproxy/errors/503.http

 compression algo gzip
 compression type text/html text/plain text/xml text/css text/javascript "text/html; charset=utf-8" text/html;charset=utf-8 application/x-javascript application/javascript application/ecmascript application/rss+xml application/atomsvc+xml application/atom+xml application/wasm application/atom+xml;type=entry application/atom+xml;type=feed application/cmisquery+xml application/cmisallowableactions+xml application/cmisatom+xml application/cmistree+xml application/cmisacl+xml application/msword application/vnd.ms-excel application/vnd.ms-powerpoint image/svg+xml font/ttf font/otf font/opentype application/xhtml+xml image/x-icon application/x-font-ttf application/x-font-truetype application/x-font-otf application/x-font-opentype application/vnd.ms-fontobject

# ----------------------------------------------------------------
listen stats
 bind *:234
 stats enable
 stats hide-version
 stats realm Haproxy\ Statistics
 stats uri /stats
 stats auth admin:password
 stats refresh 30s
 stats show-modules

# ----------------------------------------------------------------
frontend https_in
 mode http
 bind *:80
 bind *:443 ssl crt-list /etc/haproxy/ssl/crt-list.txt alpn h2,http/1.1
 bind quic4@:443 ssl crt-list /etc/haproxy/ssl/crt-list.txt alpn h3

 http-request redirect scheme https unless { ssl_fc }
 http-response set-header alt-svc "h3=\":443\";ma=86400,h3-29=\":443\";ma=86400,h3-Q050=\":443\";ma=86400,h3-Q046=\":443\";ma=86400,h3-Q043=\":443\";ma=86400,quic=\":443\";ma=86400;v=\"43,46\""

 http-response del-header Server
 http-response add-header Server "TILYANPRISTKA"

 http-response del-header X-Firefox-Spdy
 http-response del-header X-Frame-Options
 http-response add-header X-Frame-Options SAMEORIGIN
 http-response add-header X-LiteSpeed-Cache hit

# ACL ------------------------------------------------------------
 acl dns_www_tpid hdr_dom(host) -i tilyanpristka.id

# BACKEND --------------------------------------------------------
 use_backend www_prod if dns_www_tpid

# DEFAULT --------------------------------------------------------
 default_backend www_prod

# BACKEND --------------------------------------------------------
backend www_prod
 #balance roundrobin
 #cookie SRVR insert indirect nocache

 server srvr1 10.11.10.15:80 maxconn 2048 weight 1 check
 #server srvr1 10.11.10.15:80 maxconn 1024 weight 1 cookie srvr1 check
 #server srvr2 10.11.10.16:80 maxconn 1024 weight 1 cookie srvr2 check
----------------------------------------------------------------------------------------------------------
```

```
systemctl restart haproxy
```
