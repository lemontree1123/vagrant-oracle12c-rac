vagrant-oracle12c-rac
=====================

[Japanese version here](README_ja.md)

Vagrant + Oracle Linux 6.5 + Oracle Database 12cR1 (Enterprise Edition) RAC setup.
OS (+packages, users) setup is fully automated, GI/DB setup is done via silent install (no X).

You need to download Grid Infrastructure, Database binary separately.

Silent install part can be a shell script if you wish, but I prefer to run them manually.

as of 7/7/2014

## Setup

* node1, node2
  * Oracle Linux 6.5 (converted from CentOS6.5)
  * oracle-rdbms-server-12cR1-preinstall
  * Unbreakable Enterprise Kernel
  * Memory: 2GB each
  * Shared Disk: 10GB (uses ASM)

```
192.168.101.11  node1
192.168.101.12  node2
192.168.101.21  node1-vip
192.168.101.22  node2-vip
192.168.101.100 scan
192.168.102.11  node1-priv
192.168.102.12  node2-priv
```

## Download

Download the Grid Infrastructure / Database binary form below.  Unzip to the subdirectory name "grid" and "database".

http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html

go to Linux x86-64 -> "See All"

* into "database" subdirectory
  * linuxamd64_12c_database_1of2.zip
  * linuxamd64_12c_database_2of2.zip

* into "grid" subdirectory
  * linuxamd64_12c_grid_1of2.zip
  * linuxamd64_12c_grid_2of2.zip

## Install OS

If you are behing a proxy, install vagrant-proxyconf.

```
(MacOSX)
$ export http_proxy=proxy:port
$ export https_proxy=proty:port

(Windows)
$ set http_proxy=proxy:port
$ set https_proxy=proxy:port

$ vagrant plugin install vagrant-proxyconf
```

Install VirtualBox plugin.

```
$ vagrant plugin install vagrant-vbguest
```

Clone this repository to the local directory.  Move the "grid" and "database" directories to the same folder.

```
$ git clone https://github.com/yasushiyy/vagrant-oracle12c-rac
$ cd vagrant-oracle12c-rac
```

If you are behind a proxy, add follwing to Vagrantfile:

```
config.proxy.http     = "http://proxy:port"
config.proxy.https    = "http://proxy:port"
config.proxy.no_proxy = "localhost,127.0.0.1"
```

Boot.  This might take a long time.

```
$ vagrant up
```

Restart to use the UEK kernel.

```
$ vagrant reload

$ vagrant ssh node1

[vagrant@node1 ~]$ dmesg | more
Initializing cgroup subsys cpuset
Initializing cgroup subsys cpu
Linux version 2.6.39-400.215.3.el6uek.x86_64 (mockbuild@ca-build44.us.oracle.com) (gcc version 4.4.6 20110731 (Red Hat 4.4.6-3) (GCC) ) #1 SMP Fri Jun 20 00:37:05 PDT 2014
Command line: ro root=/dev/mapper/VolGroup-lv_root rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD rd_LVM_LV=VolGroup/lv_swap SYSFONT=latarcyrheb-sun16 rd_LVM_LV=VolGroup/lv_root  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet numa=off transparent_hugepage=never
```

## Install Clusterware (as grid)

Install Clusterware.

```
[vagrant@node1 ~]$ sudo su - grid

(optional)
[grid@node1 ~]$ /vagrant/grid/runcluvfy.sh stage -pre crsinst -n node1,node2

[grid@node1 ~]$ /vagrant/grid/runInstaller -silent -responseFile /vagrant/grid_install.rsp -ignoreSysPrereqs

the follwing WARNING can be ignored:
[WARNING] [INS-41170] You have chosen not to configure the Grid Infrastructure Management Repository. Not configuring the Grid Infrastructure Management Repository will permanently disable the Cluster Health Monitor, QoS Management, Memory Guard, and Rapid Home Provisioning features. Enabling of these features will require reinstallation of the Grid Infrastructure.
[WARNING] [INS-30011] The SYS password entered does not conform to the Oracle recommended standards.
[WARNING] [INS-30011] The ASMSNMP password entered does not conform to the Oracle recommended standards.
[WARNING] [INS-13014] Target environment does not meet some optional requirements.
  -> INFO: ERROR: [Result.addErrorDescription:607]  PRVF-7530 : Sufficient physical memory is not available on node "node2" [Requi
red physical memory = 4GB (4194304.0KB)]
  -> INFO: ERROR: [Result.addErrorDescription:607]  PRVF-7573 : Sufficient swap size is not available on node "node2" [Required =
2.7501GB (2883732.0KB) ; Found = 927.9922MB (950264.0KB)]
  -> INFO: ERROR: [Result.addErrorDescription:607]  PRVG-11550 : Package "cvuqdisk" is missing on node "node2"

   :
The installation of Oracle Grid Infrastructure 12c was successful.
   :
```

