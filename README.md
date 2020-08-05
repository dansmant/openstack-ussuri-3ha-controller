# openstack-ussuri 3 node controller HA on CentOS 8 Vertion 1.0
HA 3 node controller with 1 node compute and 1 node haproxy


reference source;

	https://docs.openstack.org/install-guide/
	https://www.server-world.info/en/note?os=CentOS_8&p=openstack_ussuri&f=2
	https://www.golinuxcloud.com/configure-openstack-high-availability-pacemaker/
	https://www.golinuxcloud.com/configure-haproxy-in-openstack-high-availability/
	https://github.com/hoperays/openstack-ha-manual-deploy
	https://hub.packtpub.com/deploying-highly-available-openstack/
	https://www.sebastien-han.fr/blog/2012/05/21/openstack-high-availability-rabbitmq/ 


deploy HA controller openstack Ussuri using 3 node controller, 1 node compute, 1 node haproxy;

Lab running on KVM, you need nestad VM to configutration VM; https://docs.fedoraproject.org/en-US/quick-docs/using-nested-virtualization-in-kvm/

and you need install virtualbmc for fencing clutser on KVM mechine;
1. https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/environments/virtualbmc.html
2. https://docs.openstack.org/virtualbmc/latest/user/index.html

example vbmc on lab;

	sentry-02 ~# vbmc list
	+------------------------+---------+------------+------+
	| Domain name            | Status  | Address    | Port |
	+------------------------+---------+------------+------+
	| Yudi-rhv               | running | 10.19.33.3 | 6012 |
	| daniel-compute1-c8x    | running | 10.19.33.3 | 6004 |
	| daniel-controller1-c8x | running | 10.19.33.3 | 6001 |
	| daniel-controller2-c8x | running | 10.19.33.3 | 6002 |
	| daniel-controller3-c8x | running | 10.19.33.3 | 6003 |
	| mysql1-cluster         | running | 10.19.33.3 | 6010 |
	| mysql2-cluster         | running | 10.19.33.3 | 6011 |
	| yudi-rhv2              | running | 10.19.33.3 | 6013 |
	+------------------------+---------+------------+------+
	sentry-02 ~#  ipmitool -H 10.19.33.3 -p 6001 -U admin -P password -I lanplus power status
	Chassis Power is on

-----
Topology deployment opentsack 3 node ha


						VIP 10.19.33.14
							|
							|
		----------------------	---------------------	----------------------	---------------------	-----------------------
		|controller1         |	|    controller2     |	| controller3        |	|compute-node        |	| haproxy             |
		|enp1s0: 10.19.33.11 |	| enp1s0: 10.19.33.12|	| enp1s0: 10.19.33.13|	|enp1s0: 10.19.33.13 |	| enp1s0: 10.19.33.15 | 
		|enp2s0: ovs         |	| enp2s0: ovs        |	| enp2s0: ovs        |	|enp2s0: ovs         |	|                     |
		----------------------	----------------------	----------------------	----------------------	-----------------------
			|                               |                              | 
			|_______________________________|______________________________|
			                                |
 			                                |
					-------------------------
					|    Storage-NFS-NODE   |
					| enp1s0: 10.19.33.22   |
					|                       |
					-------------------------

Service required at each node mentioned above;

Controller Service;
1. Service Mariadb-galera
2. Service rabbitMQ
3. Service Memcache
4. Service Pacemaker
5. Service http keystone
6. Service glance API
7. Service Cinder (Cinder-API, Cinder-Volume, Cinder-Backup, Cinder-Sheduler)
8. Service Nova (nova-api, nova-conductor, nova-scheduler).
9. Service Neutron (neutron-metadata-agent, neutron-dhcp-agent, neutron-server, neutron-l3-agent, neutron-openvswitch-agent) 
10. Service Dashboard (Horizon)


Compute Service;
1. Service nova-compute
2. neutron-openvswitch-agent

Storage Service;
Nfs server;
how to config;

! config disk for mount point nfs

	# yum install nfs-utils
	# vim /etc/exports

	/backend/glance         *(sync,rw,no_root_squash)
	/backend/cinder         *(sync,rw,no_root_squash)
	/backend/cinder-backup  *(sync,rw,no_root_squash)
	~                                               
	:wq 


	# systemctl enable --node nfs-server
	# exportfs -vr

HA proxy node;
Only haproxy service
~
Additional config: Selinux configed permissive and disable firewalld


Requirment config ntp chrony each node (config ntp ke sumua node);
	
	# yum install chrony
	# vim /etc/chrony.conf
	server 10.19.31.3 iburst
	:wq
	~
	~
----

	# systemctl restart chronyd
	# systemctl enable chronyd


Add Repository of Openstack Ussuri; 
	# dnf -y install centos-release-openstack-ussuri  
	# dnf -y install epel-release

Install Pacemaker ALL host controller
install pacemaker on controller1, controlle2 and controller3

All host controller:

	# dnf --enablerepo=HighAvailability -y install pcs pacemaker fence-agents-all fence-agents-virsh
	# systemctl start pcsd
	# systemctl enable pcsd
	# passwd hacluster ( any each host )


controller1
	
	# pcs host auth controller1 controller2 controller3 -u hacluster -p password
	# pcs cluster setup openstack-cluster controller1 controller2 controller3
	# pcs cluster start --all
	# pcs cluster enable --all
	# pcs status corosync
	# pcs property set pe-warn-series-max=1000 pe-input-series-max=1000 pe-error-series-max=1000 cluster-recheck-interval=1min
	# pcs stonith create fence_controller1 fence_ipmilan ipaddr=10.19.33.3 ipport=6001 login=admin passwd=password  lanplus=1 op monitor interval=60s
	# pcs stonith create fence_controller2 fence_ipmilan ipaddr=10.19.33.3 ipport=6002 login=admin passwd=password  lanplus=1 op monitor interval=60s
	# pcs stonith create fence_controller3 fence_ipmilan ipaddr=10.19.33.3 ipport=6003 login=admin passwd=password  lanplus=1 op monitor interval=60s
	# pcs stonith level add 1 controller1 fence_controller1
	# pcs stonith level add 1 controller2 fence_controller2
	# pcs stonith level add 1 controller3 fence_controller3

on controller 1:

	# pcs resource create controller-vip IPaddr2 ip=10.19.33.14 cidr_netmask=24 nic=enp1s0 op monitor interval=30s
	# pcs status


