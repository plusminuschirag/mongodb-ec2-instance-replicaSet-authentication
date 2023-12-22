# Practical World MongoDB : Setting up Database Replica, Internal and External Authentication.

# General Introduction

## Why MongoDB?

- DB Engine has ranked MongoDB consistently as the top Document based Database Model. The process itself in ranking is not a small feat:
  - It takes Google Trends, Job Posting, Stack Overflow Discussion and General Total Name Appearance over a subset of web in account.
  - Be it MERN, MEVN or MEAN stack there’s an M for MongoDB in all of them. Your webapps generally prefer MongoDB too. This begs the questions but why?
  - Good Question, MongoDB ships with features that are on the spot,
    - You are new to DBs? MongoDB’s entry threshold is very slow, just simply run command that insert json(also csv) like structure directly in it.
    - You are experience but not sure how DB’s structure will change over the time? MongoDB allows you to be schema-less if you want and you can keep changing the definition of fields in schema as time passes.
    - You are coming from SQL world? Will it work? Answer is a light yes, yes in sense you can be sure that with tight schema definition you can get feel of SQL like operations in MongoDB, you can get consistency of PostgreSQL by using transactions with appropriate read and write concerns.
    - MongoDB should be used in a way that leverages its strengths, and transactions should be employed judiciously where strong consistency is a requirement.
    - All in all, it’s a good DB tech to try on. It really solves a lot of your DB related problems very quickly and understand what project generally demand from a tech stack.

## Who is this for?

- If you are totally new to MongoDB, it’s easy to follow this one, all you need is an AWS instance preferably **t3.medium** rolling up for some hours to follow this hands-on.
- If you really aren’t able to grasp the problems directly by seeing the heading of the article it would be better to first go off this article understand the problems faced by DBs in production, and why replication works and how common and necessary it is to do it.
- But once you get the idea you can follow through this guide without a hitch.

## What are we gonna do?

- This whole setup is about getting MongoDB up and running on a vanilla ec2 instance:
  - Setting up EC2 Instance
  - Downloading and Installing MongoDB on ubuntu
  - Bringing the vanilla installation down
  - Explain Database Replication
  - Running Custom MongoDB with replicas.
  - Connecting MongoDB replicas to MongoDB Compass
  - Verifying if our write was successful to all the instances or not

# Setting up EC2 Instance

- You can setup a vanilla EC2 Instance using Ubuntu 22.04

  - Select the t3.medium
  - Allow HTTP and HTTPS traffic on the server : This can be removed?

- SSH into your EC2 Instance
- Run `sudo apt update`
  - Why do we run this command?

# Downloading and Installing MongoDB for Ubuntu

- Follow this official guide for it : [https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/)
- It’s fairly easy and straight forward.

## Verifying the Installation

- Way-1

  - Run `sudo systemctl status mongod`

  ```python
  mongod.service - MongoDB Database Server
       Loaded: loaded (/lib/systemd/system/mongod.service; disabled; vendor preset: enabled)
       Active: active (running) since Wed 2023-12-20 21:14:52 UTC; 6s ago
         Docs: https://docs.mongodb.org/manual
     Main PID: 36881 (mongod)
       Memory: 169.7M
          CPU: 1.490s
       CGroup: /system.slice/mongod.service
               └─36881 /usr/bin/mongod --config /etc/mongod.conf

  Dec 20 21:14:52 ip-172-31-84-245 systemd[1]: Started MongoDB Database Server.
  Dec 20 21:14:52 ip-172-31-84-245 mongod[36881]: {"t":{"$date":"2023-12-20T21:14:52.794Z"},"s":"I",  "c":"CONTROL",  "id":7484500, "ctx":"mai>
  ```

  - You will see output like the above one, some values will differ but this will be general outline for your mongod service too.

- Way - 2

  - Run `mongosh --port 27017`

  ```python
  Current Mongosh Log ID: 658359d05a6dc1eaa85c4c51
  Connecting to:          mongodb://127.0.0.1:27016/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.1.1
  Using MongoDB:          7.0.4
  Using Mongosh:          2.1.1

  For mongosh info see: https://docs.mongodb.com/mongodb-shell/

  ------
     The server generated these startup warnings when booting
     2023-12-20T21:14:52.823+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
     2023-12-20T21:14:54.227+00:00: Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
     2023-12-20T21:14:54.227+00:00: vm.max_map_count is too low
  ------

  test>
  ```

  - You will connect into mongo shell and land on the test db. This ensures that you have correctly installed the MongoDB and are able to access it.

