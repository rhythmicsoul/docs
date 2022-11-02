# Troubleshooting MariaDB  Galera Cluster

## Slow Cluster Identification

Execute the following command and ensure both variables are close to 0. The one having the highest values is the slowest node.

``` sql
SELECT * FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME LIKE 'wsrep_flow_control_sent'
OR VARIABLE_NAME LIKE 'wsrep_local_recv_queue_avg';
```

- [wsrep_flow_control_sent](https://galeracluster.com/library/documentation/galera-status-variables.html#wsrep-flow-control-sent) This status variable shows the number of Flow Control pause events sent by the local node since the last status query.
- [wsrep_local_recv_queue_avg](https://galeracluster.com/library/documentation/galera-status-variables.html#wsrep-local-recv-queue-avg) Recv queue length averaged over interval since the last `FLUSH STATUS` command. Values considerably larger than `0.0` mean that the node cannot apply write-sets as fast as they are received and will generate a lot of replication throttling

## Bootstrapping the Cluster Incase of a Crash

### Ensuring safe_to_bootstrap flag

The node should have its mariadb service stopped cleanly before this bootstrapping could be done. Stopping the service cleanly ensures that the safe_to_bootstrap flag in the /var/lib/mysql/grastate.dat is set to 1. 

When the node/service is not shutdown properly or is currently running, the grastate.dat will have the following data/flags.

``` 
# GALERA saved state
version: 2.1
uuid:    505fcbc2-f299-11ec-a3be-4beafe45879f
seqno:   2020642883
safe_to_bootstrap: 0
```

If one needs to forcefully bootstrap the cluster then modify the safe_to_bootstrap flag  to 1 as shown in the snippet below and then instantiate the new cluster.

```
# GALERA saved state
version: 2.1
uuid:    505fcbc2-f299-11ec-a3be-4beafe45879f
seqno:   2020642883
safe_to_bootstrap: 0
```

### Bootstrapping a New Cluster

Execute the following command in the node that could act as the source.

```bash
# galera_new_cluster
```

### Starting the Remaining Nodes

Then start the mariadb service in the remaining nodes sequentially. **Don't start the mariadb service simultaneously on the remaining nodes**

```bash
# systemctl start mariadb
```

## Resyncing Node/Node Fails to Restart Service

Follow the following process if a node needs to be resynced if the mariadb service within it is not able to start or join  the galera cluster.

1. **Kill the currently running  mariadb process and sst processes.**

   ``` bash
   # ps aux | grep mysql | awk '{print $2}' | xargs kill -9
   ```

2. **Remove the files under the mariadb datadir i.e. /var/lib/mysql**

   ```bash
   # rm -rvf /var/lib/mysql/*
   ```

3. **Start the mariadb server in tmux (or any terminal multiplexer) and wait for the data to be synced and the events to  be processed.**

   ```bash
   # tmux
   # systemctl start mariadb
   ```

4. **Watch out the files in /var/lib/mysql for the sst transfers and mariadb.err logfile for progress and errors.**

   **SST Transfer**

   * sst_in_progress: If this file is present then then SST transfer are on going and the conf and pid files for the rsync sst will be present accordingly (wsrep_sst.pid, rsync_sst.conf, rsync_sst.pid).

   * During the SST transfers 

   * The Donor would be selected from the running  nodes and the wsrep_local_state_comment would be set to Donor/Desynced for that specific node.

     ```
     MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_local_state_comment';
     +---------------------------+----------------+
     | Variable_name             | Value          |
     +---------------------------+----------------+
     | wsrep_local_state_comment | Donor/Desynced |
     +---------------------------+----------------+
     1 row in set (0.001 sec)
     ```

   * Note: Do not stop the mariadb service or kill the process during the  transfers.

   **After the completion of SST Transfer**

   * After the SST Transfer completes the galera processes the event queues which contains the transactions which happened in the master while the SST transfers were going on.

   * The progress of the event processing could be gained by tailing the mariadb.err file and would something like below:

     ``` 
     root@db:/var/lib/mysql# grep -i event mariadbd.err 
     2022-11-02  6:00:48 0 [Note] WSREP: Receiving IST...  0.0% (    0/16472 events) complete.
     2022-11-02  6:00:49 0 [Note] WSREP: Receiving IST...100.0% (16472/16472 events) complete.
     
     ```

   * The wsrep_local_state_comment status of the node would be shown as JOINED until the events are processed completely which can be seen by executing the following query in the server.

     ```sql
     MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_local_state_comment';
     +---------------------------+--------+
     | Variable_name             | Value  |
     +---------------------------+--------+
     | wsrep_local_state_comment | Joined |
     +---------------------------+--------+
     1 row in set (0.001 sec)
     ```

   * Verify the cluster size and the wsrep_local_state_comment in each of the nodes.

     All of the nodes' wsrep_local_state_comment should be shown as Synced.

     The cluster size should be shown as according  to the nodes available in the cluster. A working node  with 3 node cluster should show something like below.

     ```sql
     MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_local_state_comment';
     +---------------------------+--------+
     | Variable_name             | Value  |
     +---------------------------+--------+
     | wsrep_local_state_comment | Synced |
     +---------------------------+--------+
     1 row in set (0.001 sec)
     
     MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_cluster_size';
     +--------------------+-------+
     | Variable_name      | Value |
     +--------------------+-------+
     | wsrep_cluster_size | 3     |
     +--------------------+-------+
     1 row in set (0.001 sec
     ```

     