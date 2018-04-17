# Recovery

## Task 1 solution

Lets assume you need to bring platform to the usable state, but you still dont have second datacenter available. For this we can switch single surviving etcd node to the single master configuration. And when surviving etcd becomes available for us, we can add them back to the cluster.

### Switch master1/etcd1 to single master mode

ssh to master1:
```
ssh master1.example.com
```
Force new cluster from 1 etcd node:
```
[root@master1]# sed -i '/ExecStart=/s/$/  --force-new-cluster/' /etc/systemd/system/etcd_container.service
[root@master1]# systemctl daemon-reload
[root@master1]# systemctl restart etcd_container

#Check logs of the container
journalctl -fu etcd_container
```

Recovery might take up to few minutes. But you should see platform getting to better shape now.

![alt text](img/img7-one-etcd.png)

![alt text](img/img7-one-etcd.png)

All other operations should get back to normal. We just told master1 (surviving etcd node) to "start new cluster with existing data". You can check builds in ci-cd namespace. They should be starting again.

### Task 2 solution

Now lets assume you got your second DC back. But etcd cluster is now out of sync. We need to create new cluster, by adding 2 lost etcd to the survivor as new members.

Remove `--force-new-cluster` flag from member one.
ssh to master1:
```
ssh master1.example.com
```
re-edit the `/etc/systemd/system/etcd_container.service` file and remove the --force-new-cluster option:
```
[root@master1]# sed -i '/ExecStart/s/ --force-new-cluster//' /etc/systemd/system/etcd_container.service
[root@master1]# systemctl show etcd_container.service --property ExecStart --no-pager

ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd"
```

At this point etcd still runs with old systemd.

```
[root@master1]# systemctl daemon-reload
[root@master1]# systemctl restart etcd_container
```
Check etcd 1 member list:
```
[root@master1]# docker exec -it etcd_container sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/etcd/peer.crt --key=/etc/etcd/peer.key --cacert=/etc/etcd/ca.crt --endpoints="[https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379]" member list"

b2fc96740d4db02e, started, master1.example.com, https://192.168.0.11:2380, https://192.168.0.11:2379
```

Add member 2:
```
[root@master1]#  docker exec -it etcd_container sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/etcd/peer.crt --key=/etc/etcd/peer.key --cacert=/etc/etcd/ca.crt --endpoints="[https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379]" member add master2.example.com --peer-urls="https://192.168.0.12:2380""

Member 8d13245ff0d59b2b added to cluster 447e150364ce5cc3

ETCD_NAME="master2.example.com"
ETCD_INITIAL_CLUSTER="master2.example.com=https://192.168.0.12:2380,master1.example.com=https://192.168.0.11:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```
Save these variables somewhere. We will need them in a second.

Now ssh to master2 and update main etcd details with these variables:

<b>IMPORTANT: remove double quotes from the output when updating etcd config file. systemd and etcd does not like them</b>

```
ssh master2.example.com
```

Update etcd configuration:
```
[root@master2 ~]# vi /etc/etcd/etcd.conf
ETCD_NAME=master2.example.com
ETCD_INITIAL_CLUSTER=master2.example.com=https://192.168.0.12:2380,master1.example.com=https://192.168.0.11:2380
ETCD_INITIAL_CLUSTER_STATE=existing
```

remove old member data:
```
[root@master2 ~]# rm -rf /var/lib/etcd/member
```

start etcd container
```
systemctl start etcd_container
```

<b>NOTE: If you see something like error below, you potentially didnt removed quotes in the etcd config file</b>
```
Mar 31 16:31:35 master2.example.com etcd_container[14625]: 2018-03-31 20:31:35.854136 W | pkg/netutil: failed resolving host master2.example.com:2380" (address tcp/2380": unknown port); retrying in 1
```

Check logs `journalctl -fu etcd_container`

Member list on master1 should show you now 2 running members:
```
[root@master1 ~]# docker exec -it etcd_container sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/etcd/peer.crt --key=/etc/etcd/peer.key --cacert=/etc/etcd/ca.crt --endpoints="[https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379]" member list"
4f1716f1da0e8dd9, started, master2.example.com, https://192.168.0.12:2380, https://192.168.0.12:2379
b2fc96740d4db02e, started, master1.example.com, https://192.168.0.11:2380, https://192.168.0.11:2379
```


Repeate same for master3:

on master1:
```
[root@master1 ~]# docker exec -it etcd_container sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/etcd/peer.crt --key=/etc/etcd/peer.key --cacert=/etc/etcd/ca.crt --endpoints="[https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379]" member add master3.example.com --peer-urls="https://192.168.0.13:2380""

Member ffcc8fc41a1321d7 added to cluster 447e150364ce5cc3

ETCD_NAME="master3.example.com"
ETCD_INITIAL_CLUSTER="master2.example.com=https://192.168.0.12:2380,master3.example.com=https://192.168.0.13:2380,master1.example.com=https://192.168.0.11:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

Change to master3:
```
ssh master3.example.com
```

Update etcd configuration:
```
[root@master3 ~]# vi /etc/etcd/etcd.conf
ETCD_NAME=master3.example.com
ETCD_INITIAL_CLUSTER=master2.example.com=https://192.168.0.12:2380,master3.example.com=https://192.168.0.13:2380,master1.example.com=https://192.168.0.11:2380
ETCD_INITIAL_CLUSTER_STATE=existing
```

remove old member data:
```
[root@master3 ~]# rm -rf /var/lib/etcd/member
```
start container
```
[root@master3 ~]# systemctl start etcd_container
```

Now check again cluster health with command from the beginning of the scenario.

This was simple failure and recover scenario, where old nodes was available for us. We have ansible playbooks to do all these things for you. If nodes are lost unrecoverably, there is addition steps involved to generate new certificates and distribute them. But this is out of scope for this lab.

Now you should see all 3 ETCD back online in Grafana and no alerts in prometheus. If this is not a case - call instructor :)

[instructions Task 1](part1.md)

[lab Intro](../README.md)

[appendix](appendix.md)

[next lab](../scenario1/part1.md)
