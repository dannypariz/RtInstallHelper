# INSTALL JFROG ARTIFACTORY (HA-CLUSTER WITH THREE NODES) ON LINUX IN GCP
## CREATE 5 VMs in GCP
You can do this the way that is most convenient for you. Ensure you have the machines up and running in the same VPC, DNS resolution works (Cloud DNS) and SSH is configured so you can access the machines. 
You may assign the following roles to the machines: 

    1.) COMPUTE1: NGINX
    2.) COMPUTE2: ARTIFACTORY-NODE-1
    3.) COMPUTE3: ARTIFACTORY-NODE-2
    4.) COMPUTE4: ARTIFACTORY-NODE-3
    5.) COMPUTE5: DATABASE

As the database must be available at installation time of Artifactory, you may want to start with the database installation.

For the ease of use, all commands are performed as root ("sudo -i").

## INSTALL THE DATABASE - POSTGRESQL DATABASE INSTALLATION (UBUNTU 20.04 LTS FOCAL FOSSA GCP MINIMAL)
### install database and additional tools
    apt-get update && apt-get install postgresql rsync net-tools vim -y

### stop database
    service postgresql stop

### configure additional disks
To avoid your database server to stop due to a full disk, you may want to add another (bunch of) virtual disks. The flexibility of LVM2 allows you to start with a small one while having the possibility to provision additional storage at a later point in time without any service interruptions. 
You may decide to create a folder "/data" (or whatever you consider a good path). Then: 

    1.) create a physical volume on each of your additional disks (e.g. "/dev/sdb",...): 
    2.) create a volume group (e.g. "data") and add your physical volume to it
    3.) create a logical volume (e.g. "storage") in the volume group data; assign all available diskspace to it
    4.) format the logical volume with an appropriate filesystem (e.g. "ext4")
    5.) determine the disk UUID and add it to the "/etc/fstab" file to ensure it will be mounted at boot time
    6.) mount the logical volume

Commands that may help you to achieve this:     

    mkdir /data
    pvcreate /dev/sdb
    vgcreate data /dev/sdb
    lvcreate -l 100%FREE -n storage data 
    mkfs.ext4 /dev/mapper/data-storage
    myUUID=$(blkid | grep data-storage | awk '{ print $2 }' | sed -r 's/["]+//g')
    myDefinition=" /data ext4 defaults 0 2"
    myFstabConfig="$myUUID$myDefinition"
    echo $myFstabConfig >> /etc/fstab
    mount -a

### copy database data to alternative path
    rsync -av /var/lib/postgresql /data

### backup original postgresql.conf
    cp /etc/postgresql/12/main/postgresql.conf /etc/postgresql/12/main/postgresql.conf.bak

### determine ip address of your database server
The IP address will be required to define on what network interfaces the database will provide services; you can use a wildcard or be more specific configuring your ip address and - important - localhost. Start determining your ip address: 

    <myIpAddress>: ip a | grep inet | grep scope | grep global | awk '{ print $4 }'

### perform changes in postgresql.conf
- documentation: https://www.jfrog.com/confluence/display/JFROG/PostgreSQL

You will add the data_directory path according to your needs and configuration. In this example the data directory will reside on the LVM logcial volume configured above. You can chose whatever port you want (default is 5432). 

    vim /etc/postgresql/12/main/postgresql

        => CHANGE: data_directory = '/data/postgresql/12/main'
        => CHANGE: listen_addresses = '<myIpAddress>, localhost'
        => CHANGE: port = 10433

### backup pg_hba.conf
    cp /etc/postgresql/12/main/pg_hba.conf /etc/postgresql/12/main/pg_hba.conf.bak

### perform changes in pg_hba.conf
Now that you configured the database service, you need to configure who has access to the database service. You may want to restrict access to machines from your local subnet only. The single database you want to make externally available is the artifactory database ("artifactory"). The single user you want to grant access, is the user "artifactory".

    echo "host artifactory artifactory <mySubnet> md5" >> /etc/postgresql/12/main/pg_hba.conf

### start the database
    service postgresql start

### remove leftovers from copy job
    rm -rf /var/lib/postgresql/12/main/

### prepare database for artifactory
#### login with predefined admin user "postgres"
    sudo -u postgres psql

#### create user "artifactory" as service account for artifactory nodes
    CREATE USER artifactory WITH PASSWORD '<myPassword>';
    CREATE DATABASE artifactory WITH OWNER=artifactory ENCODING='UTF8';
    GRANT ALL PRIVILEGES ON DATABASE artifactory TO artifactory;