Run root scripts.  When asked for a root password, enter "vagrant".

```
[grid@node1 ~]$ ssh root@node1 /u01/oraInventory/orainstRoot.sh
[grid@node1 ~]$ ssh root@node2 /u01/oraInventory/orainstRoot.sh
[grid@node1 ~]$ ssh root@node1 /u01/12.1.0.1/grid/root.sh
[grid@node1 ~]$ ssh root@node2 /u01/12.1.0.1/grid/root.sh
[grid@node1 ~]$ /u01/12.1.0.1/grid/cfgtoollogs/configToolAllCommands RESPONSE_FILE=/vagrant/grid_install.rsp
```

Check status.

```
[grid@node1 ~]$ crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA.dg
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
ora.LISTENER.lsnr
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
ora.asm
               ONLINE  ONLINE       node1                    Started,STABLE
               ONLINE  ONLINE       node2                    Started,STABLE
ora.net1.network
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
ora.ons
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       node1                    STABLE
ora.cvu
      1        ONLINE  ONLINE       node1                    STABLE
ora.node1.vip
      1        ONLINE  ONLINE       node1                    STABLE
ora.node2.vip
      1        ONLINE  ONLINE       node2                    STABLE
ora.oc4j
      1        OFFLINE OFFLINE                               STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       node1                    STABLE
--------------------------------------------------------------------------------

[grid@node1 ~]$ asmcmd lsdg
State    Type    Rebal  Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
MOUNTED  EXTERN  N         512   4096  1048576     10240     9943                0            9943              0             Y  DATA/
```

## Install DB (as oracle)

Install Database binary.

```
[grid@node1 ~]$ exit

[vagrant@node1 ~]$ sudo su - oracle

[oracle@node1 ~]$ /vagrant/database/runInstaller -silent -ignorePrereq -responseFile /vagrant/db_install.rsp

the follwing WARNING can be ignored:
[WARNING] - My Oracle Support Username/Email Address Not Specified


  :
The installation of Oracle Database 12c was successful.
  :
```

Run root scripts.  When asked for a root password, enter "vagrant".

```
[oracle@node1 ~]$ ssh root@node1 /u01/oracle/product/12.1.0.1/dbhome_1/root.sh
[oracle@node1 ~]$ ssh root@node2 /u01/oracle/product/12.1.0.1/dbhome_1/root.sh
```

Check status.

```
[oracle@node1 ~]$ which sqlplus
/u01/oracle/product/12.1.0.1/dbhome_1/bin/sqlplus

[oracle@node1 ~]$ sqlplus / as sysdba

SQL*Plus: Release 12.1.0.1.0 Production on Mon Jul 7 06:01:08 2014

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to an idle instance.

SQL> exit
Disconnected
```

## Create DB (as oracle)

Create Database.

```
[oracle@node1 ~]$ dbca -silent -createDatabase -responseFile /vagrant/dbca.rsp

Copying database files
1% complete
3% complete
9% complete
15% complete
21% complete
27% complete
30% complete
Creating and starting Oracle instance
32% complete
36% complete
40% complete
44% complete
45% complete
48% complete
50% complete
Creating cluster database views
52% complete
70% complete
Completing Database Creation
73% complete
76% complete
85% complete
94% complete
100% complete
Look at the log file "/u01/oracle/cfgtoollogs/dbca/orcl/orcl.log" for further details.
```

Check status.

```
[oracle@node1 ~]$ /u01/12.1.0.1/grid/bin/crsctl stat res ora.orcl.db -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.orcl.db
      1        ONLINE  ONLINE       node1                    Open,STABLE
      2        ONLINE  ONLINE       node2                    Open,STABLE
--------------------------------------------------------------------------------
```

Check connection.

```
[oracle@node1 ~]$ sqlplus system/oracle@node2-vip:1521/orcl

SQL*Plus: Release 12.1.0.1.0 Production on Mon Jul 7 06:53:49 2014

Copyright (c) 1982, 2013, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.1.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
Advanced Analytics and Real Application Testing options

SQL> select * from dual;

D
-
X

SQL> exit
Disconnected from Oracle Database 12c Enterprise Edition Release 12.1.0.1.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
Advanced Analytics and Real Application Testing options
```

## FYI

VKTM background process consumes extra CPU under Virtualbox, because it executes `gettimeofday()` frequently.
You can execute below to disable this.  But NEVER do this in the production system.

```
(DB and ASM)
SQL> alter system set "_high_priority_processes"='' scope=spfile;
```
