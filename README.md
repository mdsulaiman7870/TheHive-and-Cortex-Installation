# TheHive-and-Cortex-Installation
This repository contains the easiest method of installation and configuration of TheHive and Cortex
> **Note**
> I am using Ubuntu 22.0 for this installation.

# TheHive Installation
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

## Install TheHive:

I am installing stable version here.
```
echo 'deb https://deb.thehive-project.org release main' | sudo tee -a /etc/apt/sources.list.d/thehive-project.list 
sudo apt-get update 
sudo apt-get install thehive4
```

## THeHive Configuration
We will do configuration for Database, Indexes, and File Storage System. In order to do so, edit application.conf file of TheHive by ```nano /etc/thehive/application.conf```. 

### Database and Indexes

Firstly, create a directory for indexing, and give permissions to it.
```
mkdir /opt/thp/thehive/index
chown thehive:thehive -R /opt/thp/thehive/index
```
after that, edit the configuration file ```ano /etc/thehive/application.conf```

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

```service thehive start```
Visit http://127.0.0.1:9000

---
> GitHub [@Salman7870](https://github.com/Salman7870/) &nbsp;&middot;&nbsp;
> LinkedIn [@muhammad-sulaiman7870](https://www.linkedin.com/in/muhammad-sulaiman7870/)

