WebLogic 12c in Docker POC Guide
===================
We have 2 server installed Oracle Linux 7 update 3 <br>
node1: 10.1.107.1 <br>
node2: 10.1.107.2 <br>
Configure bond0 use 2 network. <br>

#Docker deploy

1 Install Docker on Oracle Linux 7u3

Configure local yum repo(you can use "http://public-yum.oracle.com" and enable addon channel)
```
# cat /etc/yum.repos.d/
docker.repo  ol7.repo
[root@node1 ~]# cat /etc/yum.repos.d/docker.repo
[docker]
name=docker 1.12
baseurl=file:///media/docker
gpgcheck=0
enabled=1
[root@node1 ~]# cat /etc/yum.repos.d/ol7.repo
[ol7]
name=oracle linux 7 update 3
baseurl=file:///media/ol7
gpgcheck=0
enabled=1
```

Install Docker and Start Docker engine service
```
# yum install docker-engine
# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
# systemctl start docker
# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.12.2
Storage Driver: devicemapper
 Pool Name: docker-251:0-134415588-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 Data file: /dev/loop0
 Metadata file: /dev/loop1
…
```

2 setup devicemapper use direct-lvm for performance and production environment </br>
The Devicemapper default use loop device, but loop is not supported in production environment, so we change to direct-lvm.

prepare thin-provision lvm pool</br>
Create PV and VG named docker
```
[root@node2 ~]# fdisk /dev/sda
Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200   374484991   186192896   8e  Linux LVM
/dev/sda3       374484992   585871963   105693486   83  Linux
# pvcreate /dev/sda3
  Physical volume "/dev/sda3" successfully created.
# vgcreate docker /dev/sda3
  Volume group "docker" successfully created
```

Create thin pool
```
#  lvcreate --wipesignatures y -n thinpool docker -l 95%VG
  Logical volume "thinpool" created.
# lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG
  Logical volume "thinpoolmeta" created.

#  lvconvert -y --zero n -c 512K --thinpool docker/thinpool --poolmetadata docker/thinpoolmeta
  WARNING: Converting logical volume docker/thinpool and docker/thinpoolmeta to thin pool's data and metadata volumes with metadata wiping.
  THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
  Converted docker/thinpool to thin pool.
```

Configure thinpool autoentend
```
# vi /etc/lvm/profile/docker-thinpool.profile
activation {
    thin_pool_autoextend_threshold=80
    thin_pool_autoextend_percent=20
}

#  lvchange --metadataprofile docker-thinpool docker/thinpool
  Logical volume docker/thinpool changed.
```

check thinpool status
```
[root@node1 ~]# lvs -o+seg_monitor
  LV       VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Monitor
  thinpool docker twi-a-t---  95.75g             0.00   0.01                             monitored
```

Configure docker to use lvm
```
# cat /etc/sysconfig/docker-storage
DOCKER_STORAGE_OPTIONS = --storage-driver=devicemapper --storage-opt dm.fs=xfs --storage-opt dm.thinpooldev=/dev/mapper/docker-thinpool --storage-opt dm.use_deferred_removal=true
```

Clear docker data and restart docker(you need backup your /var/lib/docker directory first)
```
[root@node1 docker.service.d]# systemctl stop docker
# rm -rf /var/lib/docker/*
[root@node1 docker.service.d]# systemctl daemon-reload
[root@node1 docker.service.d]# systemctl start docker
# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.12.2
Storage Driver: devicemapper
 Pool Name: docker-thinpool
 Pool Blocksize: 524.3 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 Data file:
 Metadata file:
 Data Space Used: 19.92 MB
 Data Space Total: 102.8 GB
 Data Space Available: 102.8 GB
 Metadata Space Used: 155.6 kB
 Metadata Space Total: 1.082 GB
 Metadata Space Available: 1.082 GB
 Thin Pool Minimum Free Space: 10.28 GB
```

Load a image and run a container to check lvm status</br>
memo: I save the oracle linux 7 images first, if your server can connect docker hub, you can pull the image use command: docker pull oracle/oraclelinux:7 

