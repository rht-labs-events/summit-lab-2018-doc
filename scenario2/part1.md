# Scenario 2 - Storage issues


```
Complexity: Medium
Length: 20-30 min
Dashboard: Labs Generic
```

In this scenario we learn how to detect and identify application storage problems. In this scenario we are using Container Native Storage (CNS), running on the same Openshift cluster.
We will investigate few issues and solutions how to see if you get storage problems.

*NOTE: GlusterFS metrics will be available in Prometheus from 3.9+. For same of "real world" we will not use those for now as it was not available in previous releases. If you want more details on this - get in touch with your instructors for more information. *


To start scenario, execute on the bastion. 

<b>This might take a minute or two, relax :)</b>

```
lab -s 2 -a init
```

Now try build something. We have pre-deployed apps on the platform for you to consume. Check project ci-cd for any running builds and inspect them.

Or inspect any other build in `ci-cd` namespace (change any failed builds):

```
[root@workstation]# oc project ci-cd
[root@workstation]# oc logs -f summit-labs-fe-5-build
...
Pushing image docker-registry.default.svc:5000/ci-cd/summit-labs-fe:uat-summit-labs-fe-bake.2
Registry server Address:
Registry server User Name: serviceaccount
Registry server Email: serviceaccount@example.org
Registry server Password: <<non-empty>>
error: build error: Failed to push image: received unexpected HTTP status: 500 Internal Server Error
```

Build should fail. Check other builds, you might have few already failed, as those build are being executed periodically (part of the lab noise making).

<b>If you dont have any, you can check build we just launched for you:</b>

```
oc project build-test
oc get pods
```
and inspect logs for it.

Now we will debug whats the issue with our build.


<b>NOTE: To be proactive on this type of issue we recommend to follow "failed build" metrics `openshift_build_total` in prometheus. This would give you healthy view on what is happening in the cluster. At the same time we recommend to execute some "smoke test" build on timely basis on the cluster so issue could be detected early.</b>

Logs in the build is not very helpful.

In this case its not clear whats happening. For this we will go and inspect registry logs, as it failed during a "push" phase of the build.


Task 1: Identify why build failed.

Task 2: Solve the issue. and confirm solution worked.

Useful commands:
```
oc log <pod> - get logs from the pod
oc rsh <pod> - rsh into pod
heketi-cli --user admin --secret $HEKTI_ADMIN_KEY - execute heketi commands
gluster - gluster commands
```
[solution Task 1](solution_part1.md)

[lab Intro](../README.md)

[appendix](appendix.md)

[next lab](../scenario3/part1.md)