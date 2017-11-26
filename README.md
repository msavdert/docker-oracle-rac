# docker-oracle-rac
Oracle 12.2 RAC on Docker

## Description

- Docker Host Information (on GCE)

|||
|-----|-----|
|Operating System|Centos Linux 7.x|
|OSType_Architecture|Linux_86_64|
|Kernel Version|3.10.0-693.5.2.el7.x86_64|
|CPUs|4|
|Memory|16GB|

- Network infomation

|Name|Address IP|Type IP|Interface|
|--------|--------|-------|-------|
|storage|192.168.100.20|Public IP|eth0|
|rac1|192.168.100.10|Public IP|eth0|
|rac1-vip|192.168.100.12|Virtual IP (vip)|-|
|rac1.priv|10.10.10.10|Private IP|eth1|
|rac2|192.168.100.11|Public IP|eth0|
|rac2-vip|192.168.100.13|Virtual IP (vip)|-|
|rac2.priv|10.10.10.11|Private IP|eth1|
|scan1.vip|192.168.100.14|SCAN IP|-|
|scan2.vip|192.168.100.15|SCAN IP|-|
|scan3.vip|192.168.100.16|SCAN IP|-|

- Storage infomation 

|Diskgroup name|use|asm device path|redundancy|size(GB|
|--------|--------|-------|-------|-------|
|VOTE|ocr and voting disk|/u01/asmdisks/disk6|external|47104|
|DATA|Database files|/u01/asmdisks/disk1,/u01/asmdisks/disk2|external|40
|FRA|flash recovery area|/u01/asmdisks/disk3|external|20


## Setup

### 1. Create swap

    dd if=/dev/zero of=/swapfile bs=16384 count=1M
    mkswap /swapfile
    echo "/swapfile none swap sw 0 0" >> /etc/fstab
    chmod 600 /swapfile
    swapon -a

### 2. Install docker

    curl -fsSL https://get.docker.com/ | sh

### 3. enable docker service

    systemctl start docker 
    systemctl enable docker

### 4. Create asm disks

    mkdir -p /depo/asm/

    dd if=/dev/zero of=/depo/asm/asm-data01 bs=1024k count=20000
    dd if=/dev/zero of=/depo/asm/asm-data02 bs=1024k count=20000
    dd if=/dev/zero of=/depo/asm/asm-fra01 bs=1024k count=20000
    dd if=/dev/zero of=/depo/asm/asm-crs01 bs=1024k count=2000
    dd if=/dev/zero of=/depo/asm/asm-crs02 bs=1024k count=2000
    dd if=/dev/zero of=/depo/asm/asm-crs03 bs=1024k count=2000

### 5. download Oracle 12c Release 2 (12.2) Clusterware and Database software and locate them on /depo/12.2/

    # ls -al /depo/12.2/
    total 6297260
    -rw-r--r--. 1 root root 3453696911 Feb 24  2017 linuxx64_12201_database.zip
    -rw-r--r--. 1 root root 2994687209 Oct 16 20:07 linuxx64_12201_grid_home.zip

### 6. Create Docker Network for RAC and NFS&DNS Containers

    docker network create --driver=bridge \
    --subnet=192.168.100.0/24 --gateway=192.168.100.1 \
    --ip-range=192.168.100.128/25 pub

    docker network create --driver=bridge \
    --subnet=10.10.10.0/24 --gateway=10.10.10.1 \
    --ip-range=10.10.10.128/25 priv

### 7. Start NFS&DNS Server Container

    docker run \
    --detach \
    --privileged \
    -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
    --volume=/depo/asm/:/asmdisks \
    -e TZ=Europe/Istanbul \
    --name nfs \
    --hostname nfs.example.com \
    --net pub \
    --ip 192.168.100.20 \
    melihsavdert/docker-nfs-dns-server

### 8. Start two containers (rac1 and rac2) for rac installation

	docker run --rm \
	--privileged \
	--detach \
	--name rac1 \
	-h rac1.example.com \
	--net pub \
	--add-host nfs:192.168.100.127 \
	--ip 192.168.100.10 \
	-p 1521:1521 -p 9803:9803 -p 1158:1158 \
	-p 5500:5500 -p 7803:7803 -p 7102:7102 \
	--shm-size 2048m \
	-e TZ=Europe/Istanbul \
	-v /sys/fs/cgroup:/sys/fs/cgroup:ro \
	--volume /depo:/software \
	--volume /boot:/boot \
	melihsavdert/docker-oracle-rac
	
	docker run --rm \
	--privileged \
	--detach \
	--name rac2 \
	-h rac2.example.com \
	--net pub \
	--add-host nfs:192.168.100.20 \
	--ip 192.168.100.11 \
	-p 1522:1521 -p 9804:9803 -p 1159:1158 \
	-p 5501:5500 -p 7804:7803 -p 7103:7102 \
	--shm-size 2048m \
	-e TZ=Europe/Istanbul \
	-v /sys/fs/cgroup:/sys/fs/cgroup:ro \
	--volume /depo:/software \
	--volume /boot:/boot \
	melihsavdert/docker-oracle-rac

### 9. Connect the private network to the RAC cluster containers.

	docker network connect --ip 10.10.10.10 priv rac1
	docker network connect --ip 10.10.10.11 priv rac2

### 10. Configure DNS for RAC cluster containers

	docker exec -it rac1 cp /etc/resolv.conf /tmp/resolv.conf && \
	docker exec -it rac1 sed -i '/search/s/$/ example.com\nnameserver 192.168.100.20/' /tmp/resolv.conf && \
	docker exec -it rac1 cp -f /tmp/resolv.conf /etc/resolv.conf

	docker exec -it rac2 cp /etc/resolv.conf /tmp/resolv.conf && \
	docker exec -it rac2 sed -i '/search/s/$/ example.com\nnameserver 192.168.100.20/' /tmp/resolv.conf && \
	docker exec -it rac2 cp -f /tmp/resolv.conf /etc/resolv.conf

### 11. Mount NFS server for RAC cluster containers and give permissions

	docker exec -it rac1 mount /u01/asmdisks/ && \
	docker exec -it rac1 chown -R grid:asmadmin /u01/asmdisks/ && \
	docker exec -it rac1 chmod -R 777 /u01/asmdisks/

	docker exec -it rac2 mount /u01/asmdisks/ && \
	docker exec -it rac2 chown -R grid:asmadmin /u01/asmdisks/ && \
	docker exec -it rac2 chmod -R 777 /u01/asmdisks/

### 12. Edit /etc/hosts file each nodes as follows 

	docker exec -it rac1 vi /etc/hosts
	docker exec -it rac2 vi /etc/hosts

	# Public
	192.168.100.10 rac1.example.com rac1
	192.168.100.11 rac2.example.com rac2
	# Private
	#10.10.10.10 rac1-priv.example.com rac1-priv
	#10.10.10.11 rac2-priv.example.com rac2-priv
	# Virtual
	#192.168.100.12 rac1-vip.example.com rac1-vip
	#192.168.100.13 rac2-vip.example.com rac2-vip
	# SCAN
	#192.168.100.14 rac-scan.example.com rac-scan
	#192.168.100.15 rac-scan.example.com rac-scan
	#192.168.100.16 rac-scan.example.com rac-scan

### 13. Check DNS is healthy

	$ docker exec -it rac1 nslookup rac-scan
	Server:		192.168.100.20
	Address:	192.168.100.20#53

	Name:	rac-scan.example.com
	Address: 192.168.100.16
	Name:	rac-scan.example.com
	Address: 192.168.100.15
	Name:	rac-scan.example.com
	Address: 192.168.100.14

### 14. Copy Oracle Database and Grid Infrastructure inside the rac1 container

	docker exec -it rac1 su - oracle -c ' \
	unzip -q /software/12.2/linuxx64_12201_database.zip -d /u01/software/'

	docker exec -it rac1 su - grid -c ' \
	unzip -q /software/12.2/linuxx64_12201_grid_home.zip -d /u01/app/12.2.0.1/grid/'

### 15. Install the cvuqdisk package on each cluster nodes

	docker exec -it rac1 rpm -Uvh /u01/app/12.2.0.1/grid/cv/rpm/cvuqdisk*
	docker exec -it rac1 scp /u01/app/12.2.0.1/grid/cv/rpm/cvuqdisk-1.0.10-1.rpm root@192.168.100.11:/root
	docker exec -it rac2 rpm -Uvh /root/cvuqdisk*
	
### 16. Edit grid user .bash_profile for ORACLE_SID as follows
	
	docker exec -it rac1 su - grid -c ' \
	echo -e "export ORACLE_SID=+ASM1" >> /home/grid/.bash_profile && source /home/grid/.bash_profile'

	ocker exec -it rac2 su - grid -c ' \
	echo -e "export ORACLE_SID=+ASM2" >> /home/grid/.bash_profile && source /home/grid/.bash_profile'

### 17. Configure the Grid Infrastructure by running the following as the "grid" user.

	docker exec -it rac1 su - grid -c ' \
	/u01/app/12.2.0.1/grid/gridSetup.sh -silent \
	-ignorePrereqFailure \
	-responseFile /u01/app/12.2.0.1/grid/install/response/gridsetup.rsp \
	INVENTORY_LOCATION=/u01/app/oraInventory \
	oracle.install.option=CRS_CONFIG \
	ORACLE_BASE=/u01/app/grid \
	oracle.install.asm.OSDBA=asmdba \
	oracle.install.asm.OSOPER=asmoper \
	oracle.install.asm.OSASM=asmadmin \
	oracle.install.crs.config.gpnp.scanName=rac-scan \
	oracle.install.crs.config.gpnp.scanPort=1521 \
	oracle.install.crs.config.ClusterConfiguration=STANDALONE \
	oracle.install.crs.config.configureAsExtendedCluster=false \
	oracle.install.crs.config.clusterName=rac \
	oracle.install.crs.config.gpnp.configureGNS=false \
	oracle.install.crs.config.autoConfigureClusterNodeVIP=false \
	oracle.install.crs.config.clusterNodes=rac1.example.com:rac1-vip.example.com:HUB,rac2.example.com:rac2-vip.example.com:HUB \
	oracle.install.crs.config.networkInterfaceList=eth0:192.168.100.0:1,eth1:10.10.10.0:5 \
	oracle.install.asm.configureGIMRDataDG=false \
	oracle.install.asm.storageOption=ASM \
	oracle.install.asmOnNAS.configureGIMRDataDG=false \
	oracle.install.asm.SYSASMPassword=oracle \
	oracle.install.asm.diskGroup.name=DATA \
	oracle.install.asm.diskGroup.redundancy=EXTERNAL \
	oracle.install.asm.diskGroup.AUSize=4 \
	oracle.install.asm.diskGroup.disks=/u01/asmdisks/asm-data01,/u01/asmdisks/asm-data02 \
	oracle.install.asm.diskGroup.diskDiscoveryString=/u01/asmdisks/asm* \
	oracle.install.asm.monitorPassword=oracle \
	oracle.install.asm.configureAFD=false \
	oracle.install.crs.configureRHPS=false \
	oracle.install.crs.config.ignoreDownNodes=false \
	oracle.install.config.managementOption=NONE \
	oracle.install.config.omsPort=0 \
	oracle.install.crs.rootconfig.executeRootScript=false \
	-waitForCompletion'

### 18. Execute the root script each nodes

	As a root user, execute the following script(s):
		1. /u01/app/12.2.0.1/grid/root.sh

	docker exec -it rac1 /u01/app/12.2.0.1/grid/root.sh
	docker exec -it rac2 /u01/app/12.2.0.1/grid/root.sh
	
### 19. As install user, execute the following command to complete the configuration

	docker exec -it rac1 su - grid -c ' \
	/u01/app/12.2.0.1/grid/gridSetup.sh -executeConfigTools -silent \
	-ignorePrereqFailure \
	-responseFile /u01/app/12.2.0.1/grid/install/response/gridsetup.rsp \
	INVENTORY_LOCATION=/u01/app/oraInventory \
	oracle.install.option=CRS_CONFIG \
	ORACLE_BASE=/u01/app/grid \
	oracle.install.asm.OSDBA=asmdba \
	oracle.install.asm.OSOPER=asmoper \
	oracle.install.asm.OSASM=asmadmin \
	oracle.install.crs.config.gpnp.scanName=rac-scan \
	oracle.install.crs.config.gpnp.scanPort=1521 \
	oracle.install.crs.config.ClusterConfiguration=STANDALONE \
	oracle.install.crs.config.configureAsExtendedCluster=false \
	oracle.install.crs.config.clusterName=rac \
	oracle.install.crs.config.gpnp.configureGNS=false \
	oracle.install.crs.config.autoConfigureClusterNodeVIP=false \
	oracle.install.crs.config.clusterNodes=rac1.example.com:rac1-vip.example.com:HUB,rac2.example.com:rac2-vip.example.com:HUB \
	oracle.install.crs.config.networkInterfaceList=eth0:192.168.100.0:1,eth1:10.10.10.0:5 \
	oracle.install.asm.configureGIMRDataDG=false \
	oracle.install.asm.storageOption=ASM \
	oracle.install.asmOnNAS.configureGIMRDataDG=false \
	oracle.install.asm.SYSASMPassword=oracle \
	oracle.install.asm.diskGroup.name=DATA \
	oracle.install.asm.diskGroup.redundancy=EXTERNAL \
	oracle.install.asm.diskGroup.AUSize=4 \
	oracle.install.asm.diskGroup.disks=/u01/asmdisks/asm-data01,/u01/asmdisks/asm-data02 \
	oracle.install.asm.diskGroup.diskDiscoveryString=/u01/asmdisks/asm* \
	oracle.install.asm.monitorPassword=oracle \
	oracle.install.asm.configureAFD=false \
	oracle.install.crs.configureRHPS=false \
	oracle.install.crs.config.ignoreDownNodes=false \
	oracle.install.config.managementOption=NONE \
	oracle.install.config.omsPort=0 \
	oracle.install.crs.rootconfig.executeRootScript=false \
	-waitForCompletion'
	
### 20. Check status of grid software
	
	docker exec -it rac1 su - grid -c ' crsctl check cluster -all'
	docker exec -it rac1 su - grid -c ' crs_stat -t'
	docker exec -it rac1 su - grid -c ' crsctl stat res -t'
	
### 21. 	
	
