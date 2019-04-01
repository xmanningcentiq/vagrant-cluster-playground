# Pacemaker Cluster Playground

## Purpose

A playground for performing operations against pacemaker clustering and testing
failover behavior of resources. The idea is that you can break, throw away and
rebuild these clusters quickly.

There are two flavours of cluster built:

  1. CentOS (`pcs`) - which is similar to the Red Hat High Availability Add-on.
  1. openSUSE (`crm`) - which is similar to the SLES High Availability 
     extension.

### Architecture Diagram

#### CentOS

![Cluster Diagram](images/centosCluster.png)

#### openSUSE

![Cluster Diagram](images/openSuseCluster.png)

## DISCLAIMER

:warning: **This is not production grade automation!**

The automation to build these clusters is at best "sketch" grade, it's sole
purpose is creating clusters in the context of Vagrant.

The STONITH devices used for fencing in this cluster are effectively dummies.
Whilst the openSUSE cluster does have an SSH STONITH device, is is only likely
to ever work in a very small set of circumstances.

I have also had very little exposure to PostgreSQL, I jumbled this together 
from a load of online resources of mixed quality. If you run this cluster
as per the automation in production you will likely be subject to hacking or
data loss.

## Requirements

To create this cluster you will need the following resources:

  - Vagrant v2.2.0+
  - Ansible v2.6.5+

## Getting Started

Below are some quick tips to get you started in the cluster playground. For more
in-depth cluster administration refer to one of the following guides:

  - **CentOS**: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_administration/
  - **openSUSE**: https://www.suse.com/documentation/sle-ha-15/book_sleha_guide/data/book_sleha_guide.html

### Spin up the clusters

To start a brand new cluster, run `vagrant up` - this will create both the
CentOS and openSUSE clusters running PostgreSQL.

### Destroy the cluster

To destroy the cluster environments, run `vagrant destroy -f`

### Getting cluster status

You can get the health of the cluster by logging into a target VM with
`vagrant ssh <vmname>` then issuing one of the following commands:

  - `pcs status` (centOS)
  - `crm status` (openSUSE)

**Example output**:

```
vgtsldb01:~ # crm status
Stack: corosync
Current DC: vgtsldb02 (version 1.1.18+20180430.b12c320f5-lp150.1.4-b12c320f5) - partition with quorum
Last updated: Mon Apr  1 14:22:46 2019
Last change: Mon Apr  1 14:22:43 2019 by root via crm_attribute on vgtsldb01

2 nodes configured
5 resources configured

Online: [ vgtsldb01 vgtsldb02 ]

Full list of resources:

 stonith_vgtsldb01      (stonith:external/ssh): Started vgtsldb02
 stonith_vgtsldb02      (stonith:external/ssh): Started vgtsldb01
 virtual_public_ip      (ocf::heartbeat:IPaddr2):       Started vgtsldb01
 Master/Slave Set: master_slave_set_pgsql [pgsqld]
     Masters: [ vgtsldb01 ]
     Slaves: [ vgtsldb02 ]

```

### Causing a failover

You can cause a "soft" failover by powering off one node in the cluster. This
can be done by running `poweroff` on the master node.

Run `pcs status`/`crm status` on the other node to get the current cluster
state.

You can also force an UNCLEAN failure of a node by running
`/root/triggerKernelPanic.sh` - this will cause a node to kernel panic and
leave the cluster.

### Recovering a node

Should you cause a node to fail, on reboot `pacemaker` will not be running.

Start this with `systemctl start pacemaker`.

### Cleaning up resources

If a resource has failed, you can attempt to clean them up with:

  - `pcs resource cleanup <resourcename>` (CentOS)
  - `crm resource cleanup <resourcename>` (openSUSE)

The cluster will then attempt to restart the resource.

**Example of a failed resource**:

```
Stack: corosync
Current DC: vgtsldb02 (version 1.1.18+20180430.b12c320f5-lp150.1.4-b12c320f5) - partition with quorum
Last updated: Mon Apr  1 14:33:30 2019
Last change: Mon Apr  1 14:32:26 2019 by root via crm_attribute on vgtsldb02

2 nodes configured
5 resources configured

Node vgtsldb01: UNCLEAN (online)
Online: [ vgtsldb02 ]

Full list of resources:

stonith_vgtsldb01       (stonith:external/ssh): Started vgtsldb02
stonith_vgtsldb02       (stonith:external/ssh): Started vgtsldb01
virtual_public_ip       (ocf::heartbeat:IPaddr2):       Started vgtsldb02
 Master/Slave Set: master_slave_set_pgsql [pgsqld]
     pgsqld     (ocf::heartbeat:pgsqlms):       FAILED vgtsldb01
     Masters: [ vgtsldb02 ]

Failed Actions:
* pgsqld_stop_0 on vgtsldb01 'unknown error' (1): call=25, status=complete, exitreason='Unexpected state for instance "pgsqld" (returned 9)',
    last-rc-change='Mon Apr  1 14:32:54 2019', queued=1ms, exec=136ms
```

If you get the above message repeatedly on clean it is likely that the 
PostgreSQL is in a "crashed" state. Rather than fixing it a script has been
created to quickly recover it as a replica, run the following on the broken 
node: 

  - `/root/resetReplication.sh`
  - CentOS: `pcs resource cleanup pgsqld`
  - openSUSE: `crm resource cleanup pgsqld`

### Check replication status

You can connect to the master database with username `postgres` (no password)
via the appropriate VIP:

| Cluster Name | VIP            |
|--------------|----------------|
| CentOS       | 192.168.61.120 |
| openSUSE     | 192.168.61.125 |

If you're interested in checking that the PostgreSQL replication is working
between nodes, run the following query in either `pgsql` or something like
Sqlectron.

```sql
SELECT * FROM pg_stat_replication;
```

**Example output**:

```
vgtsldb01:~ # psql -U postgres -h 192.168.61.120
psql (10.6, server 10.7)
Type "help" for help.

postgres=# SELECT * FROM pg_stat_replication;

 pid  | usesysid | usename  | application_name |  client_addr   | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state 
------+----------+----------+------------------+----------------+-----------------+-------------+-------------------------------+--------------+-----------+-----------+-----------+-----------+------------+-----------+-----------+------------+---------------+------------
 7203 |       10 | postgres | vgtrhdb02        | 192.168.61.122 |                 |       51404 | 2019-04-01 12:21:29.920447+00 |              | streaming | 0/3000328 | 0/3000328 | 0/3000328 | 0/3000328  |           |           |            |             1 | sync
(1 row)

postgres=# \q
vgtsldb01:~ # 
```