```
# docker load -i docker_image_oraclelinux_7.tar
1cd9d5368290: Loading layer 228.6 MB/228.6 MB
Loaded image: localhost:5000/oraclelinux:7
# docker run -itd localhost:5000/oraclelinux:7 /bin/sh
64bde619b270ff58387a16aa80f924256f24a46273b08a30d12aa4103a641903
# docker ps
CONTAINER ID        IMAGE                          COMMAND             CREATED             STATUS              PORTS               NAMES
64bde619b270        localhost:5000/oraclelinux:7   "/bin/sh"           4 seconds ago       Up 3 seconds                            drunk_shockley
# lvs -o+seg_monitor
  LV       VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Monitor
  thinpool docker twi-aot---  95.75g             0.29   0.02                             monitored
```
we can found the image use thinpool space.

3 Configure Docker use Bridge network and connect outside direct.

Docker container use docker0 bridge and auto-setup the ip(172.x.x.x). we can't connect the container ousite server(must use port remap or overlay network), but remap and overlay network is not stable enough. so we need connect docker to ouside direct and setup ip manualy.

Install net tools
```
# yum install bridge-utils
# yum install net-tools
```

Creat a pub_net use mvcvlan.
```
# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
b3a5a54e4101        bridge              bridge              local
837df8131ced        host                host                local
f139e01dbc6a        none                null                local
# docker network create -d macvlan --subnet=10.1.107.0/26 --gateway=10.1.107.62 -o parent=bond0 -o macvlan_mode=bridge pub_net
797a015f17a9a6d069f867c17805d599f8547175b82c4ead82dafc76ec505663
```

run 2 container and setup ip.
```
# docker run -itd --net=pub_net --ip=10.1.107.4 --name=test04 localhost:5000/oraclelinux:7 /bin/sh
cd7458d407df5158e51ad4f19e16e3451933751387cbfe7971f7e12ed1678950
[root@node1 ~]# docker run -itd --net=pub_net --ip=10.1.107.5 --name=test05 localhost:5000/oraclelinux:7 /bin/sh
4c9b4e1e284871679bb53661f3bf6702428b285db7898e6167fc51f595f65a2b
```

Use ping test
```
# docker exec -it cd ping 10.1.107.4        OK
# docker exec -it cd ping 10.1.107.5        OK
# docker exec -it cd ping 10.1.107.2        OK
# docker exec -it cd ping 10.1.107.1        ERROR
```
It's work, Because MACVLAN can't connect the docker to local network. so container can't ping host network. Please search MACVLAN document

If you use virtualbox or vmware, you need open the network promisc mode,(both in virtualbox and in host os)
```
# ifconfig eth0 promisc
# ifconfig eth0 -promisc
```

#Docker local registry deploy （only for node1）
We deploy local registry on node1

1 Install docker registry
```
[root@node1 ~]# docker load -i docker_image_registry_2.tar
[root@node1 ~]# mkdir -p /var/lib/registry/conf.d
[root@node1 ~]# cd /var/lib/registry/conf.d/
[root@node1 conf.d]# vi /etc/pki/tls/openssl.cnf  增加一行：
[ v3_ca ]  
subjectAltName = IP:10.1.107.1
```
memo: if you havn't modify openssl.cnf, you will meet the error:"cannot validate certificate for 10.1.107.1 because it doesn't contain any IP SANs… "

```
[root@node1 conf.d]# openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 3650 -out domain.crt
Generating a 4096 bit RSA private key
...............++
...++
writing new private key to 'domain.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Beijing
Locality Name (eg, city) [Default City]:Chaoyang
Organization Name (eg, company) [Default Company Ltd]:Oracle
Organizational Unit Name (eg, section) []:Linux
Common Name (eg, your name or your server's hostname) []:frank
Email Address []:
[root@node1 conf.d]# ls
domain.crt  domain.key
[root@node1 conf.d]# chmod 600 domain.key
[root@node1 conf.d]# mkdir -p /etc/docker/certs.d/10.1.107.1:5000
[root@node1 conf.d]# cp /var/lib/registry/conf.d/domain.crt /etc/docker/certs.d/10.1.107.1:5000/ca.crt
[root@node1 conf.d]# chcon -Rt svirt_sandbox_file_t /var/lib/registry/conf.d/
[root@node1 lib]# chcon -Rt svirt_sandbox_file_t /var/lib/registry/
```
you need modify the file content because SELinux

