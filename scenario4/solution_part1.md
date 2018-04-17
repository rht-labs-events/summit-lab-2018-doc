= Solution

Lets start our failing application in debug mode and check whats wrong:
```
oc get pods

#example output:
[root@workstation-REPL hello-openshift]# oc get pods
NAME                       READY     STATUS             RESTARTS   AGE
hello-openshift-1-6l22c    0/1       CrashLoopBackOff   4          3m
hello-openshift-1-build    0/1       Completed          0          6m
hello-openshift-1-deploy   1/1       Running            0          3m


# start app in debug mode:
oc debug hello-openshift-1-6l22c
root@workstation-REPL hello-openshift]# oc debug hello-openshift-1-6l22c
Debugging with pod/hello-openshift-1-6l22c-debug, original command: /helo-openshift
Waiting for pod to start ...
Pod IP: 10.217.0.55
If you don't see a command prompt, try pressing enter.
```

From the output debug mode suggest us that containers entry point is `/helo-openshift`.
And debug mode gives us interactive shell to the same container. Lets try execute entrypoint command and see:
```
sh-4.2$  /helo-openshift
sh: /helo-openshift: No such file or direct
```

So lets find what is the right command:
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

Dam typo troll... Try right binary name:
```
sh-4.2$ ./hello-openshift 
serving on 8080
serving on 8888
```

Now when we know where is the issue we can change dockerfile and rebuild the application:
```
vi Dockerfile

#change 
ENTRYPOINT ["/helo-openshift"]
 to
ENTRYPOINT ["/hello-openshift"]
```

and rebuild the application:
```
oc start-build hello-openshift --from-dir . --follow
```

And check again:
```
[root@workstation-REPL hello-openshift]# oc get pods
NAME                       READY     STATUS      RESTARTS   AGE
hello-openshift-1-build    0/1       Completed   0          18m
hello-openshift-1-deploy   0/1       Error       0          16m
hello-openshift-2-build    0/1       Completed   0          6m
hello-openshift-2-t9l9r    1/1       Running     0          5m
[root@workstation-REPL hello-openshift]# oc get route
NAME              HOST/PORT                                                     PATH      SERVICES          PORT       TERMINATION   WILDCARD
hello-openshift   hello-openshift-hello-openshift.apps.129.146.122.240.xip.io             hello-openshift   8080-tcp   edge          None
[root@workstation-REPL hello-openshift]# curl https://hello-openshift-hello-openshift.apps.129.146.122.240.xip.io -k
Hello OpenShift!
```

### How to monitor these particular cases?

Each application should expose individual metrics for its own performance.  Prometheus client libraries when running on Linux will usually expose a metric by the name of process_start_time_seconds which is the Unix time at which the process started. Every time it changes for a given target means that the process has restarted. Conveniently PromQL has a changes() function which can count the number of changes in a time series over time.

So we can craft an alert, based on how often this value changes. Implying - restart happens.

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

To resolve the scenario:
```
lab -s 4 -a solve
```

[instructions Task 1](part1.md)

[lab Intro](../README.md)

[appendix](appendix.md)

[next lab](../scenario5/part1.md)