firwalld config all controller node;
if firwall on, better stop service of firewall

	#firewall-cmd --add-port={3306/tcp,4567/tcp,4568/tcp,4444/tcp} --permanent 
	#firewall-cmd --permanent --add-port=9300/tcp
	#firewall-cmd --permanent --add-port=3306/tcp
	#firewall-cmd --permanent --add-port=9200/tcp
	#firewall-cmd --permanent --add-port=35357/tcp
	#firewall-cmd --permanent --add-port=5000/tcp
	#firewall-cmd --permanent --add-port=9191/tcp
	#firewall-cmd --permanent --add-port=9292/tcp
	#firewall-cmd --permanent --add-port=8776/tcp
	#firewall-cmd --permanent --add-port=9696/tcp
	#firewall-cmd --permanent --add-port=6080/tcp
	#firewall-cmd --permanent --add-port=8775/tcp
	#firewall-cmd --permanent --add-port=8774/tcp
	#firewall-cmd --permanent --add-port=8777/tcp
	#firewall-cmd --permanent --add-port=4331/tcp
	#firewall-cmd --permanent --add-port=35022/udp
	#firewall-cmd --reload



================================================================================================================================================================

install ha proxy:

	# yum install -y haproxy
	#echo 'net.ipv4.ip_nonlocal_bind=1' >> /etc/sysctl.d/haproxy.conf
	#sysctl -p /etc/sysctl.d/haproxy.conf

	# systemctl stop firewalld
	# systemctl disabl firewalld

node ha proxy disable selinux configed


	# cat > /etc/sysctl.d/tcp_keepalive.conf << EOF
	> net.ipv4.tcp_keepalive_intvl = 1
	> net.ipv4.tcp_keepalive_probes = 5
	> net.ipv4.tcp_keepalive_time = 5
	EOF
	# sysctl -p /etc/sysctl.d/tcp_keepalive.conf

configuration haproxy (/etc/haproxy/haproxy.cfg)

vim /etc/haproxy/haproxy.cfg

	~
	~
	global
    	# to have these messages end up in /var/log/haproxy.log you will
    	# need to:
    	#
    	# 1) configure syslog to accept network log events.  This is done
    	#    by adding the '-r' option to the SYSLOGD_OPTIONS in
    	#    /etc/sysconfig/syslog
    	#
    	# 2) configure local2 events to go to the /var/log/haproxy.log
    	#   file. A line like the following can be added to
    	#   /etc/sysconfig/syslog
   	#
   	#    local2.*                       /var/log/haproxy.log
   	#
    	log         127.0.0.1 local2

    	chroot      /var/lib/haproxy
   	pidfile     /var/run/haproxy.pid
   	maxconn     4000
    	user        haproxy
    	group       haproxy
    	daemon

   	# turn on stats unix socket
    	stats socket /var/lib/haproxy/stats

    	# utilize system-wide crypto-policies
    	ssl-default-bind-ciphers PROFILE=SYSTEM
    	ssl-default-server-ciphers PROFILE=SYSTEM

	#---------------------------------------------------------------------
	# common defaults that all the 'listen' and 'backend' sections will
	# use if not designated in their block
	#---------------------------------------------------------------------
	defaults
   		mode                    tcp
    		log                     global
    		option                  dontlognull
   		option http-server-close
    		option forwardfor       except 127.0.0.0/8
    		option                  redispatch
    		retries                 3
    		timeout http-request    10s
    		timeout queue           1m
   		timeout connect         10s
    		timeout client          1m
    		timeout server          1m
   		timeout http-keep-alive 10s
    		timeout check           10s
   		 maxconn                 3000

	listen monitor
   		bind 10.19.33.15:9300
    		mode http
    		monitor-uri /status
    		stats enable
    		stats uri /admin
    		stats realm Haproxy\ Statistics
    		stats auth admin:password
    		stats refresh 15s

	frontend hap-ip
    		bind 10.19.33.15:3306
    		timeout client 90m
    		default_backend db-vms-galera

	backend db-vms-galera
    		option httpchk
    		stick-table type ip size 1000
    		stick on dst
    		timeout server 90m
    		server controller1 10.19.33.11:3306 check inter 1s port 9200 backup on-marked-down shutdown-sessions
    		server controller2 10.19.33.12:3306 check inter 1s port 9200 backup on-marked-down shutdown-sessions
   		server controller3 10.19.33.13:3306 check inter 1s port 9200 backup on-marked-down shutdown-sessions

	listen rabbitmq_cluster 
   		mode tcp
    		bind 10.19.33.15:4369
    		bind 10.19.33.15:25672
    		bind 10.19.33.15:15672
    		balance roundrobin
    		#stick-table type ip size 1000
    		#stick on dst
   		#timeout server 90m
    		server controller1-port-4369 10.19.33.11:4369 check inter 5000 rise 2 fall 3
   		server controller2-port-4369 10.19.33.12:4369 check inter 5000 rise 2 fall 3
    		server controller3-port-4369 10.19.33.13:4369 check inter 5000 rise 2 fall 3
    		server controller1-port-25672 10.19.33.11:25672 check inter 5000 rise 2 fall 3
    		server controller2-port-25672 10.19.33.12:25672 check inter 5000 rise 2 fall 3
    		server controller3-port-25672 10.19.33.13:25672 check inter 5000 rise 2 fall 3
   		server controller1-port-15672 10.19.33.11:15672 check inter 5000 rise 2 fall 3
    		server controller2-port-15672 10.19.33.12:15672 check inter 5000 rise 2 fall 3
    		server controller3-port-15672 10.19.33.13:15672 check inter 5000 rise 2 fall 3


	listen rabbitmq_cluster_openstack
    		mode tcp
    		bind 10.19.33.15:5672
    		balance roundrobin
    		#stick-table type ip size 1000
    		#stick on dst
    		#timeout server 90m
    		server controller1 10.19.33.11:5672 check inter 5000 rise 2 fall 3
    		server controller2 10.19.33.12:5672 check inter 5000 rise 2 fall 3
    		server controller3 10.19.33.13:5672 check inter 5000 rise 2 fall 3

	frontend http-in
    		# listen 80 port
   		bind 10.19.33.15:80
    		#bind 10.19.33.15:443
    		# set default backend
    	default_backend    backend_servers
    		# send X-Forwarded-For header
		option             forwardfor

	# define backend
	backend backend_servers
    		# balance with roundrobin
    		balance            roundrobin
   		# define backend servers
    		server  controller1-80 10.19.33.11:80 check
    		server	controller2-80 10.19.33.12:80 check
    		server	controller3-80 10.19.33.13:80 check
    		#server controller1-443 10.19.33.11:443 check
    		#server controller2-443 10.19.33.12:443 check
    		#server controller3-443 10.19.33.13:443 check
	
		~
		~
		:wq

