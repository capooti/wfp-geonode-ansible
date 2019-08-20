# GeoNode manual installation on CentOS 7

For the rest of this instructions we will assume that our instance is running at 192.168.100.10

## System requirements

Install system requirements:

```
sudo yum -y update
sudo yum -y install epel-release
sudo yum -y install gcc gcc-c++ git python-pip python-virtualenv gdal gdal-devel java-1.8.0-openjdk postgresql-server postgresql-contrib postgresql-devel postgis
```

The following system requirements aren't mandatory, but useful:

```
sudo yum install mlocate
```

## PostgreSQL/PostGIS

Install requirements for PostgreSQL:

```
sudo yum install -y
sudo postgresql-setup initdb
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Create the user and the database:

```
sudo su postgres
psql
create role wfp with login superuser password 'wfp';
create database geonode_data with owner wfp;
\c geonode_data
create extension postgis
```

Enable md5 connections by editing the /var/lib/pgsql/data/pg_hba.conf file:

Find the lines that looks like this, near the bottom of the file:

```
host    all             all             127.0.0.1/32            ident
host    all             all             ::1/128                 ident
```

Then replace "ident" with "md5", so they look like this:

```
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

## GeoNode and GeoNode project

Install GeoNode from source:

```
pip install --upgrade pip
git clone https://github.com/GeoNode/geonode.git
virtualenv --no-site-packages env
. env/bin/activate
pip install -r geonode/requirements.txt
pip install pygdal==1.11.4.5
cd geonode
pip install -e .
paver setup
paver sync
```

Install WFP GeoNode project from source:

```
git clone https://github.com/wfp-ose/wfp_geonode-2.10.git
```


Set Env variables (create a vars file and source it):

```
export DEFAULT_BACKEND_DATASTORE=data
export GEODATABASE_URL=postgis://wfp:wfp@localhost:5432/geonode_data
export ALLOWED_HOSTS="['localhost', '192.168.100.10',]"
export STATIC_ROOT=/home/vagrant/www/geonode/static/
export GEOSERVER_LOCATION=http://192.168.100.10:8080/geoserver/
export GEOSERVER_PUBLIC_LOCATION=http://192.168.100.10/geoserver/
export GEOSERVER_ADMIN_PASSWORD=geoserver
```

Start GeoNode and GeoServer:

```
paver start_geoserver
./manage.py runserver 0.0.0.0:8000
```

## nginx

Install nginx and enable it for starting at boot:

```
sudo yum -y install nginx
sudo systemctl enable nginx
```

Create the directory for static files:

```
cd ~
chmod og+x .
mkdir -p www/geonode/static
./manage.py collectstatic
```

Use the configuration sample provided in config/nginx.sample to create /etc/nginx/conf.d/geonode.conf.
Then run the following commmands:

```
setsebool -P httpd_can_network_connect 1
sudo setenforce 0
```

Now restart nginx:

```
sudo systemctl restart nginx
```

To debug in case of errors:

```
sudo tail -f /var/log/nginx/error.log
```

## uwsgi

This section describe how to setup the GeoNode service using uwsgi.

Create a geonode.ini file for uwsgi in the home directory (~/geonode.ini).
The file should look like config/uwsgi.sample - change it accordingly.

Now - with the virtualenv activated - test if the configuration is correct (you can check for errors in /tmp/geonode.log)

```
uwsgi --ini geonode.ini
```

If GeoNode correctly works, you can install it as a service using the provided config/geonode.service script. Copy the content of this script (and adapt to your needs) in /etc/systemd/system/geonode.service


Now you can start GeoNode by this command (and you can enable it for automatic start at reboot):

```
sudo systemctl start geonode
sudo systemctl enable geonode
```

## Tomcat and GeoServer

Install system requirements:

```
sudo yum install java-1.8.0-openjdk-devel
```

Create an user for Tomcat:

```
sudo useradd -m -U -d /opt/tomcat -s /bin/false tomcat
```

Download and install Tomcat:

```
cd /tmp/
wget https://www-eu.apache.org/dist/tomcat/tomcat-9/v9.0.22/bin/apache-tomcat-9.0.22.tar.gz
tar -xf apache-tomcat-9.0.22.tar.gz
sudo mv apache-tomcat-9.0.22 /opt/tomcat/
sudo ln -s /opt/tomcat/apache-tomcat-9.0.22 /opt/tomcat/latest
sudo chown -R tomcat: /opt/tomcat
sudo sh -c 'chmod +x /opt/tomcat/latest/bin/*.sh'
```

Create a systemd unit file in /etc/systemd/system/tomcat.service, using the provided config/tomcat.service script.

Enable and start Tomcat:

```
sudo systemctl daemon-reload
sudo systemctl enable tomcat
sudo systemctl start tomcat
```

Deploy geoserver.war to Tomcat:

```
cp /home/vagrant/geonode/downloaded/geoserver-2.15.2.war /opt/tomcat/latest/webapps/geoserver.war
```

Stop Tomcat and copy data directory in /home/vagrant/gs_data.

Setup GeoNode OAuth parameters with correct IP (in our case 192.168.100.10) in GeoServer and Django OAuth (using admin)
