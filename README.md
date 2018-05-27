# KVM_webvirtmgr
```
sudo yum -y install http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
sudo yum -y install git python-pip libvirt-python libxml2-python
                    python-websockify nginx novnc
wget -O python-supervisor-3.0-3.noarch.rpm
                    https://packagecloud.io/haf/oss/packages/el/6/python-supervisor-3.0-3.noarch.rpm/download
yum localinstall -y python-supervisor-3.0-3.noarch.rpm
```

mkdir -p /var/www/
cd /var/www/
git clone git://github.com/retspen/webvirtmgr.git
cd webvirtmgr
pip install -r requirements.txt
<br>这里会提示让创建一个登录用户，按照提示创建就可以了，
以后可以执行`./manage.py createsuperuser`再创建其它登录用户<br>
./manage.py syncdb
./manage.py collectstatic


#将该目录的拥有者修改为nginx用户<br>
chown -R nginx:nginx /var/www/webvirtmgr

#增加webvirtmgr的nginx配置<br>
/etc/nginx/conf.d/webvirtmgr.conf

```
server {
    listen 80 default_server;

    server_name $hostname;
    #access_log /var/log/nginx/webvirtmgr_access_log;

    location /static/ {
        root /var/www/webvirtmgr/webvirtmgr; # or /srv instead of /var
        expires max;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-for $proxy_add_x_forwarded_for;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 600;
        proxy_read_timeout 600;
        proxy_send_timeout 600;
        client_max_body_size 1024M; # Set higher depending on your needs
    }
}
```
#注释掉nginx原有的默认主机配置<br>
sed -i -e "s/^/#/g" /etc/nginx/conf.d/default.conf

#增加webvirtmgr的supervisor配置<br>
/etc/supervisor/conf.d/webvirtmgr.conf

```
[program:webvirtmgr]
command=/usr/bin/python /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.conf.py
directory=/var/www/webvirtmgr
autostart=true
autorestart=true
logfile=/var/log/supervisor/webvirtmgr.log
log_stderr=true
user=nginx

[program:webvirtmgr-console]
command=/usr/bin/python /var/www/webvirtmgr/console/webvirtmgr-console
directory=/var/www/webvirtmgr
autostart=true
autorestart=true
stdout_logfile=/var/log/supervisor/webvirtmgr-console.log
redirect_stderr=true
user=nginx
```


ssh-keygen -t rsa
ssh-copy-id ip  //ip为kvm部署的IP
ssh ip   -L localhost:8000:localhost:8000 -L localhost:6080:localhost:60

#设置nginx、supervisord开机自启动并启动它们<br>
```
chkconfig nginx on
service nginx start
/usr/sbin/setsebool httpd_can_network_connect true
```
```
chkconfig supervisord on<br>
service supervisord start
```
```
cd /var/www/webvirtmgr
sudo git pull
sudo ./manage.py collectstatic
sudo service supervisord restart
```


---
```
curl http://retspen.github.io/libvirt-bootstrap.sh | sh
service messagebus restart
service libvirtd restart
```


```
adduser webvirtmgr
passwd webvirtmgr
usermod -G qemu,kvm -a webvirtmgr
/etc/polkit-1/localauthority/50-local.d/50-libvirt-remote-access.pkla   （webvirtmgr为所属者)
[Remote libvirt SSH access]
Identity=unix-user:webvirtmgr
Action=org.libvirt.unix.manage
ResultAny=yes
ResultInactive=yes
ResultActive=yes
```
```
su - nginx -s /bin/bash
ssh-keygen
touch ~/.ssh/config && echo -e "StrictHostKeyChecking=no\nUserKnownHostsFile=/dev/null" >> ~/.ssh/config
chmod 0600 ~/.ssh/config
ssh-copy-id webvirtmgr@${libvirt_host_ip}
```