Start registry use key
```
[root@node1 conf.d]# docker run -d -p 5000:5000 --name registry --restart=always \
-v /var/lib/registry:/registry_data \
-e REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/registry_data \
-e REGISTRY_HTTP_TLS_KEY=/registry_data/conf.d/domain.key \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/registry_data/conf.d/domain.crt \
registry:2
[root@node1 conf.d]# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      965/sshd
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1216/master
tcp6       0      0 :::5000                 :::*                    LISTEN      4897/docker-proxy
tcp6       0      0 :::22                   :::*                    LISTEN      965/sshd
tcp6       0      0 ::1:25                  :::*                    LISTEN      1216/master
[root@node1 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
5aa63079866a        registry:2          "/entrypoint.sh /etc/"   39 minutes ago      Up About a minute   0.0.0.0:5000->5000/tcp   registry
```

2 upload ol6 and ol7 docker image into docker registry
```
[root@node1 ~]# docker load -i docker_image_oraclelinux_6.tar
e029908bae4b: Loading layer 164.9 MB/164.9 MB
Loaded image: oracle/oraclelinux:6
[root@node1 ~]# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
registry                     2                   182810e6ba8c        2 weeks ago         37.6 MB
localhost:5000/oraclelinux   7                   accae9d046b5        3 weeks ago         219.4 MB
oracle/oraclelinux           6                   0636f157b6b8        3 weeks ago         157.7 MB
[root@node1 ~]# docker tag oracle/oraclelinux:6 10.1.107.1:5000/oraclelinux:6
[root@node1 ~]# docker tag localhost:5000/oraclelinux:7 10.1.107.1:5000/oraclelinux:7
[root@node1 ~]# docker tag registry:2 10.1.107.1:5000/registry:2

[root@node1 ~]# docker push 10.1.107.1:5000/registry
[root@node1 ~]# docker push 10.1.107.1:5000/oraclelinux:7
[root@node1 ~]# docker push 10.1.107.1:5000/oraclelinux:6
```
have a test:
```
[root@node1 ~]# docker rmi 10.1.107.1:5000/oraclelinux:7
[root@node1 ~]# docker pull 10.1.107.1:5000/oraclelinux:7
7: Pulling from oraclelinux
2af59c6d346c: Extracting 48.46 MB/78.73 MB
```

configure node2 use this registry:
```
[root@node1 docker]# scp -r certs.d/ root@10.1.107.2:/etc/docker/
[root@node3 docker]# systemctl restart docker
[root@node3 docker]# docker pull 10.1.107.1:5000/oraclelinux:7
```

# Use Docker

1 container create\start\stop\delete
```
[root@node1 docker]# docker run -itd --name testc01 oraclelinux:latest /bin/sh
44c3664bbfb0856ada79c70cc4b13c24af61d2147f863d653f9729917f2d8616
[root@node1 docker]# docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                    NAMES
44c3664bbfb0        oraclelinux:latest             "/bin/sh"                5 seconds ago       Up 4 seconds                                 testc01
```
inspect docker container message
```
[root@node1 docker]# docker inspect 44
[
    {
        "Id": "44c3664bbfb0856ada79c70cc4b13c24af61d2147f863d653f9729917f2d8616",
        "Created": "2017-01-17T15:17:38.417766233Z",
        "Path": "/bin/sh",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
…
"Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "43f8a3e7cdfd54ed215820516482bcff3c1ee276514ac64960f0abca2fbd6f0d",
                    "EndpointID": "f3316f76ecc4f8594e1474dc5a6a85550104754f79c42c632fcce3aa2aa39581",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03"
                }
…
```

go into container:
```
[root@node1 docker]# docker exec -it 44 /bin/sh
sh-4.2# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
sh-4.2# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
58: eth0@if59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:3/64 scope link
       valid_lft forever preferred_lft forever
sh-4.2# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 15:17 ?        00:00:00 /bin/sh
root         7     0  0 15:24 ?        00:00:00 /bin/sh
root        15     7  0 15:24 ?        00:00:00 ps -ef

sh-4.2# exit
exit
```

