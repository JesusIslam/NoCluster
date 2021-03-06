# NoCluster
----------
A Node.js backend cluster web server with HAProxy front-end

```
            __________
Internet ---| HAProxy|---| front_server1         back_server1       ------------------------- 
            |   or   |   |               | === | back_server2| === | MongoDB Sharded Cluster |
            |_Nginx__|---| front_server2         back_servern       -------------------------
                                |                                         |
                                |____________session_manager______________|
```

Dependencies :

* [HAProxy](http://haproxy.1wt.eu) or [Nginx](http://nginx.org) or [LVS](www.linuxvirtualserver.org)
* [Axon](https://github.com/visionmedia/axon)
* [Express](http://www.expressjs.com)
* [Busboy](https://github.com/mscdex/busboy)
* [Swig](https://paularmstrong.github.io/swig/)
* [Mandrill-API](https://www.npmjs.org/package/mandrill-api)
* [Mongoskin](https://github.com/kissjs/node-mongoskin)
* [MongoDB](http://mongodb.org)
* [connect-mongo](https://github.com/kcbanner/connect-mongo)
* [express-session](https://github.com/expressjs/session)
* [cookie-parser](https://github.com/expressjs/cookie-parser)
* [morgan](https://github.com/expressjs/morgan)
* [node-uuid](https://github.com/broofa/node-uuid)

### Why Axon?

Because it is easy. You do not have to install message passing interface at all, the module done it for you. The obvious disadvantage is lower throughput and higher latency. Consider using [ZeroMQ](http://zeromq.org) for better performance.

### Installation

`apt-get install haproxy` or `apt-get install nginx`

Enable HAProxy to be started by your init script while nginx will automatically started

`nano /etc/default/haproxy`

Change this line

`ENABLED=1`

Install the Node.js dependencies

`npm install express axon busboy swig mandrill-api mongoskin connect-mongo express-session cookie-parser morgan gridfs-locking-stream --save`

Install mongodb

Follow [this](http://docs.mongodb.org/manual/installation) tutorial



### HAProxy Configurations
```
global
    log 127.0.0.1 local0 notice
    maxconn 8000 #2000 x 4 core
    user haproxy
    group haproxy
    
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    retries 3
    option redispatch
    timeout connect  5000
    timeout client  10000
    timeout server  10000
    
listen nocluster *:80
    mode    http
    stats   enable
    stats uri /haproxy?stats
    stats realm Strictly\ Private
    stats auth username:YourPassword
    option httpclose
    option forwardfor
    balance roundrobin
    cookie JSESSIONID prefix indirect nocache
    server SERV1 192.168.0.3:8000 check cookie SERV1
    server SERV2 192.168.0.4:8000 check cookie SERV2
    
# and the list goes on, note that these servers are the frontends
# the application logic lies on the backend behind axon
```

And then save it as `/etc/haproxy/haproxy.cfg`

### Nginx Configurations
```
upstream cluster {
    ip_hash;
    server 192.168.0.3:8000;
    server 192.168.0.4:8000;
    keepalive 64;
}
server {
    listen 80;
    server_name worksinmagic.com;
    access_log /var/log/nginx/worksinmagic.com-access.log;
    error_log /var/log/nginx/worksinmagic.com-error.log;
    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_pass http://cluster/;
        proxy_redirect off;
    }
}
```

And then save it as `/etc/nginx/sites-enabled/cluster`

### LVS Configurations

Install ipvsadm `sudo apt-get install ipvsadm`

Describe the topology
```
ipvsadm -A -t 127.0.0.1:80 -s sh
ipvsadm -a -t 127.0.0.1:80 -r 192.168.0.3:8000 -m
ipvsadm -a -t 127.0.0.1:80 -r 192.168.0.4:8000 -m
```

Now save it `ipvsadm -S -n > /etc/ipvsadm.conf`

Next, make the failover work using `mon` or build your own or use `monitor.js` on this repo but I will describe how to use `mon`.

First, install `sudo apt-get install mon`
Then edit the configuration file by adding these lines
```
### group definitions (hostnames or IP addresses)
hostgroup HTTP1 192.168.0.3
 
watch HTTP1
   service http
        interval 5s
        monitor http.monitor -p 8000
        allow_empty_group
        period wd {Sun-Sat}
            alert http.alert
            upalert http.alert
 
hostgroup HTTP2 192.168.0.4
 
watch HTTP2
   service http
        interval 5s
        monitor http.monitor -p 8000
        allow_empty_group
        period wd {Sun-Sat}
            alert http.alert
            upalert http.alert
```

We then add http.alert to `/usr/lib/mon/alert.d/http.alert`
```
#!/bin/sh
#
# $Id: test.alert,v 1.1.1.1 2004/06/09 05:18:07 trockij Exp $
# echo "`date` $*" >> /var/log/lvs.http.alert.log
 
if [ "$9" = "-u" ]
then
   echo "`date` Real Server $6 is UP" >> /var/log/lvs.http.alert.log
   ipvsadm -a -t 127.0.0.1:80 -r $6:8000 -m
else
   echo "`date` Real Server $6 is DOWN" >> /var/log/lvs.http.alert.log
   ipvsadm -d -t 127.0.0.1:80 -r $6:8000 
fi
```

Then we can finally start `mon` to monitor our servers.

### MongoDB (Optional)

##### Configurations if you want to build replica set

Set up your DNS first in `/etc/hosts`, don't forget to change the last word of first line
reflecting your current host

```
# adjust this accordingly
127.0.0.1   localhost mongo0

192.168.0.5 mongo0.worksinmagic.com
192.168.0.6 mongo1.worksinmagic.com
# and the list goes on
```

Issue this command and modify accordingly on each machine

`hostname mongo0.worksinmagic.com`

and then edit the `/etc/hostname`to reflect this
`mongo0.worksinmagic.com`

Now the DNS setting has been completed.

Stop all server from running

`service mongod stop`

Now we create the config file

```
# Remember to create the directory first
dbpath=/mongodb-database 
port=27017
logpath=/mongodb-database/mongodb.log
logappend=true
#auth=true
diaglog=1
nohttpinterface=true
nssize=64
# in master/slave replicated mongo databases, specify here whether
# this is a slave or master
#slave = true
#source = master.worksinmagic.com
# Slave only: specify a single database to replicate
#only = master.worksinmagic.com
# or
#master = true
#source = slave0.worksinmagic.com

# in replica set configuration, specify the name of the replica set
replSet=rs0
fork=true

```

Save and close the file

On one of your member (or master.worksinmagic.com), do

`mongo`

On the prompt, enter

`rs.initiate()`

This will initiate the replication set and add the server you are currently connected to as the first member of the set. 

Check by typing

`rs.conf()`

it should return something like this

```
{
    "_id" : "rs0"
    "version" : 1,
    "members" : [
        {
            "_id" : 0,
            "host" "mongo0.worksinmagic.com:27017"
        }
    ]
}
```

Now, you can add the additional nodes to the replication set by referencing the hostname you gave them in the `/etc/hosts` file:

````
rs.add("mongo1.worksinmagic.com")
```

And then you can restart the server `service mongod start`

##### Configurations if you want to use sharded cluster

You need a minimum of 6 machines as :

* ##### Config server :
Each production sharding implementation must contain exactly three configuration servers. This is to ensure redundancy and high availability. Config servers are used to store the metadata that links requested data with the shard that contains it. It organizes the data so that information can be retrieved reliably and consistently.

* ##### Query routers :
The query routers are the machines that your application actually connects to. These machines are responsible for communicating to the config servers to figure out where the requested data is stored. It then accesses and returns the data from the appropriate shard(s). Each query router runs the "mongos" command. The most common practice is to run mongos instances on the same systems as your application servers, but you can maintain mongos instances on the shards or on other dedicated resources.

* ##### Shard servers :
Shards are responsible for the actual data storage operations. In production environments, a single shard is usually composed of a replica set instead of a single machine. This is to ensure that data will still be accessible in the event that a primary shard server goes offline. Implementing replicating sets is outside of the scope of this tutorial, so we will configure our shards to be single machines instead of replica sets. You can easily modify this if you would like to configure replica sets for your own configuration.

With :

* 3 Config servers

* 1 Query router minimum

* 2 Shard servers minimum

In reality, some of these functions can overlap (for instance, you can run a query router on the same machine you use as a config server)


Use these settings and set it like you set the DNS of replica sets

```
config0.worksinmagic.com
config1.worksinmagic.com
config2.worksinmagic.com

query0.worksinmagic.com

mongo0.worksinmagic.com
mongo1.worksinmagic.com
```

Log in to config0 and create a directory `/mongodb-database`, and stop mongodb `service mongod stop`

And then run `mongod --configsvr --dbpath /mongodb-database --port 27019`

You can add that on upstart or init.d if you want, also remove the default upstart and init.d. Do that for each config server.

Now log in to query0 and stop mongodb `service mongod stop`, do not forget to turn it off in upstart or init.d because `mongod` will conflict with the router.

Ok, now run this command (do not press enter after `--configdb`, its a space):

```
mongos --configdb config0.worksinmagic.com:27019,config1.worksinmagic.com:27019,config2.worksinmagic.com:27019
```

Add that command to upstart if you want. Do this for every query server if you have more than one.

Then log in to your one of shard cluster and run `mongo --host query0.worksinmagic.com --port 27017`, do this for every query server you have.

And add your shards there

```
# if single instances
sh.addShard("mongo0.worksinmagic.com:27017")
sh.addShard("mongo1.worksinmagic.com:27017")

# if replica
sh.addShard("rs0/0:27017")
sh.addShard("rs0/1:27017")
```

Now back to query0 again, we will enable sharding. Connect to the query0 server `mongo --host query0.worksinmagic.com --port 27017`

Then enable sharding on database level
```
use sharded_db
sh.enableSharding("sharded_worksinmagic_db")
```

Then we shard on collection level. Be sure to choose `shard key` that will be evenly distributed or you can use hashed shard key based on existing field. Now we will do this on a new collection:
```
use sharded_worksinmagic_db
db.users.ensureIndex({ _id : "hashed" })
sh.shardCollection("sharded_worksinmagic_db.users", { "_id" : "hashed" })
db.articles.ensureIndex({ _id : "hashed" })
sh.shardCollection("sharded_worksinmagic_db.users", { "_id" : "hashed" })
```

To get information about specific shards you can type `sh.status()`

Which will return something like this:
```
--- Sharding Status --- 
  sharding version: {
    "_id" : 1,
    "version" : 3,
    "minCompatibleVersion" : 3,
    "currentVersion" : 4,
    "clusterId" : ObjectId("529cae0691365bef9308cd75")
}
  shards:
    {  "_id" : "shard0000",  "host" : "192.168.0.5:27017" }
    {  "_id" : "shard0001",  "host" : "192.168.0.6:27017" }
. . .
```

And then you have to make sure you connect to the query server to access a shard cluster.

Or really, if you don't expect high throughput or larger than RAM dataset (or don't mind losing all your data if something happened), using a single mongod server is enough.

### About Saving Uploaded Files
To save uploaded files, you have some choices:

* Place the uploaded file in a variable and send that to back server to be saved or processed or
* Save the uploaded file directly to the shared storage (NFS or something like that) or
* Use [Binary.js](http://binaryjs.com) and create a dedicated streaming server for file handling or
* Deploy a new front-end server dedicated for file handling

Personally, I would go with the second choice because it's the easiest and you need a shared storage anyway. But that would means more load on the front server and that's not good if you want a snappy response. 

Meanwhile, with the first choice we would make the back server busy and you wouldn't want that on a write heavy back end. 

The third choice probably be good for a single page web app because you can show users their file upload progress easily and the server would be more flexible. But this would mean we can only support browsers with websocket feature.

The fourth choice is similar to the third except not as flexible but support all browsers.

Please choose carefully.