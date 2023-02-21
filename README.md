How to: Docker Oracle 19.3 and ORDS 22.4.2 and APEX 22.2.0
-----------------------------------------------------------
1) Configure Disk (Optional)
Create virtual disk at Virtual Box level 
(more info https://www.tutorialspoint.com/how-to-add-disk-storage-to-oracle-virtual-box-on-linux#:~:text=Adding%20the%20Virtual%20Drive,a%20new%20hard%20disk%20drive.)


Shutdown Virtualbox
Settings -> Storage -> Controller: Sata Controller press Add hard disks
30GB
Attach it to Virtual box
Start Virtual Box.

Login as root

#ls /dev/sd*
/dev/sda /dev/sda1 /dev/sda2 /dev/sdb

/dev/sdb --> is new disk...

create partition (fdisk) / create filesystem / add it to /etc/fstab / reboot linux

# MOUNT_POINT=/var/lib/docker
# DISK_DEVICE=/dev/sdb

New partition for the whole disk.
# echo -e "n\np\n1\n\n\nw" | fdisk ${DISK_DEVICE}

Add file system.
# mkfs.xfs -f ${DISK_DEVICE}1

Mount it using the UUID of the VirtualBox virtual disk.
# rm -Rf /var/lib/docker
# mkdir /var/lib/docker
# UUID=`blkid -o export ${DISK_DEVICE}1 | grep UUID | grep -v PARTUUID`
# mkdir ${MOUNT_POINT}
# echo "${UUID}  ${MOUNT_POINT}    xfs    defaults 1 2" >> /etc/fstab
# mount ${MOUNT_POINT}

-----------------------------------------------------------
2) Docker : Install Docker on Oracle Linux 8 (OL8)
(more info https://oracle-base.com/articles/linux/docker-install-docker-on-oracle-linux-ol8)

Linux9'a kurarsan alınan hata:Docker - library initialization failed - unable to allocate file descriptor table - out of memory
Linux7'e kurarsan DNF yok, yum install dnf de çalışmıyor Error: Unable to find a match



# dnf install -y dnf-utils zip unzip
# dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

# dnf remove -y runc
# dnf install -y docker-ce --nobest


Enable and start the Docker service.

# systemctl enable docker.service
# systemctl start docker.service

You can get information about docker using the following commands.

# systemctl status docker.service
# docker info
# docker version

-----------------------------------------------------------

At this point you have to either pull oracle 19.3 db image from container-registry.oracle.com or build it.
Item 3 is about pulling the image from registery and starting it up.
Item 7 below is about building container image for 19.3 db. 
-----------------------------------------------------------
3) Pulling Oracle 19.3.0 Database Image from container-registry.oracle.com & Starting it up (it is alterative of item 3)

# docker pull container-registry.oracle.com/database/enterprise:19.3.0.0
Error response from daemon: pull access denied for container-registry.oracle.com/database/enterprise, repository does not exist or may require 'docker login': denied: requested access to the resource is denied

goto URL 
https://container-registry.oracle.com/ords/f?p=113:10::::::

login and accept the Term and conditons
use the same user to login to "docker login container-registry.oracle.com"

# docker login container-registry.oracle.com
Username:

# docker login container-registry.oracle.com
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store


# docker pull container-registry.oracle.com/database/enterprise:19.3.0.0
19.3.0.0: Pulling from database/enterprise
66fb34780033: Pull complete 
09fdde9a2705: Pull complete 
8eef51bdc524: Pull complete 
ef20be24fdb7: Pull complete 
3eec5dd39cb8: Pull complete 
f54e46803ca1: Pull complete 
bbf9b5dc6e4f: Pull complete 
acce8c5c4fc6: Pull complete 
4eed3ae7ef08: Pull complete 
ea799524d345: Pull complete 
dc921b3204ee: Pull complete 
ff21863176fa: Pull complete 
a3f0ecf1aafc: Pull complete 
58fb15665386: Pull complete 
630de49b8518: Pull complete 
94e9e3c3db5e: Pull complete 
c62e2b0d3050: Pull complete 

---------------------------
Tidy up the images -- you may skip this section

# docker images
REPOSITORY                                          TAG         IMAGE ID       CREATED        SIZE
oracle/jdk                                          17          b9f5a25fc208   22 hours ago   565MB
oracle/database                                     19.3.0-ee   a0311eaa3feb   42 hours ago   6.54GB
container-registry.oracle.com/database/ords         latest      13917e2320d4   8 weeks ago    2.11GB
container-registry.oracle.com/database/enterprise   19.3.0.0    26f422a3e69f   4 months ago   7.75GB <<<<<<-----new one