running service ha proxy

		# systemctl enable --now

how to access haproxy admin via browser

http://10.19.33.15:9300/admin

user=admin
password=password



================================================================================================================================================================

install mariadb galera;


	# yum install mariadb-galera-server xinetd rsync

configure firewall if firewall of on;

	# firewall-cmd --add-service=mysql --permanent 
	# firewall-cmd --add-port={3306/tcp,4567/tcp,4568/tcp,4444/tcp} --permanent
	# firewall-cmd --reload 


all node controller node

	# vim /etc/sysconfig/clustercheck
----
	
	MYSQL_USERNAME="clustercheck"
	MYSQL_PASSWORD="password"
	MYSQL_HOST="localhost"
	MYSQL_PORT="3306"
	~
	:wq
-----

	# systemctl start mariadb
	# mysql -e "CREATE USER 'clustercheck'@'localhost' IDENTIFIED BY 'password';"
	# systemctl stop mariadb
----

	# mkdir /root/backup-config
	# cp /etc/my.cnf.d/galera.cnf /root/backup-config
	# mv /etc/my.cnf /root/backup-config
----	

	# vim /etc/my.cnf.d/galera.cnf

----

	[mysqld]
	skip-name-resolve=1
	innodb_locks_unsafe_for_binlog=1
	max_connections=8192
	query_cache_size=0
	query_cache_type=0
	binlog_format=ROW
	default-storage-engine=innodb
	innodb_autoinc_lock_mode=2
	bind-address=0.0.0.0
	wsrep_on=1
	wsrep_provider=/usr/lib64/galera/libgalera_smm.so
	wsrep_cluster_name="galera_cluster"
	wsrep_cluster_address="gcomm://controller1,controller2,controller3"
	wsrep_slave_threads=1
	wsrep_certify_nonPK=1
	wsrep_max_ws_rows=131072
	wsrep_max_ws_size=1073741824
	wsrep_debug=0
	wsrep_convert_LOCK_to_trx=0
	wsrep_retry_autocommit=1
	wsrep_auto_increment_control=1
	wsrep_drupal_282555_workaround=0
	wsrep_causal_reads=0
	wsrep_notify_cmd=
	wsrep_sst_method=rsync
	wsrep_sst_auth=root:

for node controller1

	# galera_new_cluster
	# systemctl enable mariadb

for node controller2 and controller3

	# systemctl enable --now mariadb

	# mysql_secure_installation

	# mysql -u root -p

	# CREATE USER 'clustercheck'@'localhost' IDENTIFIED BY 'password';

if service mariadb galera can no running, do it delete file "grastate.dat" at path /var/lib/mysql then run sevice mariadb galera

	# rm /var/lib/mysql/grastate.dat
	# galera_new_cluster

controller2 dan controller3

	# systemctl restart mariadb

----

	# vim /etc/xinetd.d/galera-monitor

	service galera-monitor
	{
    		port = 9200
    		disable = no
   		socket_type = stream
   		protocol = tcp
    		wait = no
    		user = root
    		group = root
    		groups = yes
    		server = /usr/bin/clustercheck
   		type = UNLISTED
    		per_source = UNLIMITED
    		log_on_success = 
    		log_on_failure = HOST
    		flags = REUSE
	
	}

----

	# systemctl start xinetd
	# systemctl enable xinetd

----

	# mysql -u root -p
	# check garela running
	# show status like 'wsrep_%'; 

	# GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED by 'password' WITH GRANT OPTION;


===============================================================================================================================================================

INSTALL RABBITMQ cluster:

	# yum -y install rabbitmq-server

if firewalld on

	# firewall-cmd --add-port={4369/tcp,25672/tcp} --permanent 
	# firewall-cmd --add-port=15672/tcp --permanent 
	# irewall-cmd --permanent --add-port=5672/tcp
	# firewall-cmd --reload

only on controller1 first; the start stop service rabbitmq;

	# systemctl start rabbitmq-server
	# systemctl stop rabbitmq-server

All controller node;

	#vim /etc/rabbitmq/rabbitmq-env.conf

		NODE_IP_ADDRESS=$(ip addr | grep 'enp1s0:' -A2 | tail -n1 | awk '{print $2}' | awk -F '/' '{print $1}')
		~
		~
		:wq
		
------		

	# scp /var/lib/rabbitmq/.erlang.cookie root@controller2:/var/lib/rabbitmq/
	# scp /var/lib/rabbitmq/.erlang.cookie root@controller3:/var/lib/rabbitmq/

	# scp /etc/rabbitmq/rabbitmq-env.conf root@controller2:/etc/rabbitmq/
	# scp /etc/rabbitmq/rabbitmq-env.conf root@controller3:/etc/rabbitmq/


di rubah owner rabbitmq  controller2 dan controller3 /var/lib/rabbitmq/.erlang.cookie:

	# chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie (di setiap node)

----

	# pcs resource create rabbitmq-server systemd:rabbitmq-server op monitor start-delay=20s clone
	# pcs status

(khusus node controller2 & controller3)

	# rabbitmqctl stop_app
	# rabbitmqctl join_cluster rabbit@controller1
	# rabbitmqctl start_app


monitor rabbitmq via http:
all node

	# rabbitmq-plugins enable rabbitmq_management

download rabbit admin di http:/controller1:15672/cli/index.html

	# cp rabbitmqadmin /usr/local/bin/
	# chmod +x /usr/local/bin/rabbitmqadmin
	# rabbitmqctl list_users 

install phyton36:

	# dnf module -y install python36
	# python3 -V 
	# echo -e "import sys\nprint(sys.version)" > python3_test.py 
	# python3 python3_test.py
	# alternatives --config python (pilih 2 phyton 3)
	# python -V 

only node controller1:

	# rabbitmqadmin declare queue name=shared_queue 
	# rabbitmqctl set_policy ha-policy "shared_queue" '{"ha-mode":"all"}'
	# rabbitmqadmin list queues name node policy slave_nodes state synchronised_slave_nodes 
	# rabbitmqctl add_user user-rabbit password 
	# rabbitmqctl list_users 
	# rabbitmqctl set_user_tags user-rabbit administrator 


	# rabbitmqctl add_user openstack password   
	# rabbitmqctl set_permissions openstack ".*" ".*" ".*" 


link browser rabbitmq http://10.19.33.15:15672/
login with credential
user = user-rabbit
password = password

=================================================================================================================================================================

Install and enable service pacemaker memcache

