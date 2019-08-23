
# HDP install using Ambari in Ubuntu 16.04
## Prerequisites
### Hardware
- RAM: 10 Gb
- CPU: 4 cores 2.4 GHz
- HDD: 150 Gb

### Software
- Python dev packages
	```sh
	sudo apt install python-dev
	```
- OpenSSH
	```sh
	sudo apt install openssh-server
	```
- Java JDK
	```sh
	sudo apt-get install default-jdk
	```
	Edit .bashrc file to load java
	```sh
	cd ~
	sudo nano .bashrc
	```
	Add the following line to the end of the .bashrc file
	```sh
	export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
	```
	Reload CLI
	```sh
	. .bashrc
	```	
## Install Guide
1. Login as root user on master and node machines
	```sh
	sudo su
	```
2. Edit the Maximum Open Files
- Verify the maximum numbers
	```sh
	ulimit -n -u
	```
- Edit the maximum files
	```sh
	ulimit -n 32768
	ulimit -u 65536
	```
- Edit the maximum files permanently on limits.conf file
	```sh
	nano /etc/security/limits.conf
	```
	Add the following lines above `# End of file`
	```
	root - nofile 32768
	root - nproc 65536
	```
3. Set Up Password-less SSH
- Execute ssh-keygen 
	```sh
	ssh-keygen
	```
- Copy the keygen to each host
	```sh
	ssh-copy-id HOSTIP
	```
	Where `HOSTIP` is the IP of the other host that will be in the cluster
- Test connection to HOSTIP within each machine
	```sh
	ssh HOSTIP
	```
4. Install ntp in each host
	```sh
	apt install ntp -y
	```