stop container
```
[root@node1 docker]# docker stop 44
44
[root@node1 docker]# docker ps -a|grep 44
44c3664bbfb0        oraclelinux:latest             "/bin/sh"                8 minutes ago       Exited (137) 34 seconds ago                            testc01
```

delete container
```
[root@node1 docker]# docker rm 44
44
[root@node1 docker]# docker ps -a|grep 44
[root@node1 docker]#
```

2 run container use user-define network and setup ip

memo:the network must be user-define network by "docker network create" command
```
# docker run -itd --net=pub_net --ip=10.1.107.4 --name=test04 localhost:5000/oraclelinux:7 /bin/sh
```

3 Container cpu quota
docker can limited cpu\mem\io for a container, we demo the CPU quota setup:
```
[root@node1 docker]# docker run -itd --cpu-period=10000 --cpu-quota=1000 --name=testc03 oraclelinux:latest /bin/sh
72c0d8c42421df45256b0d226fd671434e3dd1f0a6be4d4aa58600b5bdfe9b3d
```
period is 10ms，and this container cpu quota is 1ms，so, it can use 10% CPU max

exec cpu load in container:
```
[root@node1 docker]# docker exec -it 72 /bin/sh
sh-4.2# while true; do true;done
```
lockup the cpu status in node1:
```
[root@node1 data]# top
…
%Cpu17 : 10.6 us,  0.0 sy,  0.0 ni, 89.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
20401 root      20   0   11764   2916   2580 R  10.0  0.0   0:05.62 sh
```
exec another cpu load.
```
[root@node1 ~]# docker exec -it 72 /bin/sh
sh-4.2# while true; do true;done

[root@node1 data]# top
%Cpu2  :  0.0 us,  0.3 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu4  :  0.0 us,  0.3 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu5  :  5.6 us,  0.0 sy,  0.0 ni, 94.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu6  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu14 :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu15 :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu16 :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu17 :  5.3 us,  0.0 sy,  0.0 ni, 94.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
20401 root      20   0   11764   2916   2580 R   5.0  0.0   0:16.96 sh
20451 root      20   0   11764   2856   2636 R   5.0  0.0   0:01.90 sh

```
max cpu load is 10% </br>
memo: for X86, if you have 8 cpus，the total cpu performance is 800%

# Weblogic container create and use

1 create weblogic container

download server-jre-8u112-linux-x64.tar.gz and fmw_12.2.1.2.0_wls_Disk1_1of1.zip from oracle. <br>
download oracle image files from github: <br>
https://github.com/oracle/docker-images <br>
then untar the dockerfiles into node1 <br>

first create OracleJava image：
```
/root/docker-images-master/OracleJava
[root@node1 OracleJava]# cp /root/server-jre-8u112-linux-x64.tar.gz  ./java-8/
[root@node1 OracleJava]# cd java-8/
[root@node1 java-8]# ls
build.sh    server-jre-8u101-linux-x64.tar.gz.download
Dockerfile  server-jre-8u112-linux-x64.tar.gz
```
Because this dockfile use "FROM oraclelinux:lastest", so we need tag a oracle linux 7 image.
```
# docker tag 10.1.107.1:5000/oraclelinux:7 oraclelinux:latest
# docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
10.1.107.1:5000/oraclelinux        7                   accae9d046b5        3 weeks ago         219.4 MB
oraclelinux                        latest              accae9d046b5        3 weeks ago         219.4 MB
```
Create docker JAVA image 
```
[root@node1 java-8]# ./build.sh
```

base on Oracle Java Image, Create Weblogic image
```
[root@node1 dockerfiles]# pwd
/root/docker-images-master/OracleWebLogic/dockerfiles
[root@node1 dockerfiles]# cp /root/fmw_12.2.1.2.0_wls_Disk1_1of1.zip ./12.
12.1.3/   12.2.1/   12.2.1.1/ 12.2.1.2/
[root@node1 dockerfiles]# cp /root/fmw_12.2.1.2.0_wls_Disk1_1of1.zip ./12.2.1.2/

[root@node1 dockerfiles]# ./buildDockerImage.sh -v 12.2.1.2 -g
```