All node config memcache:

	# yum install memcached

	# vi /etc/sysconfig/memcached  (listen all)

		OPTIONS="-l 0.0.0.0,::"

if firewalld on

	# firewall-cmd --add-service=memcache --permanent
	# firewall-cmd --reload

-----

	# pcs resource create memcached systemd:memcached clone interleave=true



=================================================================================================================================================================
configuration keystone on controller1, contoller2 & controller3

login mysql

	# mysql -u 10.19.33.15 -u root -p
	# create database keystone;
	# grant all privileges on keystone.* to keystone@'localhost' identified by 'password';
	# grant all privileges on keystone.* to keystone@'%' identified by 'password';
	#flush privileges; 
	# exit

install packet keystone on all node

	# yum install epel-release
	# dnf --enablerepo=centos-openstack-ussuri,epel,PowerTools -y install openstack-keystone python3-openstackclient httpd mod_ssl python3-mod_wsgi python3-oauth2client


generate a keystone service token

	# openssl rand -hex 10 > ~/keystone_service_token
	# scp ~/keystone_service_token controller2:~/
	# scp ~/keystone_service_token controller3:~/

if  firewalld on;

	# firewall-cmd --add-port=5000/tcp --permanent 
	# firewall-cmd --add-port=35357/tcp --permanent
	# firewall-cmd --add-servic=http --permanent
	# firewall-cmd --add-service=https --permanent
	# firewall-cmd --reload

config /etc/keystone/keystone.conf
line 14:

	admin_token = /root/keystone_service_token

line 267

	transport_url = rabbit://openstack:password@10.19.33.15

line 429
	
	memcache_servers = "10.19.33.11:11211,10.19.33.12:11211,10.19.33.13"

line 573

	connection = mysql+pymysql://keystone:password@10.19.33.15/keystone

line 2446

	provider = fernet

line 1834

	rabbit_ha_queues = true
	~
	:wq

controller1:

	# scp /etc/keystone/keystone.conf root@10.19.33.12:/etc/keystone/
	# scp /etc/keystone/keystone.conf root@10.19.33.13:/etc/keystone/

generate keystone KPI and sync the database:

	# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone 
	# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

	# scp -r /etc/keystone/credential-keys root@10.19.33.12:/etc/keystone/
	# scp -r /etc/keystone/credential-keys root@10.19.33.13:/etc/keystone/
	# scp -r /etc/keystone/fernet-keys root@10.19.33.13:/etc/keystone/
	# scp -r /etc/keystone/fernet-keys root@10.19.33.12:/etc/keystone/

controller2 & controller3:

	# chown -R keystone:keystone /etc/keystone/credential-keys
	# chown -R keystone:keystone /etc/keystone/fernet-keys
	# restorecon -Rv /etc/keystone/credential-keys
	# restorecon -Rv /etc/keystone/

controller1:

	# su -s /bin/bash keystone -c "keystone-manage db_sync"

config bootstrap keystone:

	# keystone-manage bootstrap --bootstrap-password adminpassword \
	  --bootstrap-admin-url http://10.19.33.14:5000/v3/ \
	  --bootstrap-internal-url http://10.19.33.14:5000/v3/ \
	  --bootstrap-public-url http://10.19.33.14:5000/v3/ \
	  --bootstrap-region-id RegionOne  

Selinux Enable all node:setsebool -P httpd_use_openstack on

	# setsebool -P httpd_use_openstack on
	# setsebool -P httpd_can_network_connect on
	# setsebool -P httpd_can_network_connect_db on


	# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/ 


running service controller1, controller2 dan controller3

	# systemctl enable --noe httpd


Create keystone credential file:

	#vi ~/keystonerc

	export OS_PROJECT_DOMAIN_NAME=default
	export OS_USER_DOMAIN_NAME=default
	export OS_PROJECT_NAME=admin
	export OS_USERNAME=admin
	export OS_PASSWORD=adminpassword
	export OS_AUTH_URL=http://10.19.33.14:5000/v3
	export OS_IDENTITY_API_VERSION=3
	export OS_IMAGE_API_VERSION=2
	export PS1='[\u@\h \W(keystone)]\$ '
	export OS_VOLUME_API_VERSION=3
	~
	~

	:wq


	# chmod 600 ~/keystonerc

----

	# openstack project create --domain default --description "Service Project" service 
	# openstack project list 

=================================================================================================================================================================

Glance instaltation

All node controller

	# dnf --enablerepo=centos-openstack-ussuri,PowerTools,epel -y install openstack-glance 

if firewall on;

	# firewall-cmd --add-port=9292/tcp --permanent
	# firewall-cmd --reload 

	# setsebool -P glance_api_can_network on 


controller1:

create [glance] user in [service] project;

	# source ~/keystonerc
	# openstack user create --domain default --project service --password servicepassword glance

add [glance] user in [admin] role;
	
	# openstack role add --project service --user glance admin

create service entry for [glance];

	# openstack service create --name glance --description "OpenStack Image service" image

create endpoint for [glance] (public);

	# openstack endpoint create --region RegionOne image public http://10.19.33.14:9292 

create endpoint for [glance] (internal);

	# openstack endpoint create --region RegionOne image internal http://10.19.33.14:9292

create endpoint for [glance] (admin);

	# openstack endpoint create --region RegionOne image admin http://10.19.33.14:9292

create database Glance;

	#  mysql -u root -p 

	MariaDB [(none)]> create database glance;
	MariaDB [(none)]> grant all privileges on glance.* to glance@'localhost' identified by 'password'; 
	MariaDB [(none)]> grant all privileges on glance.* to glance@'%' identified by 'password'; 
	MariaDB [(none)]> flush privileges;
	MariaDB [(none)]> exit

using nfs backend 

	on all node mountinfg to nfs filesystem
	# mkdir -p /backend/glance
	# vim /etc/fstab
	...
	...

  	10.19.33.22:/backend/glance	/backend/glance	nfs	_netdev,defaults	0 0
	~
	:wq
---
	# mount -av

	# vim /etc/glance/glance-api.conf

	[DEFAULT]
	bind_host = 0.0.0.0
	transport_url = rabbit://openstack:password@10.19.33.15

	[glance_store]
	stores = file,http
	default_store = file
	filesystem_store_datadir = /backend/glance

	[database]
	connection = mysql+pymysql://glance:password@10.19.33.15/glance

	[keystone_authtoken]
	www_authenticate_uri = http://10.19.33.14:5000
	auth_url = http://10.19.33.14:5000
	memcached_servers = "10.19.33.11:11211,10.19.33.12:11211,10.19.33.13:11211"
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = glance
	password = servicepassword


	[oslo_messaging_rabbit]
	rabbit_ha_queues = true
	rabbit_hosts = 10.19.33.11,10.19.33.12,10.19.33.13

	[paste_deploy]	
	flavor = keystone

	~
	:wq