# docker rmi oracle/jdk 

# docker image tag container-registry.oracle.com/database/enterprise:19.3.0.0 container/database:19.3.0-eedocker

# docker images 
REPOSITORY                                          TAG         IMAGE ID       CREATED        SIZE
oracle/database                                     19.3.0-ee   a0311eaa3feb   44 hours ago   6.54GB
container-registry.oracle.com/database/ords         latest      13917e2320d4   8 weeks ago    2.11GB
container-registry.oracle.com/database/enterprise   19.3.0.0    26f422a3e69f   4 months ago   7.75GB
container/database                                  19.3.0-ee   26f422a3e69f   4 months ago   7.75GB

# docker rmi container-registry.oracle.com/database/enterprise:19.3.0.0
Untagged: container-registry.oracle.com/database/enterprise:19.3.0.0
Untagged: container-registry.oracle.com/database/enterprise@sha256:111e8b3c943bee7017cfd3465bd604aaa2966782b60dd2bff9abe7201a5808cc

-----------------------------------------------
Creating and Starting it up 

# docker images 
REPOSITORY                                    TAG         IMAGE ID       CREATED        SIZE
oracle/database                               19.3.0-ee   a0311eaa3feb   44 hours ago   6.54GB
container-registry.oracle.com/database/ords   latest      13917e2320d4   8 weeks ago    2.11GB
container/database                            19.3.0-ee   26f422a3e69f   4 months ago   7.75GB

It’s better to create a network in a docker environment so our dockers can communicate with other dockers using a hostname

# docker network create oracle_network
# docker network inspect oracle_network

if oracle user does not exist create oracle user

# groupadd -g 54322 dba
# groupadd -g 54321 oinstall
# useradd -u 54321 -g oinstall -G dba oracle


# rmdir -rf /opt/docker19/
# mkdir -p /opt/docker19/oradata
# chown -R oracle:oinstall /opt/docker19/

-------------------------
Start Docker Oracle Image

# docker run -it --rm --name oracledb19c --network=oracle_network -p 1521:1521 -p 5500:5500 -e ORACLE_PWD=Welcome1##  -v /opt/docker19/oradata:/opt/oracle/oradata  container/database:19.3.0.0-ee

-p 1521:1521 -p 5500:5500 exposes the container port 1521 and 5500 to the same ports on the host so they can be used outside the container.
-e ORACLE_PWD=Welcome1## creates the environment variable ORACLE_PWD inside the container, which will be used as the password of the SYS, SYSTEM, and PDB_ADMIN accounts. 
The variable is only used the first time the container is run.
-v /opt/docker19/oradata:/opt/oracle/oradata mounts the container directory /opt/oracle/oradata to /opt/docker19/oradata directory (of course, you can specify any directory). 
If you omit this instruction, the data will not be persisted and the database will be recreated every time the container starts, so be sure to use it and always point to the same directory.
--network=oracle_network  network in docker environment so our dockers can communicate with other dockers using a hostname "oracledb" (name of 19.3.0 db container)

The process of creating the database will take between 10 and 20 minutes, 
you just have to wait for the message DATABASE IS READY TO USE!:

-----------
Regular startup of DB containers

# docker run -it --rm --name oracledb --network=oracle_network -p 1521:1521 -p 5500:5500 -v /opt/docker19/oradata:/opt/oracle/oradata  container/database:19.3.0-ee

------------------
to change password of sys/system
# docker exec oracledb /opt/oracle/setPassword.sh Welcome1##

------------------
to login to docker 
# docker exec -it oracledb /bin/bash

# docker exec -it ords /bin/bash

########
#### At this point it is assumed that db is up and running 
########

-----------------------------------------------------------
4) Install oracle instant client to test that you can connect to database 19.3 running inside of docker container
(Oracle Linux 8)
As root

# dnf install oracle-instantclient-release-el8
# dnf install oracle-instantclient-basic
# dnf install oracle-instantclient-sqlplus

---------
--Additionally, you may have to perform the following tasks before you start your application:
--copy or modify tnsnames.ora, sqlnet.ora, ldap.ora, or oraaccess.xml with Oracle Instant Client, 
--/usr/lib/oracle/21/client64/lib/network/admin subdirectory.
--Or set the environment variable TNS_ADMIN to directory that contain files.
-------

