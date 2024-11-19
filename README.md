# openvpn-monitor

Fork from the original work by `furlongm`. All credits to the original author:
https://github.com/furlongm/openvpn-monitor

This repo aims to update dependencies and keep a working `openvpn-monitor` solution in newer systems.


## Summary

openvpn-monitor is a simple python program to generate html that displays the
status of an OpenVPN server, including all current connections. It uses the
OpenVPN management console. It typically runs on the same host as the OpenVPN
server, however it does not necessarily need to.

[![](https://raw.githubusercontent.com/furlongm/openvpn-monitor/gh-pages/screenshots/openvpn-monitor.png)](https://raw.githubusercontent.com/furlongm/openvpn-monitor/gh-pages/screenshots/openvpn-monitor.png)


## Supported Operating Systems
  - Ubuntu 20.04 LTS (focal)
  - Debian 10, 11, 12
  - CentOS/RHEL 8


## Source

The current source code is available on github:

https://github.com/ynad/openvpn-monitor


## Install Options

  - [venv + pip + gunicorn](#venv--pip--gunicorn)
  - [apache](#apache)
  - [docker](#docker)
  - [nginx + uwsgi](#nginx--uwsgi)
  - [deb/rpm](#deb--rpm)

N.B. all CentOS/RHEL instructions assume the EPEL repository has been installed:

```shell
dnf -y install epel-release

```

If selinux is enabled the following changes are required for host/port to work:

```
dnf -y install policycoreutils-python-utils
semanage port -a -t openvpn_port_t -p tcp 5555
setsebool -P httpd_can_network_connect=1
```

---

### venv + pip + gunicorn

```shell
apt -y install python3-venv geoip-database   # (debian/ubuntu)
dnf -y install python3-venv geolite2-city    # (centos/rhel)
```

#### 1) Checkout `openvpn-monitor`

```shell
cd /srv/
git clone https://github.com/ynad/openvpn-monitor.git
cd openvpn-monitor
```

#### 2) Set-up venv and install pip requirements
```shell
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements_gunicorn.txt
```

#### 3) Gunicorn manual run
```shell
gunicorn openvpn-monitor -b 0.0.0.0:8080
```

#### 4) Gunicorn systemd service
```shell
sudo cp gunicorn-openvpn-monitor.service.example /etc/systemd/system/gunicorn-openvpn-monitor.service
sudo systemctl enable gunicorn-openvpn-monitor.service
sudo systemctl restart gunicorn-openvpn-monitor.service
```

See [configuration](#configuration) for details on configuring openvpn-monitor.

---

### apache

#### 1) Checkout `openvpn-monitor`

```shell
cd /var/www/html
git clone https://github.com/ynad/openvpn-monitor.git
```

#### 2) Install dependencies and configure apache

##### Debian / Ubuntu

```shell
apt -y install git apache2 libapache2-mod-wsgi python3-geoip2 python3-humanize python3-bottle python3-semantic-version geoip-database geoip-database-extra
echo "WSGIScriptAlias /openvpn-monitor /var/www/html/openvpn-monitor/openvpn-monitor.py" > /etc/apache2/conf-available/openvpn-monitor.conf
a2enconf openvpn-monitor
systemctl restart apache2
```

##### CentOS / RHEL

```shell
dnf -y install git httpd mod_wsgi python3-geoip2 python3-humanize python3-bottle python3-semantic_version geolite2-city
echo "WSGIScriptAlias /openvpn-monitor /var/www/html/openvpn-monitor/openvpn-monitor.py" > /etc/httpd/conf.d/openvpn-monitor.conf
systemctl restart httpd
```

See [configuration](#configuration) for details on configuring openvpn-monitor.

---

### docker
(out-of-date) Work by https://github.com/ruimarinho/docker-openvpn-monitor
```shell
docker run -p 80:80 ruimarinho/openvpn-monitor
```

Read the [docker installation instructions](https://github.com/ruimarinho/docker-openvpn-monitor#usage)
for details on how to generate a dynamic configuration using only environment
variables.

---

### nginx + uwsgi

#### 1) Install dependencies

```shell
apt -y install git gcc nginx uwsgi uwsgi-plugin-python3 python3-venv python3-dev libgeoip-dev geoip-database   # (debian/ubuntu)
dnf -y install git gcc nginx uwsgi uwsgi-plugin-python3 python3-venv python3-devel geoip-devel geolite2-city   # (centos/rhel)
```

#### 2) Checkout `openvpn-monitor`

```shell
cd /srv/
git clone https://github.com/ynad/openvpn-monitor.git
cd openvpn-monitor
```

#### 3) Set-up venv and install pip requirements
```shell
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
```

#### 4) uWSGI app config

Create a uWSGI config: `/etc/uwsgi/apps-available/openvpn-monitor.ini`

```
[uwsgi]
base = /srv
project = openvpn-monitor
logto = /var/log/uwsgi/app/%(project).log
plugins = python3
chdir = %(base)/%(project)
virtualenv = %(chdir)
module = openvpn-monitor:application
manage-script-name = true
mount=/openvpn-monitor=openvpn-monitor.py
```

#### 5) Nginx site config

Create an Nginx config: `/etc/nginx/sites-available/openvpn-monitor`

```
server {
    listen 80;
    location /openvpn-monitor/ {
        uwsgi_pass unix:///run/uwsgi/app/openvpn-monitor/socket;
        include uwsgi_params;
    }
}
```

#### 6) Enable uWSGI app and Nginx site, and restart services

```shell
ln -s /etc/uwsgi/apps-available/openvpn-monitor.ini /etc/uwsgi/apps-enabled/
systemctl restart uwsgi
ln -s /etc/nginx/sites-available/openvpn-monitor /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
systemctl restart nginx
```

See [configuration](#configuration) for details on configuring openvpn-monitor.

---

### deb / rpm

```shell
TBD
```

---

## Configuration

### Configure OpenVPN

Add the following line to your OpenVPN server configuration to run the
management console on 127.0.0.1 port 5555, with the management password
in /etc/openvpn/pw-file:

```
management 127.0.0.1 5555 pw-file
```

To run the management console on a socket, with the management password
in /etc/openvpn/pw-file:

```
management socket-name unix pw-file
```

Refer to the OpenVPN documentation for further information on how to secure
access to the management interface.

---

### Configure openvpn-monitor

Copy the example configuration file `openvpn-monitor.conf.example` to the same
directory as openvpn-monitor.py.

```shell
cp openvpn-monitor.conf.example openvpn-monitor.conf

```

In this file you can set site name, add a logo, set the default map location
(latitude and longitude). If not set, the default location is New York, USA.

Once configured, navigate to `http://myipaddress/openvpn-monitor/`

Note the trailing slash, the images may not appear without it.

---

### Debugging

openvpn-monitor can be run from the command line in order to test if the html
generates correctly:

```shell
cd /var/www/html/openvpn-monitor
python3 openvpn-monitor.py
```

Further debugging can be enabled by specifying the `--debug` flag:

```shell
cd /var/www/html/openvpn-monitor
python3 openvpn-monitor.py -d
```

---

## License

openvpn-monitor is licensed under the GPLv3, a copy of which can be found in
the COPYING file.


## Acknowledgements

Flags are created by Matthias Slovig (flags@slovig.de) and are licensed under
Creative Commons License Deed Attribution-ShareAlike 3.0 Unported
(CC BY-SA 3.0). See http://flags.blogpotato.de/ for more details.