----

	# scp glance-api.conf root@10.19.33.12:/etc/glance/glance-api.conf
	# scp glance-api.conf root@10.19.33.13:/etc/glance/glance-api.conf


	# su -s /bin/bash glance -c "glance-manage db_sync" 

	# pcs resource create openstack-glance-api systemd:openstack-glance-api clone interleave=true


=================================================================================================================================================================

install and config cinder-api and backend storage nfs


	# dnf --enablerepo=centos-openstack-ussuri,PowerTools,epel -y install openstack-cinder 

	# dnf --enablerepo=centos-openstack-ussuri -y install openstack-selinux  

	# setsebool -P virt_use_nfs on

if firewall on

	# firewall-cmd --add-port=8776/tcp --permanent
	# firewall-cmd --reload


create database cinder:


	# mysql -u root -p
	MariaDB [(none)]> create database cinder; 
	MariaDB [(none)]> grant all privileges on cinder.* to cinder@'localhost' identified by 'password';
	MariaDB [(none)]> grant all privileges on cinder.* to cinder@'%' identified by 'password'; 
	MariaDB [(none)]> flush privileges; 
	MariaDB [(none)]> exit 


create [cinder] user in [service] project;

	# openstack user create --domain default --project service --password servicepassword cinder

add [cinder] user in [admin] role;
openstack role add --project service --user cinder admin 

create service entry for [cinder];

	openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3 

define Cinder API Node;
create endpoint for [cinder] (public);

	# openstack endpoint create --region RegionOne volumev3 public http://10.19.33.14:8776/v3/%\(tenant_id\)s

create endpoint for [cinder] (internal);

	# openstack endpoint create --region RegionOne volumev3 internal http://10.19.33.14:8776/v3/%\(tenant_id\)s 

create endpoint for [cinder] (admin)

	# openstack endpoint create --region RegionOne volumev3 admin http://10.19.33.14:8776/v3/%\(tenant_id\)s

configutre cinder.conf;

	# vim /etc/cinder/cinder.conf 


	[DEFAULT]
	my_ip = 10.19.3.14
	log_dir = /var/log/cinder
	state_path = /var/lib/cinder
	auth_strategy = keystone
	transport_url = rabbit://openstack:password@10.19.33.15
	glance_api_servers = http://10.19.33.14:9292
	enable_v3_api = True
	#enable_v2_api = False
	enabled_backends = nfs 


	# config cinder-backup (optional)
	backup_driver = cinder.backup.drivers.nfs.NFSBackupDriver
	backup_mount_point_base = $state_path/backup_nfs
	backup_share = 10.19.33.22:/backend/cinder-backup

	[database]
	connection = mysql+pymysql://cinder:password@10.19.33.15/cinder
	#max_retries = -1

	[keystone_authtoken]
	www_authenticate_uri = http://10.19.33.14:5000
	auth_url = http://10.19.33.14:5000
	memcached_servers = "10.19.33.11:11211,10.19.33.12:11211,10.19.33.13:112211"
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = cinder
	password = servicepassword

	[oslo_messaging_rabbit]
	rabbit_ha_queues = true

	[oslo_concurrency]
	lock_path = $state_path/tmp

	# line to the end 
	[nfs]
	volume_driver = cinder.volume.drivers.nfs.NfsDriver
	nfs_shares_config = /etc/cinder/nfs_shares
	nfs_mount_point_base = $state_path/mnt
	~
	~
	:wq

create new : specify NFS shared directory;

	# vi /etc/cinder/nfs_shares
	10.19.33.22:/backend/cinder
	~

	:wq
---

	# su -s /bin/bash cinder -c "cinder-manage db sync" 

running service cinder-api, cinder-schaduler & cinder-volume

	# pcs resource create cinder-api systemd:openstack-cinder-api clone interleave=true
	# pcs resource create cinder-scheduler systemd:openstack-cinder-scheduler clone interleave=true
	# pcs resource create cinder-volume systemd:openstack-cinder-volume clone interleave=true
	# pcs resource create cinder-backup systemd:openstack-cinder-backup clone interleave=true

	# pcs constraint order start cinder-api-clone then cinder-scheduler-clone
	# pcs constraint colocation add cinder-scheduler-clone with cinder-api-clone
	# pcs constraint order start cinder-scheduler-clone then cinder-volume-clone
	# pcs constraint colocation add cinder-volume with cinder-scheduler-clone
	# pcs constraint order start cinder-scheduler-clone then cinder-backup-clone
	# pcs constraint colocation add cinder-backup-clone with cinder-scheduler-clone



=================================================================================================================================================================

install & configure NOVA compute;


	# dnf --enablerepo=centos-openstack-ussuri,PowerTools,epel -y install openstack-nova openstack-placement-api

	# dnf --enablerepo=centos-openstack-ussuri -y install openstack-selinux 


if firewalld on;

	# firewall-cmd --add-port={6080/tcp,6081/tcp,6082/tcp,8774/tcp,8775/tcp,8778/tcp} --permanent

	# firewall-cmd --reload

	# semanage port -a -t http_port_t -p tcp 8778 
 

create [nova] user in [service] project;

	# openstack user create --domain default --project service --password servicepassword nova

add [nova] user in [admin];

	# openstack role add --project service --user nova admin 

create [placement] user in [service] project

	# openstack user create --domain default --project service --password servicepassword placement

add [placement] user in [admin] role

	# openstack role add --project service --user placement admin 

create service entry for [nova];

	# openstack service create --name nova --description "OpenStack Compute service" compute 

create service entry for [placement]

	# openstack service create --name placement --description "OpenStack Compute Placement service" placement 

define Nova Host;
create endpoint for [nova] (public);

	# openstack endpoint create --region RegionOne compute public http://10.19.33.14:8774/v2.1/%\(tenant_id\)s 

create endpoint for [nova] (internal);

	# openstack endpoint create --region RegionOne compute internal http://10.19.33.14:8774/v2.1/%\(tenant_id\)s

create endpoint for [nova] (admin);

	# openstack endpoint create --region RegionOne compute admin http://10.19.33.14:8774/v2.1/%\(tenant_id\)s

create endpoint for [placement] (public);

	# openstack endpoint create --region RegionOne placement public http://10.19.33.14:8778

