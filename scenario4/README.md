### Problem to solve

```
Complexity: Easy
Length: 10-20 min
Dashboard: Builds Overview
```

### Intro

In this scenario, we will be debugging an application with the command `oc debug`.

Type `oc debug -h` for more information on how to use it.

To start this scenario execute the following command on the `bastion` host:
```
> lab -s 4 -a init
```

This should create a new project, `hello-openshift`, in the OpenShift Container Platform cluster. At first it is an empty project, but the next steps will populate it with content.

First, navigate to the directory called `hello-openshift` in root's home directory. It is a very simple `goLang` application to demonstrate binary build and simple application debugging methods.

```
> cd ~/hello-openshift
> oc project hello-openshift
```

Next, create the application build as a docker build and of type `binary`:
```
> oc new-build --strategy docker --binary --docker-image centos:centos7 --name hello-openshift
```

Next, start the build which will create the application image. The flag `--from-dir` tells the BuildConfig that we will be using the local file system as source for the build:
```
> oc start-build hello-openshift --from-dir . --follow
```

Inspect the ImageStream to see that the application image build succeeded:
```
> oc get is
NAME              DOCKER REPO                                                        TAGS      UPDATED
centos            docker-registry.default.svc:5000/hello-openshift/centos            centos7   3 minutes ago
hello-openshift   docker-registry.default.svc:5000/hello-openshift/hello-openshift   latest    16 seconds ago
```

Next, deploy the application from the ImageStream.
```
> oc new-app hello-openshift
> oc expose svc/hello-openshift
```

... then check the status of the application:
```
> oc get pods
NAME                       READY     STATUS              RESTARTS   AGE
hello-openshift-1-6l22c    0/1       RunContainerError   3          47s
hello-openshift-1-build    0/1       Completed           0          3m
hello-openshift-1-deploy   1/1       Running             0          53s
```

From here, use `oc debug` to check why the application is not starting by starting it in debug mode. As part of fixing it, change the local files and rebuild the application again (from `oc start-build`) and confirm that application is running.

### Solution

Start the failing application in debug mode and check what is wrong:
```
> oc get pods

# example output:
> oc get pods
NAME                       READY     STATUS             RESTARTS   AGE
hello-openshift-1-6l22c    0/1       CrashLoopBackOff   4          3m
hello-openshift-1-build    0/1       Completed          0          6m
hello-openshift-1-deploy   1/1       Running            0          3m


# start the application in debug mode:
> oc debug hello-openshift-1-6l22c
Debugging with pod/hello-openshift-1-6l22c-debug, original command: /helo-openshift
Waiting for pod to start ...
Pod IP: 10.217.0.55
If you don't see a command prompt, try pressing enter.
```

The output in debug mode suggest that the container's entry point is `/helo-openshift`. To further investigate, running in debug mode provides an interactive shell to the container. Try executing the entrypoint command to see what happens:
```
> /helo-openshift
sh: /helo-openshift: No such file or directory
```

As this command does not exist, find the correct one and change the entrypoint:
```
sh-4.2$ ls -la
total 6024
drwxr-xr-x.  16 root root     278 Mar 23 15:45 .
drwxr-xr-x.  16 root root     278 Mar 23 15:45 ..
-rwxr-xr-x.   1 root root       0 Mar 23 15:45 .dockerenv
-rw-r--r--.   1 root root   11976 Mar  2 01:07 anaconda-post.log
lrwxrwxrwx.   1 root root       7 Mar  2 01:06 bin -> usr/bin
drwxr-xr-x.   5 root root     400 Mar 23 15:45 dev
drwxr-xr-x.  47 root root    4096 Mar 23 15:45 etc
-rwxr-xr-x.   1 root root 6149040 Mar 23 15:39 hello-openshift
drwxr-xr-x.   2 root root       6 Nov  5  2016 home
lrwxrwxrwx.   1 root root       7 Mar  2 01:06 lib -> usr/lib
lrwxrwxrwx.   1 root root       9 Mar  2 01:06 lib64 -> usr/lib64
drwxr-xr-x.   2 root root       6 Nov  5  2016 media
drwxr-xr-x.   2 root root       6 Nov  5  2016 mnt
drwxr-xr-x.   2 root root       6 Nov  5  2016 opt
dr-xr-xr-x. 241 root root       0 Mar 23 15:45 proc
dr-xr-x---.   2 root root     114 Mar  2 01:07 root
drwxr-xr-x.  11 root root     145 Mar 23 15:45 run
lrwxrwxrwx.   1 root root       8 Mar  2 01:06 sbin -> usr/sbin
drwxr-xr-x.   2 root root       6 Nov  5  2016 srv
dr-xr-xr-x.  13 root root       0 Mar 23 08:37 sys
drwxrwxrwt.   7 root root     132 Mar  2 01:07 tmp
drwxr-xr-x.  13 root root     155 Mar  2 01:06 usr
drwxr-xr-x.  18 root root     238 Mar  2 01:07 var
```

First, try running again with the correct binary name - `hello-openshift`:
```
sh-4.2$ ./hello-openshift
sh: ./hello-openshift: Permission denied
```

Because we are running non-privileged container we cant change file permissions here.

Now that we know where the issues are.
1. Change the ENTRYPOINT value in the Dockerfile
2. Open the file for execution
3. Rebuild the application.

We can do this by modifying dockerfile on the bastion host:
```
> sudo vi Dockerfile
# change
USER 1001
ENTRYPOINT ["/helo-openshift"]

# to

RUN chmod 777 /hello-openshift
USER 1001
ENTRYPOINT ["/hello-openshift"]
```


And rebuild the application:
```
> oc start-build hello-openshift --from-dir . --follow
```

And check again:
```
> oc get pods
NAME                       READY     STATUS      RESTARTS   AGE
hello-openshift-1-build    0/1       Completed   0          18m
hello-openshift-1-deploy   0/1       Error       0          16m
hello-openshift-2-build    0/1       Completed   0          6m
hello-openshift-2-t9l9r    1/1       ContainerRunError     0          5m
```

Once the build has finished, check the route to the application:
```
> oc get route
NAME              HOST/PORT                                                     PATH      SERVICES          PORT       TERMINATION   WILDCARD
hello-openshift   hello-openshift-hello-openshift.apps.129.146.122.240.xip.io             hello-openshift   8080-tcp   edge          None
```

Check if the application is responding to requests:
```
> curl -k https://hello-openshift-hello-openshift.apps.129.146.122.240.xip.io
Hello OpenShift!
```

#### How to monitor these particular cases?

Each application should expose individual metrics for its own performance.  Prometheus client libraries running on Linux will usually expose a metric by the name of `process_start_time_seconds` which is the Unix time at which the process started. Every time it changes for a given target means that the process has restarted. Conveniently PromQL has a changes() function which can count the number of changes in a time series over time.

Based on this information, an alert can be created - i.e.: triggered by how often this value changes - implying that restarts happens.

```
groups:
 - name: example
   rules:
    - alert: JobRestarting
     expr: avg without(instance)(changes(process_start_time_seconds[1h])) > 3
     for: 10m
     labels:
       severity: LOW
```

To resolve the scenario, run the following command:
```
> lab -s 4 -a solve
```

### Appendix

#### Materials used in the scenario

1. Openshift routes guide:
https://docs.openshift.com/container-platform/3.9/dev_guide/routes.html

2. Openshift Binary build guide:
https://docs.openshift.com/container-platform/3.9/dev_guide/dev_tutorials/binary_builds.html



### [**-- HOME --**](https://rht-labs-events.github.io/summit-lab-2018-doc/)
