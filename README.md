# TheHive-and-Cortex-Installation
This repository contains the easiest method of installation and configuration of TheHive and Cortex
> **Note**
> I am using Ubuntu 22.0 for this installation.


Table of Content
# Table of contents
1. [TheHive Installation](#thehive-installation)
    1. [Install Java VM](#install-java-vm)
    2. [Install Cassandra](#install-cassandra)
    3. [Cassandra Configuration](#cassandra-configuration)
    4. [Install TheHive](#install-thehive)
    5. [TheHive Configuration](#thehive-configuration)
        1. [Database and Indexes](#database-and-indexes)
        2. [File Storage](#file-storage)
    6. [Access TheHive on Machine/Server IP](#access-thehive-on-machine-ip)
2. [Cortex Installation](#cortex-installation)
    1. [Configure Elasticsearch](#configure-elasticsearch)
    2. [Install Cortex](#install-cortex)
3. [Integrate Cortex with TheHive](#integrate-cortex-with-thehive)

# TheHive Installation
Before TheHive Installation, we have to install few pre-requisites such cassandra for database, Java Vm, and elasticsearch for indexing. But we will not be using elasticsearch for indexing.

## Install Java VM

```
apt-get install -y openjdk-8-jre-headless 
echo JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64" >> /etc/environment 
export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
```
## Install Cassandra
```
curl -fsSL https://www.apache.org/dist/cassandra/KEYS | sudo apt-key add - 
echo "deb http://www.apache.org/dist/cassandra/debian 311x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list
sudo apt update 
sudo apt install cassandra
```
By default, data is stored in **/var/lib/cassandra**.

## Cassandra Configuration
Start by changing the cluster_name with thp. Run the command cqlsh:
```
cqlsh localhost 9042
cqlsh> UPDATE system.local SET cluster_name = 'thp' where key='local';
```
Exit and then run:
```
nodetool flush
```
Verify if the cluster_name is set to **thp** in **/etc/cassandra/cassandra.yaml file**.
```nano /etc/cassandra/cassandra.yaml```
```
# content from /etc/cassandra/cassandra.yaml
cluster_name: 'thp'
listen_address: 'xx.xx.xx.xx' # address for nodes
rpc_address: 'xx.xx.xx.xx' # address for clients
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          # Ex: "<ip1>,<ip2>,<ip3>"
          - seeds: 'xx.xx.xx.xx' # self for the first node
data_file_directories:
  - '/var/lib/cassandra/data'
commitlog_directory: '/var/lib/cassandra/commitlog'
saved_caches_directory: '/var/lib/cassandra/saved_caches'
hints_directory: 
  - '/var/lib/cassandra/hints'
```
Then restart the service:
```
service cassandra restart
```
By default Cassandra listens on 7000/tcp (inter-node), 9042/tcp (client).

## Install TheHive

I am installing stable version here.
```
echo 'deb https://deb.thehive-project.org release main' | sudo tee -a /etc/apt/sources.list.d/thehive-project.list 
sudo apt-get update 
sudo apt-get install thehive4
```

## TheHive Configuration
We will do configuration for Database, Indexes, and File Storage System. In order to do so, edit application.conf file of TheHive by ```nano /etc/thehive/application.conf```. 

### Database and Indexes

Firstly, create a directory for indexing, and give permissions to it.
```
mkdir /opt/thp/thehive/index
chown thehive:thehive -R /opt/thp/thehive/index
```
after that, edit the configuration file ```nano /etc/thehive/application.conf```

```
db {
  provider: janusgraph
  janusgraph {
    ## Storage configuration
    storage {
      #below section is for cassandra
      backend: cql
      hostname: ["127.0.0.1"]
      cql {
        cluster-name: thp
        keyspace: thehive
      }
    }
    ## Index configuration
    index {
      search {
        #below two lines will be commented, uncomment these and paste the directory here.
        backend : lucene
        directory:  /opt/thp/thehive/index
        #comment down below three lines as these are for elasticsearch
      }
    }
  }
}
```

Save the file by CTRL+x, press y.

### File Storage

In official documentations, this step is described before the installation of TheHive, but I prefer to do this step after installation of the TheHive.

To store files on the local filesystem, start by choosing the dedicated folder:

```
mkdir -p /opt/thp/thehive/files
chown -R thehive:thehive /opt/thp/thehive/files
```
Now open the configuraiton again by ```nano /etc/thehive/application.conf``` and make sure the below lines are uncomment and location direcotry is correct as created above.
```
## Attachment storage configuration
storage {
  ## Local filesystem
  provider: localfs
  localfs {
    location: /opt/thp/thehive/files
  }
}
```

Save the file and run the TheHive.

```
service thehive start
```
Visit http://127.0.0.1:9000

## Access TheHive on Machine/Server IP <a name="access-thehive-on-machine-ip"></a>

By default, TheHive will work only on the machine on which it is installed, means on the localhost e.g http://127.0.0.1:9000. But to access it on the server/host ip like http://192.168.1.x:9000, we have to add a new line in the configuration file.

Add the following line in /etc/thehive/application.conf

```
http.address = 0.0.0.0
```
Now you can access TheHive on your entire network, also from different machines by visiting http://YOUR_MACHINE_IP:9000
> **Note**
> Here, YOU_MACHINE_IP is the IP address on which TheHive is installed.

# Cortex Installation

Before Installing Cortex, Elasticsearch must be installed. You can install elasticsearch easily by following [tihs guide](https://www.geeksforgeeks.org/how-to-install-and-configure-elasticsearch-on-ubuntu/).

## Configure Elasticsearch

Edit elasticsearch.yml by ```nano /etc/elasticsearch/elasticsearch.yml``` and do the following:

* Uncomment and change ```network.host: YOUR_MACHINE_IP```

* Uncomment and change ```cluster.name: hive```

* Uncomment ```node.name: node-1```

* Uncomment and change ```cluster.initial_master_nodes: ["node-1"]```

* Add this new line ```thread_pool.search.queue_size: 100000```

Save the file and restart elasticsearch by, 

```service elasticsearch restart```

Elasticsearch is now running on YOUR_MACHINE_IP instead localhost or 127.0.0.1.

## Install Cortex

```
curl https://raw.githubusercontent.com/TheHive-Project/TheHive/master/PGP-PUBLIC-KEY | sudo apt-key add -
echo 'deb https://deb.thehive-project.org stable main' | sudo tee -a /etc/apt/sources.list.d/thehive-project.list
sudo apt-get update
```

Cortex has been installed. Before starting, we need to generate a secret key.

```
sudo mkdir /etc/cortex
(cat << _EOF_
# Secret key
# ~~~~~
# The secret key is used to secure cryptographics functions.
# If you deploy your application to several instances be sure to use the same key!
play.http.secret.key="$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1)"
_EOF_
) | sudo tee -a /etc/cortex/application.conf
```
A secret key has been generated and placed at the end of the file in **/etc/cortex/application.conf**.

Edit the config file of cortex by ```nano /etc/cortex/application.conf```

Search for Elasticsearch section and change the uri of elasticsearch in from ```127.0.0.1``` to ```YOUR_MACHINE_IP```. So the Cortex will now connect with elasticsearch which is listening on YOUR_MACHINE_IP:9200. 

Start the Cortex by,
```
systemctl start cortex.
```

Tail the log of cortex by ```tail -f /var/log/cortex/application.log``` and look if everything is fine.

Finally, visit to ```http://YOUR_MACHINE_IP:9001``` to access Cortex GUI. Update the database and you are good to go.

# Integrate Cortex with TheHive

* Create an organization in Cortex.
* Create a user for API with read and analyze privileges in the organization.
* Generate an API for the created user and copy it.

Go to TheHive configuration file ```by nano /etc/thehive/application.conf``` 

Go to Cortex section and change the following,
```
name: "CORTEX1" ## can by anything
url: YOUR_MACHINE_IP:9001 ##This is the Cortex url

key: "PASTE_THE_COPIED_API_KEY_HERE"
```

Save the file and restart TheHive.
```
Systemctl resart TheHive
```

Go to TheHive ```YOUR_MACHINE_IP:9000``` and click on the name icon on top right and select **About** option in dropdown. You will see a Crotex there if everything is fine.

TheHive and Cortex installation and integration has been completed.

---
> GitHub [@Salman7870](https://github.com/Salman7870/) &nbsp;&middot;&nbsp;
> LinkedIn [@muhammad-sulaiman7870](https://www.linkedin.com/in/muhammad-sulaiman7870/)