create endpoint for [placement] (internal);

	# openstack endpoint create --region RegionOne placement internal http://10.19.33.14:8778

create endpoint for [placement] (admin);

	# openstack endpoint create --region RegionOne placement admin http://10.19.33.14:8778


Add a User and Database on MariaDB for Nova.

	# mysql -u root -p 
	MariaDB [(none)]> create database nova; 
	MariaDB [(none)]> grant all privileges on nova.* to nova@'localhost' identified by 'password';
	MariaDB [(none)]> grant all privileges on nova.* to nova@'%' identified by 'password'; 
	MariaDB [(none)]> create database nova_api;
	MariaDB [(none)]> grant all privileges on nova_api.* to nova@'localhost' identified by 'password';
	MariaDB [(none)]> grant all privileges on nova_api.* to nova@'%' identified by 'password';
	MariaDB [(none)]> create database nova_cell0;
	MariaDB [(none)]> grant all privileges on nova_cell0.* to nova@'localhost' identified by 'password'; 
	MariaDB [(none)]> grant all privileges on nova_cell0.* to nova@'%' identified by 'password';
	MariaDB [(none)]> create database placement; 
	MariaDB [(none)]> grant all privileges on placement.* to placement@'localhost' identified by 'password';
	MariaDB [(none)]> grant all privileges on placement.* to placement@'%' identified by 'password';
	MariaDB [(none)]> flush privileges;
	MariaDB [(none)]> exit

----

	# cp /etc/nova/nova.conf /root/backup-config/ 

config nova.conf;

	vim /etc/nova/nova.conf 

	[DEFAULT]
	# define own IP address
	my_ip = 10.19.33.14
	state_path = /var/lib/nova
	enabled_apis = osapi_compute,metadata
	log_dir = /var/log/nova
	# RabbitMQ connection info
	transport_url = rabbit://openstack:password@10.19.33.15

	[api]
	auth_strategy = keystone

	# Glance connection info
	[glance]
	api_servers = http://10.19.33.14:9292

	[oslo_concurrency]
	lock_path = $state_path/tmp

	[oslo_messaging_rabbit]
	rabbit_ha_queues = true

	#MariaDB connection info
	[api_database]	
	connection = mysql+pymysql://nova:password@10.19.33.15/nova_api

	[database]
	connection = mysql+pymysql://nova:password@10.19.33.15/nova


	# Keystone auth info
	[keystone_authtoken]
	www_authenticate_uri = http://10.19.33.14:5000
	auth_url = http://10.19.33.14:5000
	memcached_servers = "10.19.33.11:11211,10.19.33.12:11211,10.19.33.13:11211"
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = nova
	password = servicepassword


	[placement]
	auth_url = http://10.19.33.14:5000
	os_region_name = RegionOne
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = placement
	password = servicepassword

	[wsgi]
	api_paste_config = /etc/nova/api-paste.ini
	~
	~

	:wq
----

config nova-placement;

	# vim /etc/placement/placement.conf 

	[DEFAULT]
	debug = false

	[api]
	auth_strategy = keystone

	[keystone_authtoken]
	www_authenticate_uri = http://10.19.33.14:5000
	auth_url = http://10.19.33.14:5000
	memcached_servers = "10.19.33.11:11211,10.19.33.12:11211,10.19.33.13:11211"
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = placement
	password = servicepassword

	[placement_database]
	connection = mysql+pymysql://placement:password@10.19.33.15/placement
	~
	~
	:wq
	---

config http placement-api

	# vim /etc/httpd/conf.d/00-placement-api.conf

add near line 15;

	<Directory /usr/bin>
    		Require all granted
  	</Directory>

	</VirtualHost>
	~
	:wq

Add Data into Database and start Nova services;

	# su -s /bin/bash placement -c "placement-manage db sync" 
	# su -s /bin/bash nova -c "nova-manage api_db sync" 
	# su -s /bin/bash nova -c "nova-manage cell_v2 map_cell0"
	# su -s /bin/bash nova -c "nova-manage db sync" 
	# su -s /bin/bash nova -c "nova-manage cell_v2 create_cell --name cell1" 

	# su -s /bin/bash nova -c "nova-manage cell_v2 list_cells"


	# pcs resource create nova-api systemd:openstack-nova-api clone interleave=true
	# pcs resource create nova-novncproxy systemd:openstack-nova-novncproxy clone interleave=true
	# pcs resource create nova-scheduler systemd:openstack-nova-scheduler clone interleave=true
	# pcs resource create nova-conductor systemd:openstack-nova-conductor clone interleave=true

	# pcs constraint order start nova-api-clone then nova-scheduler-clone
	# pcs constraint colocation add cinder-scheduler-clone with cinder-api-clone
	# pcs constraint order start nova-scheduler-clone  then nova-conductor
	# pcs constraint colocation add nova-conductor-clone with nova-scheduler-clone


--------------------------------------------------------------------------------------------------

add compute node / nova-compute;

if firewall on

	# firewall-cmd --add-port=5900-5999/tcp --permanent
	# firewall-cmd --reload


	# dnf --enablerepo=centos-openstack-ussuri,PowerTools,epel -y install openstack-nova-compute 
	# dnf --enablerepo=centos-openstack-ussuri -y install openstack-selinux 

	# setsebool -P neutron_can_network on

	# setsebool -P daemons_enable_cluster_mode on 

	# vim /etc/nova/nova.conf

		[DEFAULT]
		# define own IP address
		my_ip = 10.19.33.9
		state_path = /var/lib/nova
		enabled_apis = osapi_compute,metadata
		log_dir = /var/log/nova
		# RabbitMQ connection info
		transport_url = rabbit://openstack:password@10.19.33.15

		[api]
		auth_strategy = keystone


		[vnc]
		enabled = True
		server_listen = 0.0.0.0
		server_proxyclient_address = $my_ip
		novncproxy_base_url = http://10.19.33.14:6080/vnc_auto.html 

		[glance]
		api_servers = http://10.19.33.14:9292

		[oslo_concurrency]
		lock_path = $state_path/tmp

		[oslo_messaging_rabbit]
		rabbit_ha_queues = true

		# Keystone auth info
		[keystone_authtoken]
		www_authenticate_uri = http://10.19.33.14:5000
		auth_url = http://10.19.33.14:5000
		memcached_servers = "10.19.33.11:11211,10.19.33.12:11211,10.19.33.13:11211"
		auth_type = password
		project_domain_name = default
		user_domain_name = default
		project_name = service
		username = nova
		password = servicepassword

		[placement]
		auth_url = http://10.19.33.14:5000
		os_region_name = RegionOne
		auth_type = password
		project_domain_name = default
		user_domain_name = default
		project_name = service
		username = placement
		password = servicepassword

		[wsgi]
		api_paste_config = /etc/nova/api-paste.ini
		~
		:wq

		# systemctl enable --now openstack-nova-compute 



