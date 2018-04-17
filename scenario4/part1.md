# Scenario 4 - Binary build and debug application

```
Complexity: Easy
Length: 10-20 min
Dashboard: Builds Overview
```

Into what we will do

In this lab when debugging application you should use command `oc debug`. 

Type `oc debug -h` for information how to use it.

To start this scenario execute command on the bastion:
```
lab -s 4 -a init
```

You should have new project `hello-openshift` in your cluster. At the moment its empty project. But lets populate it. 

You now have folder `hello-openshift` in the root folder. Its very simple `goLang` application to demonstrate binary build and simple application debugging methods. 

Lets build an app:
```
cd ~/hello-openshift
oc project hello-openshift
```

Create new binary build:
```
oc new-build --strategy docker --binary --docker-image centos:centos7 --name hello-openshift
```

Flag `--binary` tells buildconfig that we will be using local file system as source for our build. Now lets star the build and build application image:
```
oc start-build hello-openshift --from-dir . --follow
```

We should see app build succ: 
```
[root@workstation-REPL hello-openshift]# oc get is
NAME              DOCKER REPO                                                        TAGS      UPDATED
centos            docker-registry.default.svc:5000/hello-openshift/centos            centos7   3 minutes ago
hello-openshift   docker-registry.default.svc:5000/hello-openshift/hello-openshift   latest    16 seconds ago
```

Now lets try deploy application:
```
oc new-app hello-openshift
oc expose svc/hello-openshift
```

And check our application. 
```
[root@workstation-REPL hello-openshift]# oc get pods
NAME                       READY     STATUS              RESTARTS   AGE
hello-openshift-1-6l22c    0/1       RunContainerError   3          47s
hello-openshift-1-build    0/1       Completed           0          3m
hello-openshift-1-deploy   1/1       Running             0          53s
[root@workstation-REPL hello-openshift]# 
```

Now use `oc debug` to check why application is not starting, try starting in debug mode. Later change local files and rebuild application again (from `oc start-build`) and confirm that app is running.

[solution Task 1](solution_part1.md)

[lab Intro](../README.md)

[appendix](appendix.md)

[next lab](../scenario5/part1.md)