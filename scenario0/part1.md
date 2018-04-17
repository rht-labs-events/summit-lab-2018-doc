# Scenario 0 - ETCD Loss and Recovery

```
Complexity: High
Length: 20-30 min
Dashboard: ETCD
```

In this lab we will see how loss of the quorum can impact Platforms behaviour and how we can recover from it for the time being. And will advise on material for future readings when planning real life DR situations.

etcd stores the persistent master state while other components watch etcd for changes to bring themselves into the desired state. etcd can be optionally configured for high availability, typically deployed with 2n+1 peer services.

In this lab etcd runs together on masters. Before starting open Grafana dashboard `ETCD` and inspect monitoring data from our etcd cluster.

You should see something like this:

![alt text](img/img1-etcd-dasboard.png)

At the same time open prometheus url and check alerts tab for any active alerts. At this point if the environment is functioning properly, you should see no alerts firing.

![alt text](img/img2-no-alerts.png)

We can check etcd manually via command from the master or any remote host which has the certificates. If you chose to do this from other host than master in your infrastructure, make sure you apply appropriate security mitigation to this host, as you are "leaking" platforms certificates outside the ring fenced hosts.

If you dont have dashboards and alerting tools in place, next big thing when it comes to debugging is `etcdctl` utility. Lets check how we can use it.

ssh to one of the master hosts:
```
ssh root@master1.example.com
```

execute a command to check etcd health status. This command will give you each endpoint health status. 
```
[root@master1]# docker exec -it etcd_container sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/etcd/peer.crt --key=/etc/etcd/peer.key --cacert=/etc/etcd/ca.crt --endpoints="[https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379]" endpoint status -w table"

#Output example:
+---------------------------+------------------+---------+---------+-----------+-----------+------------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+---------------------------+------------------+---------+---------+-----------+-----------+------------+
| https://192.168.0.11:2379 | b2fc96740d4db02e |  3.2.15 |   65 MB |      true |       964 |    1121049 |
| https://192.168.0.12:2379 | 3eea3b05deae6cbc |  3.2.15 |   65 MB |     false |       964 |    1121051 |
| https://192.168.0.13:2379 | 509718481af2e12e |  3.2.15 |   65 MB |     false |       964 |    1121051 |
+---------------------------+------------------+---------+---------+-----------+-----------+------------+
```

In this table we can see current etcd status. Which node is the leader, database size, version, unique ID, raft term, raft index. 
`Raft Term` is an integer that will increase whenever an etcd master election happens in the cluster. If this number is increasing rapidly, you may need to tune the election timeout. 
`Raft Index` - This is more complex, but you can think about this one as "data consistency" metrics. Those should be equal on all replicas (or very small difference).


Now more generic health command:

```
[root@master1]# docker exec -it etcd_container sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/etcd/peer.crt --key=/etc/etcd/peer.key --cacert=/etc/etcd/ca.crt --endpoints="[https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379]"  endpoint health -w table"

###Output:
https://master2.example.com:2379 is healthy: successfully committed proposal: took = 6.889365ms
https://master3.example.com:2379 is healthy: successfully committed proposal: took = 9.472592ms
https://master1.example.com:2379 is healthy: successfully committed proposal: took = 5.291873ms
```

One more useful command is to list all the keys and data in the etcd. In this example we get one of the templates:
```
docker exec -it etcd_container sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/etcd/peer.crt --key=/etc/etcd/peer.key --cacert=/etc/etcd/ca.crt --endpoints="[https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379]"  get /openshift.io/templates/openshift/datagrid65-postgresql --prefix"
```


<b>NOTE: Openshift consumes etcd as external service. There are scenarios when despite the fact that etcd show healthy status, cluster will not be working. This usually happens when cluster looses conectivity to the etcd cluster. This might happen because of firewalls, certificates or any other reason. For this reason you should always smoke test your cluster with "noise makers" to make sure you can use mutating API calls.</b>


Manual command are useful because if ETCD goes to Read-Only mode (when quorum is lost) we might lose nice UI monitoring features of the platform as some of those tools uses mutating api calls, and are changing platform. Those requires healthy etcd cluster.

Now lets see what would happen if we lose one etcd node. 

Execute from the bastion:

```
lab -s 0 -a break1
```

In a few seconds you should see that grafana reports that only 2 etcd are alive.

Check prometheus and alertmanager for any alerts.

We can see now lost etcd alert in prometheus and alertmanager:

![alt text](img/img3-lost-etcd-alert.png)

![alt text](img/img4-lost-etcd-alertmanager.png)

If you execute same `etcdctl` command on the masters again, you will see this represented in the output too.

Our platform still performs fine, as we have quorum available. Now lets see what will happen when we remove second node from the pool. Note, grafana dashboard will stop showing graphs. This is expected as Grafana uses mutable queries to the openshift api. And without quorum all api is responding is read-only.

Execute from the bastion:

```
lab -s 0 -a break2
```

Now you should see "hell break loose" in the all dashboards.

![alt text](img/img5-granafa-single-survivor.png)

![alt text](img/img6-hell-got-loose.png)

This is very common scenario in the deployments where you have only 2 datacenters. In this architecture one of the possible deployment ways is that you will need to split your quorum system in 2. Which means if you loose one datacenter, you lost quorum. This is just one of the possible architecture for Openshift. We always recommend to split 3 masters/etcd in 3 availability zones. But this is not always possible.

Lab Goal:

<b>When doing Task 1 of this scenario you should not do anything with master2 and master3. Assume you lost them and your platform have to be recovered using ONLY master1.</b>

Task 1: Put master1 ETCD process (etcd_container) to run as `single-node-cluster` so cluster could work using one node etcd. When done, make sure that your Openshift cluster behaves fine with one etcd.

Task 2: Now lets assume you got master2 and master3 back. You should add master2 ETCD and master3 ETCD to the existing cluster of master1.
This will involve adding new member to the master1 cluster and reconfiguring master2 and master3 to join this cluster.

If you want to skip these task, execute on the <b>bastion</b>
```
 lab -s 0 -a solve
```

Useful command for this lab:

```
journalctl -fu <service_name> - follow logs of the service
ansible all/masters/infras/ -m shell -a "hostname" - execute adhoc command on subset of servers
```

[solution Task 1](solution_part1.md)

[lab Intro](../README.md)

[appendix](appendix.md)

[next lab](../scenario1/part1.md)