=================================================================================================================================================================

install and configure neutron controller

Better serivce firewall turn off;

	# dnf --enablerepo=centos-openstack-ussuri,PowerTools,epel -y install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch 


	# firewall-cmd --add-port=9696/tcp --permanent

	# firewall-cmd --reload 


	# setsebool -P neutron_can_network on

	# setsebool -P haproxy_connect_any on

	# setsebool -P daemons_enable_cluster_mode on 


create [neutron] user in [service] project;

	# openstack user create --domain default --project service --password servicepassword neutron 

add [neutron] user in [admin] role;

	# openstack role add --project service --user neutron admin 

create service entry for [neutron];

	# openstack service create --name neutron --description "OpenStack Networking service" network 

define Neutron API Host;
create endpoint for [neutron] (public);

	# openstack endpoint create --region RegionOne network public http://10.19.33.14:9696

create endpoint for [neutron] (internal);

	# openstack endpoint create --region RegionOne network internal http://10.19.33.14:9696

create endpoint for [neutron] (admin);

	# openstack endpoint create --region RegionOne network admin http://10.19.33.14:9696


Add a User and Database on MariaDB for Neutron.

	# mysql -u root -p 

	MariaDB [(none)]> create database neutron_ml2; 
	MariaDB [(none)]> grant all privileges on neutron_ml2.* to neutron@'localhost' identified by 'password';
	MariaDB [(none)]> grant all privileges on neutron_ml2.* to neutron@'%' identified by 'password'; 
	MariaDB [(none)]> flush privileges; 
	MariaDB [(none)]> exit 


config file neutron.conf;

	# cp /etc/neutron/neutron.conf /root/backup-config/
	# vim  /etc/neutron/neutron.conf

		[DEFAULT]
		core_plugin = ml2
		service_plugins = router
		auth_strategy = keystone
		state_path = /var/lib/neutron
		dhcp_agent_notification = True
		allow_overlapping_ips = True
		notify_nova_on_port_status_changes = True
		notify_nova_on_port_data_changes = True
		# RabbitMQ connection info
		transport_url = rabbit://openstack:password@10.19.33.15


		# Keystone auth info
		[keystone_authtoken]
		www_authenticate_uri = http://10.19.33.14:5000
		auth_url = http://10.19.33.14:5000
		memcached_servers = "10.19.33.11:11211,10.19.33.12:11211,10.19.33.13:11211"
		auth_type = password
		project_domain_name = default
		user_domain_name = default
		project_name = service
		username = neutron
		password = servicepassword


		# MariaDB connection info
		[database]
		connection = mysql+pymysql://neutron:password@10.19.33.15/neutron_ml2

		# Nova connection info
		[nova]
		auth_url = http://10.19.33.14:5000
		auth_type = password
		project_domain_name = default
		user_domain_name = default
		region_name = RegionOne
		project_name = service
		username = nova
		password = servicepassword

		[oslo_concurrency]
		lock_path = $state_path/tmp

		[oslo_messaging_rabbit]
		rabbit_ha_queues = true
		~
		:wq
----

		# chmod 640 /etc/neutron/neutron.conf 
		# chgrp neutron /etc/neutron/neutron.conf 


self service network neutron on all controller node

config l3_agent.ini;

	# vim /etc/neutron/l3_agent.ini

	# line 2: add
	interface_driver = openvswitch 
	~

	:wq
-----

config neutron dhcp_agent.ini;

	# vim /etc/neutron/dhcp_agent.ini 
	# line 2: add

	interface_driver = openvswitch
	dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
	enable_isolated_metadata = true 
	~

	:wq
-----

config neutron metadata_agent.ini;

	# vim /etc/neutron/metadata_agent.ini 

 	# line 2: add

	 # specify Nova API server

	nova_metadata_host = 10.19.33.14
	# specify any secret key you like

	metadata_proxy_shared_secret = metadata_secret
	# line 212: uncomment and specify Memcache server

	memcache_servers = "10.19.33.11:11211,10.19.33.12:11211,10.19.33.13:11211"

	~
	:wq
-----

config neutron plugins ml2_conf.ini;

	# vim /etc/neutron/plugins/ml2/ml2_conf.ini 

	 # add to the end
 
 	# OK with no value for [tenant_network_types] now (set later if need)

	[ml2]
	type_drivers = flat,vlan,gre,vxlan
	tenant_network_types = vxlan
	mechanism_drivers = openvswitch
	extension_drivers = port_security

	[ml2_type_flat]
	flat_networks = physnet1

	[ml2_type_vxlan]
	vni_ranges = 1:1000
	~
	~

	:wq
----

config plungin ml2 openvswitch_agent.ini;

	# vim /etc/neutron/plugins/ml2/openvswitch_agent.ini 

	# add to the end

	[securitygroup]
	firewall_driver = openvswitch
	enable_security_group = true
	enable_ipset = true

	# add to the end

	[ovs]
	bridge_mappings = physnet1:br-enp2s0
	local_ip = 10.19.33.11
	# add to the end

	[agent]
	tunnel_types = vxlan
	prevent_arp_spoofing = True
	~
	~

	:wq
----

config neutron on nova config;

	# vim /etc/nova/nova.conf 

 	# add follows into [DEFAULT] section

	use_neutron = True
	linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
	firewall_driver = nova.virt.firewall.NoopFirewallDriver
	vif_plugging_is_fatal = True
	vif_plugging_timeout = 300

	 # add follows to the end : Neutron auth info
 	# the value of [metadata_proxy_shared_secret] is the same with the one in [metadata_agent.ini]

	[neutron]
	auth_url = http://10.19.33.14:5000
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	region_name = RegionOne
	project_name = service
	username = neutron
	password = servicepassword
	service_metadata_proxy = True
	metadata_proxy_shared_secret = metadata_secret
	~
	~

	:wq
	
----


start and enable service openvswicth on all controller node;

	# systemctl enable --now openvswitch.service


	# ovs-vsctl add-br br-int 

	# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini 

	# su -s /bin/bash neutron -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head" 

	# pcs resource create neutron-server systemd:neutron-server clone interleave=true 
	# pcs resource create neutron-dhcp-agent systemd:neutron-dhcp-agent clone interleave=true 
	# pcs resource create neutron-l3-agent systemd:neutron-l3-agent clone interleave=true 
	# pcs resource create neutron-metadata-agent systemd:neutron-metadata-agent clone interleave=true 
	# pcs resource create neutron-openvswitch-agent systemd:neutron-openvswitch-agent clone interleave=true

	# openstack network agent list 