## Bringing the Vanilla Installation Down

- Before we jump into anything, it’s a great time now to understand some concepts from MongoDB. We need to be familiar with `mongod` to run this whole guide.

### What’s is `mongod`?

- It stands for MongoDB daemon. It’s the main daemon process for the MongoDB database and serves as the primary component of the MongoDB system.
- If you are interested in understanding the internal architecture of the `MongDB` you can refer here.
- So we have to keep one thing in mind, whatever we wanna do to our `MongoDB` we do it via `mongod` which in turns reads a .conf file to understand what we want to do with your MongoDB instance. For the sake of ease, it’s also a systemctl service on Ubuntu, making it directly accessible from anywhere. This comes very handy, seriously!!

### Let’s take a look into our mongo.conf

- Run `cat /etc/mongod.conf` and you will receive the following output:

  ```python
  # mongod.conf

  # for documentation of all options, see:
  #   http://docs.mongodb.org/manual/reference/configuration-options/

  # Where and how to store data.
  storage:
    dbPath: /var/lib/mongodb
  #  engine:
  #  wiredTiger:

  # where to write logging data.
  systemLog:
    destination: file
    logAppend: true
    path: /var/log/mongodb/mongod.log

  # network interfaces
  net:
    port: 27017
    bindIp: 127.0.0.1

  # how the process runs
  processManagement:
    timeZoneInfo: /usr/share/zoneinfo

  #security:

  #operationProfiling:

  #replication:

  #sharding:

  ## Enterprise-Only Options:

  #auditLog:
  ```