# export PATH=/usr/lib/oracle/21/client64/bin:$PATH.  >> (instant client pathini export ediyor böylece heryerden sqlplus yapılabilir)


# cd /usr/lib/oracle/21/client64/lib/network/admin
vi tnsnames.ora
add following lines and wq!

# cat tnsnames.ora 
ORCLCDB=localhost:1521/ORCLCDB
ORCLPDB1= 
(DESCRIPTION = 
  (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = ORCLPDB1)
  )
)

Test local to docker connection!!!

# sqlplus system/Welcome1##@//localhost:1521/ORCLPDB1
exit

# sqlplus system/Welcome1##@ORCLPDB1
exit

----------------
At this connection following error is given...
( more info https://franckpachot.medium.com/19c-instant-client-and-docker-1566630ab20e )

sqlplus system/Welcome1##@//localhost:1521/ORCLPDB1

SQL*Plus: Release 21.0.0.0.0 - Production on Fri Feb 3 19:33:41 2023
Version 21.9.0.0.0

Copyright (c) 1982, 2022, Oracle.  All rights reserved.

ERROR:
ORA-12637: Packet receive failed

---
to disbale OOB

docker exec -t oracledb bash -ic 'echo DISABLE_OOB=ON > $ORACLE_HOME/network/admin/sqlnet.ora'

Then the connection is ok.

At this point DB is up and running and listenening and you can connect it using sqlplus

-----------------------------------------------------------
5) Continue with ORDS and APEX installation
(altough it is for Mac it is very helpful -> https://luca-bindi.medium.com/oracle-xe-and-apex-in-a-docker-container-25f00a2b8306)

open following link in a browser
https://container-registry.oracle.com/ords/f?p=113:10::::::

click on Database -> to goto Database Repository -> click on ords --> Orgo to below URL

(https://container-registry.oracle.com/ords/f?p=113:4:5003129767841:::4:P4_REPOSITORY,AI_REPOSITORY,AI_REPOSITORY_NAME,P4_REPOSITORY_NAME,P4_EULA_ID,P4_BUSINESS_AREA_ID:1183,1183,Oracle%20REST%20Data%20Services%20(ORDS)%20with%20Application%20Express,Oracle%20REST%20Data%20Services%20(ORDS)%20with%20Application%20Express,1,0&cs=3B3LrvS0RouFJgCIichl3-LNVIIpsPfD6v0e3DLbhwOrKRaAitxdzdLpg6-6e4_F1zajZ5Ye4N9-XQB5-aIuVjg
) 

follow the steps in the page..
Pull ords image from oracle repository

# docker pull container-registry.oracle.com/database/ords:latest
it takes time to complete.....

# docker images
REPOSITORY                                    TAG         IMAGE ID       CREATED             SIZE
oracle/database                               19.3.0-ee   a0311eaa3feb   22 hours ago        6.54GB
container-registry.oracle.com/database/ords   latest      13917e2320d4   8 weeks ago         2.11GB

Preliminary work needed before runnig ords for the first time

# mkdir -p /opt/ords/ords_secrets /opt/ords/ords_config

# docker inspect oracledb | grep IPAddress
172.18.0.2

# echo CONN_STRING=sys/Welcome1##@172.18.0.2:1521/ORCLPDB1 > /opt/ords/ords_secrets/conn_string.txt

# cat opt/ords/ords_secrets/conn_string.txt

CONN_STRING=sys/Welcome1##@172.18.0.2:1521/ORCLPDB1

# docker run --rm --name ords --network=oracle_network -v /opt/ords/ords_secrets:/opt/oracle/variables -v /opt/ords/ords_config/:/etc/ords/config/ -p 8181:8181 container-registry.oracle.com/database/ords:latest


(Apex Objeleri hem PDB hem CDB de varsa alırsın biz DB uçurduk öyle çözdük)

The container will install/upgrade APEX and ORDS and before start the ORDS service

Open browser on localhost with the mapped port by docker (http://localhost:8181/ords) and use below credentials:

- Workspace: internal
- User:      ADMIN
- Password:  Welcome_1




----------
Mapped local pools from /etc/ords/config/databases:
  /ords/                              => default                        => VALID     
2023-02-04T14:00:17.269Z INFO        Oracle REST Data Services initialized
Oracle REST Data Services version : 22.4.0.r3401044
Oracle REST Data Services server info: jetty/10.0.12
Oracle REST Data Services java info: Java HotSpot(TM) 64-Bit Server VM 11.0.15+8-LTS-149
-----------

http://192.168.56.10:8181/ords/

docker stop ords
docker ps


docker stop ords

docker stop oracledb

-----------------------------------------------------------
6) Install Java 17 as docker image - Optional

# cd /root/docker-images/OracleJava
# cd 17

# docker build --tag oracle/jdk:17 .

# docker images
REPOSITORY                                    TAG         IMAGE ID       CREATED             SIZE
oracle/jdk                                    17          b9f5a25fc208   About an hour ago   565MB
oracle/database                               19.3.0-ee   a0311eaa3feb   22 hours ago        6.54GB
container-registry.oracle.com/database/ords   latest      13917e2320d4   8 weeks ago         2.11GB

-----------------------------------------------------------
Other links:

https://oracle-base.com/articles/linux/docker-install-docker-on-oracle-linux-ol8
https://eherrera.net/using-oracle-docker-container/
https://github.com/oracle/docker-images/blob/main/OracleDatabase/SingleInstance/README.md
https://tm-apex.blogspot.com/2022/06/running-apex-in-docker-container.html
https://www.talkapex.com/2017/10/docker-oracle-and-apex/
-----------------------------------------------------------

7) Building Oracle 19.3.0 Database Image for Docker Container & Starting it up
(more info https://eherrera.net/using-oracle-docker-container/)

login as root

# git clone https://github.com/oracle/docker-images

# cd /root/docker-images/OracleDatabase/SingleInstance/dockerfiles
# ls
11.2.0.2  12.1.0.2  12.2.0.1  18.3.0  18.4.0  19.3.0  21.3.0  buildContainerImage.sh

Download Oracle Database EE 19.3.0 from  https://www.oracle.com/database/technologies/oracle-database-software-downloads.html
transfer it to Virtualbox

# cp /tmp/LINUX.X64_193000_db_home.zip 19.3.0
# ./buildContainerImage.sh -v 19.3.0 -ee 

it takes time to complete

# docker images
REPOSITORY        TAG         IMAGE ID       CREATED         SIZE
oracle/database   19.3.0-ee   a0311eaa3feb   2 minutes ago   6.54GB

It’s better to create a network in a docker environment so our dockers can communicate with other dockers using a hostname

# docker network create oracle_network
# docker network inspect oracle_network

if oracle user does not exist create oracle user

# groupadd -g 54322 dba
# groupadd -g 54321 oinstall
# useradd -u 54321 -g oinstall -G dba oracle

# mkdir -p /opt/docker19/oradata
# chown -R oracle:oinstall /opt/docker19/

-------------------------
Start Docker Oracle Image

# docker run -it --rm --name oracledb --network=oracle_network -p 1521:1521 -p 5500:5500 -e ORACLE_PWD=Welcome1##  -v /opt/docker19/oradata:/opt/oracle/oradata  oracle/database:19.3.0-ee

-p 1521:1521 -p 5500:5500 exposes the container port 1521 and 5500 to the same ports on the host so they can be used outside the container.
-e ORACLE_PWD=Welcome1## creates the environment variable ORACLE_PWD inside the container, which will be used as the password of the SYS, SYSTEM, and PDB_ADMIN accounts. The variable is only used the first time the container is run.
-v /opt/docker19/oradata:/opt/oracle/oradata mounts the container directory /opt/oracle/oradata to /opt/docker19/oradata directory (of course, you can specify any directory). If you omit this instruction, the data will not be persisted and the database will be recreated every time the container starts, so be sure to use it and always point to the same directory.
--network=oracle_network  network in docker environment so our dockers can communicate with other dockers using a hostnmae "oracledb" (name of 19.3.0 db container)

The process of creating the database will take between 10 and 20 minutes, 
you just have to wait for the message DATABASE IS READY TO USE!:

------------------
Regular startup of DB containers

# docker run -it --rm --name oracledb --network=oracle_network -p 1521:1521 -p 5500:5500 -v /opt/docker19/oradata:/opt/oracle/oradata  oracle/database:19.3.0-ee

------------------
to change password of sys/system
# docker exec oracledb /opt/oracle/setPassword.sh Welcome1##

------------------
to login to docker 
# docker exec -it oracledb /bin/bash

# docker exec -it ords /bin/bash

########
#### At this point it is assumed that db is up and running 
########

Continue with item 4
----------------------



