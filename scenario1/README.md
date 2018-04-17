### Problem to solve

```
Complexity: Low
Length: 10-15 min
Dashboard: Labs Generic
```

In this lab we will see how external factor to etcd can impact Platforms behaviour and how we can recover from it for the time being.

To start scenario:
```
lab -s 1 -a init
```

You may not see direct impact yo the cluster, but dashboards and alerts might help you :)


Useful commands for this lab:

```
journalctl -fu <service_name> - follow logs of the service
chronyc -a 'burst 4/4' && chronyc -a makestep - force time sync using chronyc
ansible all/masters/infras/ -m shell -a "hostname" - execute adhoc command on subset of servers
```

### Solution

#### Task 3 solution

This scenario is one the most common issue we see. Distributed systems are very sensitive to time drifts. Even slight time difference can impact all platform performance. This is because one of the metrics, when agreeing on state is time.

#### Check time on all etcd nodes

Execute from the bastion:
```
[root@workstation-repl summit]# ansible etcd -m shell -a "date"
master2.example.com | SUCCESS | rc=0 >>
Mon Mar 26 07:39:17 EDT 2018

master3.example.com | SUCCESS | rc=0 >>
Mon Mar 26 07:40:57 EDT 2018

master1.example.com | SUCCESS | rc=0 >>
Mon Mar 26 07:42:36 EDT 2018
```

This is far from reliable data, but we can see already that time difference is huge. We can see same in grafana:

![alt text](img/11-time-drift.png)

Because we are running isolated system, our time drift is calculated based on average of all infrastructure. In the real world you would need external NTP server to check for the time.

We should see alert for this too:

![alt text](img/12-alert-time-drift.png)

Solution is very simple: 

Check if time keeping service is running:
```
ansible etcd -m shell -a "systemctl status chronyd"
```

Start the service:
```
ansible etcd -m shell -a "systemctl start chronyd"
```

Force time sync:
```
ansible etcd -m shell -a "chronyc -a 'burst 4/4' && chronyc -a makestep"
```

Alerts should become inactive, and grafana shows new state:

![alt text](img/13-time-drift.png)

### Appendix

#### Materials used in the scenario

1. Openshift Documentation on ETCD recovery: 
https://docs.openshift.com/container-platform/latest/admin_guide/backup_restore.html

2. Grafana dashboard and Prometheus alert rules:
<insert link to public dasboard link>

3. Etcd (v2) admin guide:
https://coreos.com/etcd/docs/latest/v2/admin_guide.html 