- You are aware of the concepts in the database, you will see very common names like replication, sharding, security and network interfaces in this file.
- What’s happening in each block?
  - `storage` : This is related to our MongoDB data, everything we save in our database will be present in the `dbPath` mentioned under the `storage`.
  - `systemLog` : This is related to the logs generated by the MongoDB instance itself.
  - `net`: This basically deals with our network interfaces, what port and host our MongoDB uses.
    - We would be changing our default `port` to `27016` because we will be running `27017` for our MongoDB replica cluster. It will be a good practice to turn this one off and bring other up from scratch.
    - Current IP is `127.0.0.1` which is [localhost](http://localhost) and not accessible from outside the server, to make our MongoDB accessible from outside, from our PC or laptop to connect via Mongo Compass, we will be changing this to `0.0.0.0` for making it visible to outside world.

### Bringing the Installation Down

- Run `sudo systemctl stop mongod`
  - This will stop mongod service
- Run `sudo nano /etc/mongo.conf` This will open our `mongo.conf` file in editor mode.
  - Change the `port` under `net` from 27017 to 27016
  - Save and exit by `CTRL + S` and `CTRL + X` respectively.
- Run `sudo systemctl daemon-reload`
- Run `sudo systemctl start mongod`
- Run `sudo systemctl status mongod`

  - This command will give you the status just the like the one we got above on the default port.

- Now `port 27017` is clear from the MongoDB instances.

# A Refresher on Database Replication

### Understanding Replication

- Replication is creating exact copy of something. If it’s an exact copy it can be used just like the original anywhere original could’ve been used.
- In context of databases, replication is a way of creating exact copy of database somewhere, it can be on same server(won’t be helpful much), can be on a different one.
  - We often associate data redundancy and high availability with the replication.
  - **Data Redundancy** : As we can guess by the name, redundancy means copies it means having data in more than one place. Replication is a database level strategy to implement that.
  - **High Availability**: Highly Available systems are the one which have high uptime, meaning they are not down. So we can imagine a simple scenario in our head, assume that we have a database and we have a copy for it. Both of them are aware whether another one is down or not. Suppose we make request to our database A or database B and the one we make request to is currently down or unreachable:
    - if database A and B are not connected then the system sees downtime, which is obvious
    - if database A and B are connected, even if one of them is down other can take it’s place and keep the system functioning until the down instances comes back up.
    - if we can keep your system running in some point of failures, it means our system is highly available.
- There are more options like disaster recovery and backup that are also part of replication but they are for some other time. Once we get the setup, these will be only configurational parts.

### Replication in context of MongoDB

![Untitled](imgs/0.png)

- To implement replication in MongoDB you need three nodes(MongoDB Instances running on port) where one will become primary and remaining two are secondary.
  - **Primary :** This is the elected leaders among the nodes in the replica set.
    - Write operations in MongoDB go through this member only. This ensures a consistent and centralized point for all.
    - It has an oplog(operation log) which stores all the record modification in the db. This log is used to replicate changes to secondary nodes.
    - These are many more reasons why MongoDB has this strategy in place, you can read them up in official documentation.
  - **Secondary**: These are the rest of the nodes in the replica set. One of them becomes primary, in case primary goes down, these secondary have voting strategy in place to elect a new primary.
- MongoDB implements replication using replica set. Writer operations are sent to the primary server (node), which applies the operations across secondary servers, replicating the data.
- If the primary server fails (through a crash or system failure), one of the secondary servers takes over and becomes the new primary node via election. If that server comes back online, it becomes a secondary once it fully recovers, aiding the new primary node.
- When you are reading and writing from replicas, we need to define what we considered as a correct read or write operation:
  - You can take one more example for writing, do we consider a write operation complete if it’s been done by 1 node or some node or all of the nodes? It depends on the level of consistency we want from our system.
  - For MongoDB, “majority” is the default write concern where it’s calculated on data voting members(we can have nodes that cannot vote for any decision but be a part of replica set)
  - Similarly for read preferences we have following modes in hand:
    - **primary** : Default mode, must read from the primary node only.
    - **primaryPreferred** : Read from primary but in case it’s not available ready from the secondary members.
    - **secondary** : Read from the secondary only
    - **secondaryPreferred** : Read from the secondary but if primary is available read from it.
    - nearest : Read from the server with lowest latency threshold.

## Setting up Replication In MongoDB

- Create directory and sub-directory for storing db, and it’s log
  - Run : `cd ~`
  - Run `mkdir -p db_replicas/db1/log`
  - Run `mkdir -p db_replicas/db2/log`
  - Run `mkdir -p db_replicas/db3/log`
  - This will create directory where we will store MongoDB instance and their corresponding logs.
    - `db_replicas/db` will have the database.
    - `db_replicas/db<x>/log` will have the logs for that corresponding database.
- Set permission such that MongoDB is able to operate in this directory:
  - Run `chown ubuntu:ubuntu db_replicas/`
  - Here `ubuntu` is my username and usergroup, you can create a specific user for the MongoDB and then assign these ownership permission to that user.
- Create a `keyFile` for internal authentication of MongoDB instances in a replica set.
  - Run `openssl rand -base64 756 > db_replicas/keyFileInternalDBs`
    - This is just a random key, it can be a normal string too.
  - Run `chmod 400 db_replicas/keyFileInternalDBs`
  - Run `chown ubuntu:ubuntu db_replica/keyFileInternalDBs`
- Next Step is to create configuration files for each MongoDB instance and in all of the three directories and then bring the whole replica set to life.

### Creating a mongo.conf File

- Run `nano db_replicas/db1/mongod1.conf`
  - This will open your conf file in editor mode.
- Paste the following content in the file:

```makefile
systemLog:
  destination: file
  path: /home/ubuntu/db_replicas/db1/log/mongodb.log
  logAppend: true
storage:
  dbPath: /home/ubuntu/db_replicas/db1
net:
  port: 27017
  bindIp: 0.0.0.0
replication:
  replSetName: rs0
processManagement:
  fork: true
security:
  authorization: enabled
  keyFile: /home/ubuntu/db_replicas/keyFileInternalDBs
```

- Save and exit using `CTRL + S` and `CTRL + X` respectively.

- Similarly Run `nano db_replicas/db2/mongod2.conf` and paste the following and save and then exit.

```makefile
systemLog:
  destination: file
  path: /home/ubuntu/db_replicas/db2/log/mongodb.log
  logAppend: true
storage:
  dbPath: /home/ubuntu/db_replicas/db2
net:
  port: 27018
  bindIp: 0.0.0.0
replication:
  replSetName: rs0
processManagement:
  fork: true
security:
  authorization: enabled
  keyFile: /home/ubuntu/db_replicas/keyFileInternalDBs
```

- Similarly Run `nano db_replicas/db3/mongod3.conf` and paste the following and save and then exit.

```makefile
systemLog:
  destination: file
  path: /home/ubuntu/db_replicas/db3/log/mongodb.log
  logAppend: true
storage:
  dbPath: /home/ubuntu/db_replicas/db3
net:
  port: 27019
  bindIp: 0.0.0.0
replication:
  replSetName: rs0
processManagement:
  fork: true
security:
  authorization: enabled
  keyFile: /home/ubuntu/db_replicas/keyFileInternalDBs
```

- We have configuration set for our 3 replicas:

  - Each replica runs on a different port, is configured to different directory.
  - Each replica is part of same set `replSetName` and has same security `keyFile`

- Before moving on verify each and every config file again.

## Bringing Instances to Life

- Run `mongod --config db_scripts/db1/mongod1.conf`
- Run `mongod --config db_scripts/db2/mongod2.conf`
- Run `mongod --config db_scripts/db3/mongod3.conf`
- Run `pgrep mongod` this will show you 4 PIDs that are running MongoDB instance on their own.

  - You will see the output like following:

  ```makefile
  pgrep mongod

  32280
  32406
  32555
  36881
  ```

  - You can get the PID details with the following command

  ```makefile
  ➜  ~ ps -f -p $(pgrep mongod)
  UID          PID    PPID  C STIME TTY      STAT   TIME CMD
  ubuntu     32280       1  1 Dec20 ?        SLl   12:45 mongod --config db_replicas/db1/mongod1.conf
  ubuntu     32406       1  1 Dec20 ?        SLl   11:50 mongod --config db_replicas/db2/mongod2.conf
  ubuntu     32555       1  1 Dec20 ?        SLl   11:50 mongod --config db_replicas/db3/mongod3.conf
  mongodb    36881       1  0 Dec20 ?        Ssl    6:09 /usr/bin/mongod --config /etc/mongod.conf
  ```

  - If you are wondering why are we seeing 4 PIDs instead of 3 that’s because one of the PID belongs to our default MongoDB that’s now running at port 27016.

- Login into your Primary Instance.
  - Currently we don’t have authentication enabled on our replica set, so logging in is a cake walk but beware without proper authentication don’t even think about using the db outside local host, you will be hacked in hours.
  - Run `mongosh --port 27017`
    - You should see `rs0 [direct: primary]` right at the left corner of your cursor in terminal if you see `rs0 [direct: secondary]` log out by typing exit and log in to another instance of replica set `mongosh --port 27018` or `mongosh --port 27019`
    - Once you have found the primary instance, stay logged in.
- Initialize the MongoDB Instance
  - Copy, paste and Run the command below to initialize our replica set.
  ```makefile
  rs.initiate({
    _id: "rs0",
    members: [
      { _id: 0, host: "hostname:27017" },
      { _id: 1, host: "hostname:27018" },
      { _id: 2, host: "hostname:27019" }
    ]
  })
  ```
  - You will get replica set initialized successfully message on your terminal.
- Check the Replica Set Status
  - Run `rs.status()`
  - You will output like the following, members keys points to the members of your replica set, all three instance, one of them acting as primary and others acting as secondary.
  ```makefile
  {
    set: 'rs0',
    date: ISODate('2023-12-21T14:47:22.691Z'),
    myState: 1,
    term: Long('14'),
    syncSourceHost: '',
    syncSourceId: -1,
    heartbeatIntervalMillis: Long('2000'),
    majorityVoteCount: 2,
    writeMajorityCount: 2,
    votingMembersCount: 3,
    writableVotingMembersCount: 3,
    optimes: {
      lastCommittedOpTime: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
      lastCommittedWallTime: ISODate('2023-12-21T14:47:20.576Z'),
      readConcernMajorityOpTime: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
      appliedOpTime: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
      durableOpTime: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
      lastAppliedWallTime: ISODate('2023-12-21T14:47:20.576Z'),
      lastDurableWallTime: ISODate('2023-12-21T14:47:20.576Z')
    },
    lastStableRecoveryTimestamp: Timestamp({ t: 1703170000, i: 1 }),
    electionCandidateMetrics: {
      lastElectionReason: 'stepUpRequestSkipDryRun',
      lastElectionDate: ISODate('2023-12-20T20:04:28.401Z'),
      electionTerm: Long('14'),
      lastCommittedOpTimeAtElection: { ts: Timestamp({ t: 1703102660, i: 1 }), t: Long('13') },
      lastSeenOpTimeAtElection: { ts: Timestamp({ t: 1703102660, i: 1 }), t: Long('13') },
      numVotesNeeded: 2,
      priorityAtElection: 1,
      electionTimeoutMillis: Long('10000'),
      priorPrimaryMemberId: 2,
      numCatchUpOps: Long('0'),
      newTermStartDate: ISODate('2023-12-20T20:04:28.423Z'),
      wMajorityWriteAvailabilityDate: ISODate('2023-12-20T20:04:28.455Z')
    },
    electionParticipantMetrics: {
      votedForCandidate: true,
      electionTerm: Long('13'),
      lastVoteDate: ISODate('2023-12-20T20:04:10.380Z'),
      electionCandidateMemberId: 2,
      voteReason: '',
      lastAppliedOpTimeAtElection: { ts: Timestamp({ t: 1703102644, i: 1 }), t: Long('12') },
      maxAppliedOpTimeInSet: { ts: Timestamp({ t: 1703102644, i: 1 }), t: Long('12') },
      priorityAtElection: 1
    },
    members: [
      {
        _id: 0,
        name: 'localhost:27017',
        health: 1,
        state: 1,
        stateStr: 'PRIMARY',
        uptime: 69970,
        optime: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
        optimeDate: ISODate('2023-12-21T14:47:20.000Z'),
        lastAppliedWallTime: ISODate('2023-12-21T14:47:20.576Z'),
        lastDurableWallTime: ISODate('2023-12-21T14:47:20.576Z'),
        syncSourceHost: '',
        syncSourceId: -1,
        infoMessage: '',
        electionTime: Timestamp({ t: 1703102668, i: 1 }),
        electionDate: ISODate('2023-12-20T20:04:28.000Z'),
        configVersion: 1,
        configTerm: 14,
        self: true,
        lastHeartbeatMessage: ''
      },
      {
        _id: 1,
        name: 'localhost:27018',
        health: 1,
        state: 2,
        stateStr: 'SECONDARY',
        uptime: 69960,
        optime: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
        optimeDurable: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
        optimeDate: ISODate('2023-12-21T14:47:20.000Z'),
        optimeDurableDate: ISODate('2023-12-21T14:47:20.000Z'),
        lastAppliedWallTime: ISODate('2023-12-21T14:47:20.576Z'),
        lastDurableWallTime: ISODate('2023-12-21T14:47:20.576Z'),
        lastHeartbeat: ISODate('2023-12-21T14:47:21.978Z'),
        lastHeartbeatRecv: ISODate('2023-12-21T14:47:21.286Z'),
        pingMs: Long('0'),
        lastHeartbeatMessage: '',
        syncSourceHost: 'localhost:27019',
        syncSourceId: 2,
        infoMessage: '',
        configVersion: 1,
        configTerm: 14
      },
      {
        _id: 2,
        name: 'localhost:27019',
        health: 1,
        state: 2,
        stateStr: 'SECONDARY',
        uptime: 69953,
        optime: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
        optimeDurable: { ts: Timestamp({ t: 1703170040, i: 1 }), t: Long('14') },
        optimeDate: ISODate('2023-12-21T14:47:20.000Z'),
        optimeDurableDate: ISODate('2023-12-21T14:47:20.000Z'),
        lastAppliedWallTime: ISODate('2023-12-21T14:47:20.576Z'),
        lastDurableWallTime: ISODate('2023-12-21T14:47:20.576Z'),
        lastHeartbeat: ISODate('2023-12-21T14:47:22.222Z'),
        lastHeartbeatRecv: ISODate('2023-12-21T14:47:22.394Z'),
        pingMs: Long('0'),
        lastHeartbeatMessage: '',
        syncSourceHost: 'localhost:27017',
        syncSourceId: 0,
        infoMessage: '',
        configVersion: 1,
        configTerm: 14
      }
    ],
    ok: 1,
    '$clusterTime': {
      clusterTime: Timestamp({ t: 1703170040, i: 1 }),
      signature: {
        hash: Binary.createFromBase64('ot8/QjoD/crlJJ32RmxkZd6Yop4=', 0),
        keyId: Long('7314411506467078150')
      }
    },
    operationTime: Timestamp({ t: 1703170040, i: 1 })
  }
  ```
  - Your replica set is good to go.
- All we need to do now is to set External Authentication and open ports of our EC2 instance so we can use this MongoDB outside of our localhost.

## Adding External Authentication to our MongoDB Replica Set

- Currently we can login to our database without username and password and it will just directly fly by. This is a huge security risk, one that cannot be tolerated in production environment, in fact you shouldn’t do this if there is external access to the instance.
- `mongosh --port 27017` : Assuming your primary instance is at 27017, in face setting up a new user is possible even via secondary instance of replica set but just for the sake of follow through log in to your primary MongoDB instance.
- Once Logged In,
  - Run `use admin`
    - Move to admin db to save our new user with new roles.
  - Run the following command in your terminal, this gives admin level access to our database, users, read and write and cluster(whole replica set) to a user named ubuntu with password.
    ```makefile
    db.createUser({
      user: "ubuntu",
      pwd: "ubuntu",
      roles: [{ role: 'dbAdminAnyDatabase', db: 'admin' },
        { role: 'userAdminAnyDatabase', db: 'admin' },
        { role: 'clusterAdmin', db: 'admin' },
        { role: 'readWriteAnyDatabase', db: 'admin' }]
    })
    ```
  - Now you have a user named `ubuntu` that has access required to make changes to the database. These roles are an overkill and only for the purpose of this guide, in real practice you have to pay close attention to the privileges of each user for db, so that there is not an irresponsive role management.
- Log Out and Log In again using `mongosh --port 27017`
  - You might be wondering why you can still login without username or password.
  - You haven’t, you have just entered the DB, you won’t be able to access anything now.
  - Run `use admin` and then run `db.getUsers()` to get list of your users. You will run into the following error:
    ```makefile
    rs0 [direct: primary] admin> db.getUsers()
    MongoServerError: Command usersInfo requires authentication
    ```
- Now to login with ability to change the database, we will have to run the following command `mongosh --username ubuntu --password ubuntu --authenticationDatabase admin --port 27017`
  - You will be able to run the above command get details of all the users without running into authentication error.
- We can now log-out of our database and we are ready to configure AWS and see our MongoDB Replica Set in Action.

## Setting up Networking of our EC2 Instance.

- Go to your dashboard, and click on the `Instance ID` of our instance.
- Click on the Security Tab in the bottom panel and click on the URL like Security Group Name under `Security groups`
- Click `Edit Inbound Rules`
- Add three Custom TCP rules for port `27017` , `27018` and `27019` as shown in the image below:
  ![Untitled](imgs/1.png)

- Go back to Instance Details Page and Copy your Public DNS
  - In my case it’s [`ec2-54-85-249-37.compute-1.amazonaws.com`](http://ec2-54-85-249-37.compute-1.amazonaws.com/) it will be different in your case. Beware this DNS value changes every time you stop or restart your server, please make sure you do the next steps by verifying your latest public DNS.
    ![Untitled](imgs/2.png)

## Running from your PC

- Go and install Mongo Compass in your system.
- Open Mongo Compass and enter the following URI and click Connect
  - `mongodb://ubuntu:ubuntu@ec2-54-85-249-37.compute-1.amazonaws.com:27017,[ec2-54-85-249-37.compute-1.amazonaws.com:27018](http://ec2-54-85-249-37.compute-1.amazonaws.com:27018/),[ec2-54-85-249-37.compute-1.amazonaws.com:27019/?replicaSet=rs0](http://ec2-54-85-249-37.compute-1.amazonaws.com:27019/?replicaSet=rs0)`
  - This URI is constructed in the following pattern mongodb://<username>:<password>@<host1-ip>:<host1-port>,<host2-ip>:<host2-port>,<host3-ip>:<host3-port>
  - This is why we need to open these 3 ports for our EC2 Instance. If I had explain it before it might not have made sense, right?
- If all goes well you will be logged in to your MongoDB cluster.
- That’s all we wanted to do right now?
