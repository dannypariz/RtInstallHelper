
### POSTGRESQL DATABASE NODE PARTITIONING 
mkdir /data
pvcreate /dev/sdb
vgcreate data /dev/sdb
lvcreate -l 100%FREE -n storage data 
mkfs.ext4 /dev/mapper/data-storage
mount /dev/mapper/data-storage /data
myUUID=$(blkid | grep data-storage | awk '{ print $2 }' | sed -r 's/["]+//g')
myDefinition=" /data ext4 defaults 0 2"
myFstabConfig="$myUUID$myDefinition"
echo $myFstabConfig >> /etc/fstab
mount -a

### POSTGRESQL DATABASE INSTALLATION (UBUNTU 20.04 LTS FOCAL FOSSA GCP MINIMAL)
## install database and additional tools
apt-get install postgresql rsync net-tools vim -y
## stop database
service postgresql stop
## copy database data to alternative path
rsync -av /var/lib/postgresql /data
## backup postgresql.conf
cp /etc/postgresql/12/main/postgresql.conf /etc/postgresql/12/main/postgresql.conf.bak
## perform changes in postgresql.conf
=> CHANGE: data_directory = '/data/postgresql/12/main'
=> CHANGE: listen_addresses = '10.156.0.13, localhost'
=> CHANGE: port = 10433
## backup pg_hba.conf
cp /etc/postgresql/12/main/pg_hba.conf /etc/postgresql/12/main/pg_hba.conf.bak
## perform changes in pg_hba.conf
=> CHANGE: access for planned artifactory instances, details: https://www.postgresql.org/docs/12/auth-pg-hba-conf.html
=> CHANGE: local all postgres md5 [from peer in the original file]
echo "host artifactory artifactory 10.156.0.0/24 md5" >> /etc/postgresql/12/main/pg_hba.conf
## start database
service postgresql start
## remove leftovers from copy job
rm -rf /var/lib/postgresql/12/main/
## prepare database for artifactory
# login with predefined user
sudo -u postgres psql
# set admin password
ALTER USER postgres with encrypted password 'EmGuemH;Ps139';
CREATE USER artifactory WITH PASSWORD 'EmGuemH;Ps139';
CREATE DATABASE artifactory WITH OWNER=artifactory ENCODING='UTF8';
GRANT ALL PRIVILEGES ON DATABASE artifactory TO artifactory;
# validate user
psql -U artifactory -h 127.0.0.1 -p 10433 artifactory

## ARTIFACTORY INSTANCES PARTITIONING
pvcreate /dev/sdb
vgcreate data /dev/sdb
lvcreate -l 100%FREE -n storage data 
mkfs.ext4 /dev/mapper/data-storage
mount /dev/mapper/data-storage /data
myUUID=$(blkid | grep data-storage | awk '{ print $2 }' | sed -r 's/["]+//g')
myDefinition=" /var/opt ext4 defaults 0 2"
myFstabConfig="$myUUID$myDefinition"
echo $myFstabConfig >> /etc/fstab
mount -a

### ARTIFACTORY NODE PREPARATION
## install database connector and additional tools
apt-get install libpostgresql-jdbc-java net-tools vim -y
## add jfrog repositories
wget -qO - https://releases.jfrog.io/artifactory/api/gpg/key/public | sudo apt-key add -;
echo "deb https://releases.jfrog.io/artifactory/artifactory-pro-debs focal main" | sudo tee -a /etc/apt/sources.list;
apt-get update
## install artifactory pro
apt-get install jfrog-artifactory-pro -y

### ARTIFACTORY NODE CONFIGURATION
## configure jdbc connector for postgresql
ln -s /usr/share/java/postgresql.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/postgresql.jar
ln -s /usr/share/java/postgresql.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/postgresql-42.2.10.jar
ln -s /usr/share/java/postgresql.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/postgresql-jdbc3.jar
ln -s /usr/share/java/postgresql.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/postgresql-jdbc4.jar
ln -s /usr/share/java/libintl.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/libintl.jar

## backup system.yaml
cp /opt/jfrog/artifactory/var/etc/system.yaml /opt/jfrog/artifactory/var/etc/system.yaml.bak
## set up system.yaml
# https://www.jfrog.com/confluence/display/JFROG/Artifactory+System+YAML
vim /opt/jfrog/artifactory/var/etc/system.yaml
# shared
javaHome: "/opt/jfrog/artifactory/app/third-party/java/"
extraJavaOpts: "-Xms512m -Xmx2g"
# security
joinKey: "<FROM RT WEBUI>"
masterKeyFile: "/opt/jfrog/artifactory/var/etc/security/master.key"
# node
# primary: true => Setting is deprecated
haEnabled: true
# database
type: postgresql
driver: org.postgresql.Driver
url: "jdbc:postgresql://fqdn:port/database"
username: artifactory
password: <primary node: cleartext, additional nodes copy encrypted password from primary node>
# CAREFUL
=> master.key is automatically generated on primary node, must be manually transferred to additional cluster members

### NGINX NODE CONFIGURATION
apt-get install nginx -y

## ADAPT STANDARD CONFIG
# uncomment the include statement in /etc/nginx/nginx.conf
=> CHANGE: uncomment include /etc/nginx/sites-enabled/*;
# save artifactory.conf in /etc/nginx/conf.d
=> CHANGE: artifactory.conf is automatically created in WebUI