Then wo use sample, create weblogic domain image:<br>
memo: we only have "12.2.1-domain" in example, so we need copy 12.2.1-domain to 12.2.1.2-domain and modify it use oracle/weblogic:12.2.1.2-generic base-image:
```
[root@node1 samples]# docker  images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
oracle/weblogic                    12.2.1.2-generic    6099f5cd1ca2        About an hour ago   2.049 GB
oracle/serverjre                   8                   af61f5a90c03        2 hours ago         376.6 MB

[root@node1 samples]# pwd
/root/docker-images-master/OracleWebLogic/samples
[root@node1 12212-domain]# cp -rf 1221-domain 12212-domain
[root@node1 samples]# vi 12212-domain/Dockerfile
# Pull base image
# ---------------
FROM oracle/weblogic:12.2.1.2-generic
...

[root@node1 samples]# cd 12212-domain
[root@node1 12212-domain]# ./build.sh welcome1  
```
memo: "welcome1" is the weblogic login password. and after build, you could create a image named 12212-domain:latest. if you want modify the name, you need modify build.sh.
```
[root@node1 12212-domain]# cat build.sh
#!/bin/sh
if [ "$#" -eq 0 ]; then echo "Inform a password for the domain as first argument."; exit; fi
docker build --build-arg ADMIN_PASSWORD=$1 -t 12212-domain .
```

Now,we have some images:
```
[root@node1 samples]# docker  images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
12212-domain                       latest              46ed643ad16d        About an hour ago   2.05 GB
oracle/weblogic                    12.2.1.2-generic    6099f5cd1ca2        About an hour ago   2.049 GB
oracle/serverjre                   8                   af61f5a90c03        2 hours ago         376.6 MB
```

3 Start Weblogic</br>
Fristly, start weblogic admin server
```
[root@node1 ~]# docker run  -d --net=pub_net --ip=10.1.107.6 --name=wls_admin 12212-domain
[root@node1 ~]# docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                    NAMES
ff908e9df5bd        12212-domain                   "startWebLogic.sh"       About an hour ago   Up About an hour                             wls_admin
```
Now you can visit http://10.1.107.6:8001/console and login use weblogic/welcome1

Next, Start a Managed server
```
[root@node1 ~]# docker run -d --net=pub_net --ip=10.1.107.7 --name=wls_server -e ADMIN_HOST=10.1.107.6 12212-domain createServer.sh
[root@node1 ~]# docker logs --tail=all 3b
…
<Jan 17, 2017, 1:40:41,12 PM UTC> <Notice> <WebLogicServer> <BEA-000360> <The server started in RUNNING mode.>
<Jan 17, 2017, 1:40:41,23 PM UTC> <Notice> <WebLogicServer> <BEA-000365> <Server state changed to RUNNING.>
```

Please check weblogic status in Weblogic-Admin

Now, we start a weblogic Managed server on node2</br>
push images into registry
```
[root@node1 12212-domain]# docker tag 12212-domain:latest 10.1.107.1:5000/wls-domain:12.2.1.2
[root@node1 12212-domain]# docker push 10.1.107.1:5000/wls-domain:12.2.1.2
```
If you havn't copy the key to node2, run command in node1:
```
[root@node1 12212-domain]# cd /etc/docker/certs.d/
[root@node1 certs.d]# ls
10.1.107.1:5000
[root@node1 certs.d]# ls
10.1.107.1:5000
[root@node1 certs.d]# cd ../
[root@node1 docker]# scp -r certs.d/ root@10.1.107.2:/etc/docker/
```

pull weblogic images and run weblogic Managed Server on node2:
```
[root@node2 docker]# docker pull 10.1.107.1:5000/wls-domain:12.2.1.2
[root@node2 docker]# docker run -d --net=pub_net --ip=10.1.107.8 --name=wls_server3 -e ADMIN_HOST=10.1.107.6 10.1.107.1:5000/wls-domain:12.2.1.2 createServer.sh
```
Then you can found new server in Weblogic Admin.

Memo: when run weblogic managed server container, you can use some ENV for configure it,mostly use ENV is below:</br>
ADMIN_HOST: Admin server IP, default is wlsadmin </br>
CLUSTER_NAME: the cluster name, default is DockerCluster </br>

more information you can found in github:https://github.com/oracle/docker-images/tree/master/OracleWebLogic