#### validate user
    psql -U artifactory -h <myIpAddress> -p 10433 artifactory

## FINISH THE DATABASE INSTALLATION
Once this test was successfully performed, the database installation is successfully finished. Go ahead and install the artifactory nodes. 


## INSTALL ARTIFACTORY CLUSTER NODES
### install database connector and additional tools
apt-get update && apt-get install libpostgresql-jdbc-java net-tools vim -y

### add jfrog repositories
    wget -qO - https://releases.jfrog.io/artifactory/api/gpg/key/public | sudo apt-key add -;
    echo "deb https://releases.jfrog.io/artifactory/artifactory-pro-debs focal main" | sudo tee -a /etc/apt/sources.list;
    apt-get update

### install artifactory pro
    apt-get install jfrog-artifactory-pro -y

### configure the datastore for artifactory 
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

### configure jdbc connector for postgresql 
- documentation: https://www.jfrog.com/confluence/display/JFROG/PostgreSQL

    ln -s /usr/share/java/postgresql.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/postgresql.jar
    ln -s /usr/share/java/postgresql.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/postgresql-42.2.10.jar
    ln -s /usr/share/java/postgresql.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/postgresql-jdbc3.jar
    ln -s /usr/share/java/postgresql.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/postgresql-jdbc4.jar
    ln -s /usr/share/java/libintl.jar /opt/jfrog/artifactory/var/bootstrap/artifactory/tomcat/lib/libintl.jar

### backup system.yaml
    cp /opt/jfrog/artifactory/var/etc/system.yaml /opt/jfrog/artifactory/var/etc/system.yaml.bak

### set up system.yaml
- documentation: https://www.jfrog.com/confluence/display/JFROG/Artifactory+System+YAML

The first node needs neither a join key nor a master key. These keys are being created during the first start of the service. The join key is then available in the Artifactory Web UI under =>Security=>Settings; the join key is not visible unless you unlock it with your password. You will need the database password you created above. 

#### first node in the cluster
    vim /opt/jfrog/artifactory/var/etc/system.yaml

in the "node" section: 
    CHANGE => haEnabled: true

in the "database" section: 
    CHANGE => url: "jdbc:postgresql://fqdn:port/database"
    username: artifactory
    password: <myPassword>

#### all other nodes in the cluster (first node must be up and running)
- documentation: https://www.jfrog.com/confluence/display/JFROG/Managing+Keys#ManagingKeys-CreatingYourKeys

    vim /opt/jfrog/artifactory/var/etc/system.yaml

in the "security" section: 
    CHANGE => joinKey "<myJoinKeyFromTheWebUI>"

in the "node" section: 
    CHANGE => haEnabled: true

in the "database" section: 
    CHANGE => url: "jdbc:postgresql://fqdn:port/database"
    username: artifactory
    password: <myPassword>

### provide master key (MUST NOT BE DONE for the first node in the cluster but for all others)
- documentation: https://www.jfrog.com/confluence/display/JFROG/Managing+Keys#ManagingKeys-CreatingYourKeys

You must pick the master key from the first cluster node. The master key resides in "/opt/jfrog/artifactory/var/etc/security/master.key". You then create the security folder and the master key on every node you are about to add: 

On the first cluster node: 
    <myMasterKey>: cat /opt/jfrog/artifactory/var/etc/security/master.key

On the next cluster node(s):
    
    mkdir /opt/jfrog/artifactory/var/etc/security
    echo "<myMasterKey>" > /opt/jfrog/artifactory/var/etc/security/master.key
    chown -R artifactory:artifactory /opt/jfrog/artifactory/var/etc/security

## FINISH THE ARTIFACTORY INSTALLATION
You may want to verify your cluster nodes are up and running in the Web UI under =>Monitoring=>Service Status. Some logs have proven to be extremely helpful: 
    - /opt/jfrog/artifactory/var/log/tomcat/tomcat-catalina*.log
    - /opt/jfrog/artifactory/var/log/artifactory-service.log

Once all nodes are up and running, the artifactory installation is done. Go ahead and install the nginx node. 

    
## NGINX NODE CONFIGURATION
###
    apt-get update && apt-get install vim nginx -y

### adapt standard config
    vim /etc/nginx/nginx.conf
    => CHANGE: comment or delete the following line: include /etc/nginx/sites-enabled/*;

### configure nginx to serve the cluster nodes
    => CHANGE: artifactory.conf is automatically created in WebUI, it can be downloaded and must be copied to /etc/nginx/conf.d/

## FINISH THE NGINX INSTALLATION
Now you are done. 