- Edit ntp.conf to add region ntp servers (check [https://www.pool.ntp.org/es/](https://www.pool.ntp.org/es/))
	```sh
	nano /etc/ntp.conf
	```
	Change pool servers to match region (in this case Europe)
	```sh
			   pool 0.europe.pool.ntp.org
			   pool 1.europe.pool.ntp.org
			   pool 2.europe.pool.ntp.org
			   pool 3.europe.pool.ntp.org
	```
- Start ntp service and check status with each server
	```sh
	service ntp start
	service ntp status
	ntpq -p
	```
5. Set the Hostname of each host
	**For master:**
	- Setup temporary hostname
		```sh
		hostname ambari.local
		hostname
		```
	- Edit in hosts file
		```sh
		nano /etc/hosts
		```
		Add master and nodes, edit also hostnames as follows:
		```sh
		127.0.0.1       localhost
		127.0.1.1       ambari.local

		# The following lines are desirable for IPv6 capable hosts
		::1     ip6-localhost ip6-loopback
		fe00::0 ip6-localnet
		ff00::0 ip6-mcastprefix
		ff02::1 ip6-allnodes
		ff02::2 ip6-allrouters

		IPMASTER	ambari.local	ambari
		IPNODE1	node1.local	node1
		```
		Where `IPMASTER` corresponds to the master ip address and `IPNODE1`  to the node1 ip address (use `ifconfig` to see the ip4)

	- Edit in hostname file
		```sh
		nano /etc/hostname
		```
		Change the old hostname with the new ambari.local name inside the hostname file
		```sh
		ambari.local
		```	

	- Change permantly on the system the hostname
		```sh
		hostnamectl set-hostname ambari.local
		```	
	**For nodes:**
	- Setup temporary hostname
		```sh
		hostname node1.local
		hostname
		```
	- Edit in hosts file
		```sh
		nano /etc/hosts
		```
		Add master and nodes, edit also hostnames as follows:
		```sh
		127.0.0.1       localhost
		127.0.1.1       node1.local

		# The following lines are desirable for IPv6 capable hosts
		::1     ip6-localhost ip6-loopback
		fe00::0 ip6-localnet
		ff00::0 ip6-mcastprefix
		ff02::1 ip6-allnodes
		ff02::2 ip6-allrouters

		IPMASTER	ambari.local	ambari
		IPNODE1	node1.local	node1
		```
		Where `IPMASTER` corresponds to the master ip address and `IPNODE1`  to the node1 ip address (use `ifconfig` to see the ip4)

	- Edit in hostname file
		```sh
		nano /etc/hostname
		```
		Change the old hostname with the new ambari.local name inside the hostname file
		```sh
		node1.local
		```	

	- Change permantly on the system the hostname
		```sh
		hostnamectl set-hostname node1.local
		```	
- Reboot the master and nodes to reflect the new hostname, then verify that hostname was updated
	```sh
	hostname
	hostname -f
	```	

6. Disable firewall on master and nodes
	```sh
	sudo ufw disable
	sudo iptables -X
	sudo iptables -t nat -F
	sudo iptables -t nat -X
	sudo iptables -t mangle -F
	sudo iptables -t mangle -X
	sudo iptables -P INPUT ACCEPT
	sudo iptables -P FORWARD ACCEPT
	sudo iptables -P OUTPUT ACCEPT
	```
7. Add [Ambari public repository](https://docs.hortonworks.com/HDPDocuments/Ambari-2.7.3.0/bk_ambari-installation/content/ambari_repositories.html) to apt sources.list.d directory
	```sh
	cd /etc/apt/sources.list.d
	wget http://public-repo-1.hortonworks.com/ambari/ubuntu16/2.x/updates/2.7.3.0/ambari.list
	``` 
- Open ambari.list
	```sh
	nano /etc/apt/sources.list.d/ambari.list
	```	
	Add the URL as trusted
	```sh
	deb [trusted=yes] http://public-repo-1.hortonworks.com/ambari/ubuntu16/2.x/updates/2.7.3.0 Ambari main
	```	
- Execute apt-get update
	```sh
	apt-get update
	```
8. Install ambari-server and ambari-agent
	```sh
	apt install ambari-server ambari-agent
	```
9. Configure ambari-server
	```sh
	ambari-server setup
	```
	Proceed through all the defaults except the GPL related, which is disabled by default. Accept it.

10. Install postgresql-jdbc connector
	```sh
	apt install libpostgresql-jdbc-java
	```	
- Change the access mode of the .jar file to 644.
	```sh
	chmod 644 /usr/share/java/postgresql.jar
	```	
- Configure ambari-server to use JDBC connector
	```sh
	ambari-server setup --jdbc-db=postgres --jdbc-driver=/usr/share/java/postgresql.jar
	```	
- Run the following command and add it to the .bashrc file
	```sh
	export HADOOP_CLASSPATH=${HADOOP_CLASSPATH}:${JAVA_JDBC_LIBS}:/connector jar path
	```	
11. Edit the pg_hba.conf file to accept connections from all db users
- Open pg_hba.conf
	```sh
	nano /etc/postgresql/9.5/main/pg_hba.conf
	```	
	Edit it as follows
	```sh
	#host    replication     postgres        ::1/128                 md5

	local   all     all     trust
	host    all     all     0.0.0.0/0       trust
	host    all     all     ::/0    trust

	```		
- Reload postgresql database
	```sh
	sudo -u postgres /usr/lib/postgresql/9.5/bin/pg_ctl -D /etc/postgresql/9.5/main reload
	```
12. Configure ambari-agent on each node with respective name
- Open ambari-agent.ini file
	```sh
	nano /etc/ambari-agent/conf/ambari-agent.ini
	```
	Edit the hostname property to match the master server
	`hostname=ambari.local`

13. Run ambari-server and ambari-agent
	```sh
	ambari-server start
	ambari-agent start
	```	
14. Create databases for all the required services of ambari
- rangerdba
	```sh
	echo "CREATE DATABASE rangerdba;" | sudo -u postgres psql -U postgres
	echo "CREATE USER rangerdba WITH PASSWORD 'administrator1';" | sudo -u postgres psql -U postgres
	echo "GRANT ALL PRIVILEGES ON DATABASE rangerdba TO rangerdba;" | sudo -u postgres psql -U postgres 
	```	
- registry
	```
	echo "CREATE DATABASE registry;" | sudo -u postgres psql -U postgres
	echo "CREATE USER registry WITH PASSWORD 'administrator1';" | sudo -u postgres psql -U postgres
	echo "GRANT ALL PRIVILEGES ON DATABASE registry TO registry;" | sudo -u postgres psql -U postgres 
	```
- streamline
	```
	echo "CREATE DATABASE streamline;" | sudo -u postgres psql -U postgres
	echo "CREATE USER streamline WITH PASSWORD 'administrator1';" | sudo -u postgres psql -U postgres
	echo "GRANT ALL PRIVILEGES ON DATABASE streamline TO streamline;" | sudo -u postgres psql -U postgres 
	```
- druid
	```
	echo "CREATE DATABASE druid;" | sudo -u postgres psql -U postgres
	echo "CREATE USER druid WITH PASSWORD 'administrator1';" | sudo -u postgres psql -U postgres
	echo "GRANT ALL PRIVILEGES ON DATABASE druid TO druid;" | sudo -u postgres psql -U postgres 

	```
- superset
	```
	echo "CREATE DATABASE superset;" | sudo -u postgres psql -U postgres
	echo "CREATE USER superset WITH PASSWORD 'administrator1';" | sudo -u postgres psql -U postgres
	echo "GRANT ALL PRIVILEGES ON DATABASE superset TO superset;" | sudo -u postgres psql -U postgres 
	```
- hive
	```
	echo "CREATE DATABASE hive;" | sudo -u postgres psql -U postgres
	echo "CREATE USER hive WITH PASSWORD 'administrator1';" | sudo -u postgres psql -U postgres
	echo "GRANT ALL PRIVILEGES ON DATABASE hive TO hive;" | sudo -u postgres psql -U postgres
	```
- rangerkms
	```
	echo "CREATE DATABASE rangerkms;" | sudo -u postgres psql -U postgres
	echo "CREATE USER rangerkms WITH PASSWORD 'administrator1';" | sudo -u postgres psql -U postgres
	echo "GRANT ALL PRIVILEGES ON DATABASE rangerkms TO rangerkms;" | sudo -u postgres psql -U postgres 
	```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI1NDEzMTEyOV19
-->