* Install MongoDB Enterprise Advanced 4.0 using the following commands:

  | Purpose | Command |
  | ------- | ------- |
  | Add MongoDB Public Key | `wget -qO - https://www.mongodb.org/static/pgp/server-4.0.asc \| sudo apt-key add -` |
  | Add MongoDB EA Repo | `echo "deb [ arch=amd64 ] http://repo.mongodb.com/apt/ubuntu bionic/mongodb-enterprise/4.0 multiverse" \| sudo tee /etc/apt/sources.list.d/mongodb-enterprise.list` |
  | Update Apt Package Cache | `sudo apt-get update` |
  | Install MongoDB EA 4.0.15 | `sudo apt-get install -y mongodb-enterprise=4.0.15 mongodb-enterprise-server=4.0.15 mongodb-enterprise-shell=4.0.15 mongodb-enterprise-mongos=4.0.15 mongodb-enterprise-tools=4.0.15` |

* Edit the MongoDB Enterprise config file by running `sudo vi /etc/mongod.etc` update the binding, add it to a replica set, and enable username and password auth as shown below or in the [sample config](../09-gcp-lm/mongod.conf).

```
    # allow connections to any local IP address
    net:
        port: 27017
        bindIp: 0.0.0.0
    # make this node part of a replica set
    replication:
        replSetName: rsMigration
    # enable basic SCRAM username/password auth
    security:
        authorization: enabled
```
* Run the following commands to start MongoDB: 

  | Purpose | Command |
  | ------- | ------- |
  | Start mongod | `sudo service mongod start` |
  | Initiate the rep set | `mongo --eval "rs.initiate()"` |
  | Add root user for administration | `mongo --eval "db = db.getSisterDB('admin');db.createUser({user:'root',pwd:'root123',roles:['root']});"` |
  | Add app user | `mongo -u root -p root123 --eval "db = db.getSisterDB('admin');db.createUser({user:'appUser',pwd:'appUser123',roles:['clusterMonitor','readWriteAnyDatabase']});"`|
  | Get rep set config | `mongo -u root -p root123 --eval "rs.config()"` |
  
  * Run the following commands to setup the load generator (POCDriver)

  | Purpose | Command |
  | ------- | ------- |
  | Update Apt package cache | `sudo apt-get update` |
  | Install Java | `sudo apt install -y default-jre` |
  | Download POC Driver | `wget https://github.com/johnlpage/POCDriver/raw/master/bin/POCDriver.jar` |
  
  * Begin to generate data on your cluster by running the following command: `java -jar POCDriver.jar -c "mongodb://appUser:appUser123@XXXX" -t 1 -e -d 60 -f 25 -a 5:5 --depth 2 -x 3` and replace the __XXXX__ with the address of the cluster
  
  * Within the CloudShell session of your `mongodb` compute instance, run the following commands to prepare `mongomirror`

| Purpose | Command |
| ------- | ------- |
| Download `mongomirror` | `wget https://s3.amazonaws.com/mciuploads/mongomirror/binaries/linux/mongomirror-linux-x86_64-ubuntu1804-0.9.1.tgz` |
| Extract the tar | `tar xvzf mongomirror*` |
| Enter extracted directory | `cd mongomirror*/bin` |

* Once complete, we will run the mirror command as follows:

```
./mongomirror 
    --host rsMigration/localhost:27017 \
    --username root --password root123 \ 
    --destination MIGRATE-shard-0/migrate-shard-00-00-oadjb.gcp.mongodb.net:27017,migrate-shard-00-01-oadjb.gcp.mongodb.net:27017,migrate-shard-00-02-oadjb.gcp.mongodb.net:27017 \
    --destinationUsername admin --destinationPassword admin123 
    --drop
```
  