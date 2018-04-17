# Scenario 1 - ETCD Loss and Recovery

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


Useful command for this lab:

```
journalctl -fu <service_name> - follow logs of the service
chronyc -a 'burst 4/4' && chronyc -a makestep - force time sync using chronyc
ansible all/masters/infras/ -m shell -a "hostname" - execute adhoc command on subset of servers
```

[solution Task 1](solution_part1.md)

[lab Intro](../README.md)

[appendix](appendix.md)

[next lab](../scenario2/part1.md)