------------------------------------
 	
Configure Networking for Virtual Machine Instances;

add bridge

	# ovs-vsctl add-br br-int
	# ovs-vsctl add-br br-enp2s0
	# ovs-vsctl add-port br-enp2s0 enp2s0


add network flat and vxlan;

	# vim /etc/neutron/plugins/ml2/ml2_conf.ini  
 	# add to the end

	[ml2_type_flat]
	flat_networks = physnet1

	:wq
-----

	# vim /etc/neutron/plugins/ml2/openvswitch_agent.ini

	# add to the end

	[ovs]
	bridge_mappings = physnet1:br-enp2s0

	:wq
-----


=============================================================================================================

install neutron om compute-node;

bettet firewall turn off


	# dnf --enablerepo=centos-openstack-ussuri,PowerTools,epel -y install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch

	# dnf --enablerepo=centos-openstack-ussuri -y install openstack-selinux 

	# setsebool -P neutron_can_network on

	# setsebool -P daemons_enable_cluster_mode on 

	# systemctl enable --now openvswitch 
	
	# ovs-vsctl add-br br-int

	# ovs-vsctl add-br br-enp2s0

	# ovs-vsctl add-port br-enp2s0 enp2s0


	# ovs-vsctl add-port br-enp2s0 enp2s0

	# cp /etc/neutron/neutron.conf /root/backup-config/ 
	
-----

	# vim /etc/neutron/neutron.conf


	[DEFAULT]
	core_plugin = ml2
	service_plugins = router
	auth_strategy = keystone
	state_path = /var/lib/neutron
	allow_overlapping_ips = True
	# RabbitMQ connection info
	transport_url = rabbit://openstack:password@10.19.33.15

	# Keystone auth info
	[keystone_authtoken]
	www_authenticate_uri = http://10.19.33.14:5000
	auth_url = http://10.19.33.14:5000
	memcached_servers = "10.19.33.11:11211,10.19.3.12:11211,10.19.33.13:11211"
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = neutron
	password = servicepassword

	[oslo_concurrency]
	lock_path = $state_path/lock

	[oslo_messaging_rabbit]
	rabbit_ha_queues = true
	~
	~
	:wq
----

	# chmod 640 /etc/neutron/neutron.conf 
	# chgrp neutron /etc/neutron/neutron.conf

self service neutron on compute node;


	# vim /etc/neutron/plugins/ml2/ml2_conf.ini 
 
	 # add to the end

 	# OK with no value for [tenant_network_types] now (set later if need)

	[ml2]
	type_drivers = flat,vlan,gre,vxlan
	tenant_network_types = vxlan
	mechanism_drivers = openvswitch
	extension_drivers = port_security

	[ml2_type_flat]
	flat_networks = physnet1

	[ml2_type_vxlan]
	vni_ranges = 1:1000
	~
	~

	:wq
----

	# vim /etc/neutron/plugins/ml2/openvswitch_agent.ini 

 	# add to the end

	[securitygroup]
	firewall_driver = openvswitch
	enable_security_group = true
	enable_ipset = true

	[agent]
	tunnel_types = vxlan
	prevent_arp_spoofing = True


	[ovs]
	bridge_mappings = physnet1:br-enp2s0
	local_ip = 10.19.33.9
	~
	~
	:wq
----

	# vim /etc/nova/nova.conf 

  	# add follows into [DEFAULT] section

	use_neutron = True
	linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
	firewall_driver = nova.virt.firewall.NoopFirewallDriver
	vif_plugging_is_fatal = True
	vif_plugging_timeout = 300

	 # add follows to the end: Neutron auth info
 	# the value of [metadata_proxy_shared_secret] is the same with the one in [metadata_agent.ini]

	[neutron]
	auth_url = http://10.19.33.14:5000
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	region_name = RegionOne
	project_name = service
	username = neutron
	password = servicepassword
	service_metadata_proxy = True
	metadata_proxy_shared_secret = metadata_secret

	:wq
-----

	# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini 

	# systemctl enable --now openvswitch  

	# systemctl restart openstack-nova-compute 

	# systemctl enable --now neutron-openvswitch-agent 


running sysnd db nova on controller node;

	# su -s /bin/bash nova -c "nova-manage cell_v2 discover_hosts" 


=====================================================================================

Install Horizon. 

	# dnf --enablerepo=centos-openstack-ussuri,PowerTools,epel -y install openstack-dashboard 

	# firewall-cmd --add-service={http,https} --permanent

	# firewall-cmd --reload

	# setsebool -P httpd_can_network_connect on 

config openstack dashboard

	# vi /etc/openstack-dashboard/local_settings 

  	# line 39: set Hosts you allow to access
 	 # to specify wildcard ['*'], allow all

	ALLOWED_HOSTS = ['*', ] 
    
 	# line 94-99: uncomment and specify Memcache server Host

	CACHES = {
    		'default': {
       		 'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        		'LOCATION': '10.19.33.11:11211,10.19.33.12:11211,10.19.33.13:11211',
    		},
	}


  	 # line 105: add

	SESSION_ENGINE = "django.contrib.sessions.backends.cache" 

   	# line 118: set Openstack Host
   	# line 119: comment out and add a line to specify URL of Keystone Host

	OPENSTACK_HOST = "10.19.33.14"
	#OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
	OPENSTACK_KEYSTONE_URL = "http://10.19.33.14:5000/v3"

   	# line 123: set your timezone

	TIME_ZONE = "Asia/Jakarta" 

  	 # add to the end

	WEBROOT = '/dashboard/'
	LOGIN_URL = '/dashboard/auth/login/'
	LOGOUT_URL = '/dashboard/auth/logout/'
	LOGIN_REDIRECT_URL = '/dashboard/'
	OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
	OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
	OPENSTACK_API_VERSIONS = {
   	 "identity": 3,
    	"volume": 3,
    	"compute": 2,
	}
	~
	!
	:wq
-----

	# vim /etc/httpd/conf.d/openstack-dashboard.conf 

  	# line 4: add

	WSGIDaemonProcess dashboard
	WSGIProcessGroup dashboard
	WSGISocketPrefix run/wsgi
	WSGIApplicationGroup %{GLOBAL}

	:wq
----

restart service http

	# systemctl restart httpd


open horizon dashboard with link http:/10.19.33.15:/dashboard











