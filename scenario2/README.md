### Problem to solve

```
Complexity: Medium
Length: 20-30 min
Dashboard: Labs Generic
```

In this scenario we learn how to detect and identify application storage problems. In this scenario we are using Container Native Storage (CNS), running on the same Openshift cluster.
We will investigate few issues and solutions how to see if you get storage problems.

:heavy_check_mark: GlusterFS metrics will be available in Prometheus from 3.9+. For same of "real world" we will not use those for now as it was not available in previous releases. If you want more details on this - get in touch with your instructors for more information. *


To start scenario, execute on the bastion.

*This might take a minute or two, relax :)*

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

If you dont have any, you can check build we just launched for you:

```
oc project build-test
oc get pods
```
and inspect logs for it.

Now we will debug whats the issue with our build.


:exclamation: To be proactive on this type of issue we recommend to follow "failed build" metrics `openshift_build_total` in prometheus. This would give you healthy view on what is happening in the cluster. At the same time we recommend to execute some "smoke test" build on timely basis on the cluster so issue could be detected early.

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

### Solution

Lets check integrated container registry logs to find out what the problem.

```
oc project default
oc get pods
#get logs of the registry pod.
oc logs -f  docker-registry-2-5gktk
```

Because our registry is running in HA replica 3 sometimes it is hard to find out which replica was serving failed request. If you have 1-3 pods, its easy to check them all. If you have more, sometimes its easier to scale down deployment to replica 1 with command `oc scale dc/docker-registry --replicas=1` and rerun build again to get the right logs.

After invetigating logs of the registry we find right error:

```
time="2018-03-21T11:51:36.238113538Z" level=debug msg="authorizing request" go.version=go1.8.3 http.request.host="docker-registry.default.svc:5000" http.request.id=89871542-6cd9-4431-aee6-61cc018cf2a6 http.request.method=POST http.request.remoteaddr="10.217.0.1:58498" http.request.uri="/v2/coolstore/cart/blobs/uploads/" http.request.useragent="docker/1.12.6 go/go1.8.3 kernel/3.10.0-693.el7.x86_64 os/linux arch/amd64 Upstr
eamClient(go-dockerclient)" instance.id=a92227ef-2c2c-4917-8548-2a23888eeb85 openshift.logger=registry vars.name="coolstore/cart"

time="2018-03-21T11:51:36.25415814Z" level=debug msg="Origin auth: checking for access to repository:coolstore/cart:pull" go.version=go1.8.3 http.request.host="docker-registry.default.svc:5000" http.request.id=89871542-6cd9-4431-aee6-61cc018cf2a6 http.request.method=POST http.request.remoteaddr="10.217.0.1:58498" http.request.uri="/v2/coolstore/cart/blobs/uploads/" http.request.useragent="docker/1.12.6 go/go1.8.3 kernel/3
.10.0-693.el7.x86_64 os/linux arch/amd64 UpstreamClient(go-dockerclient)" instance.id=a92227ef-2c2c-4917-8548-2a23888eeb85 openshift.auth.user="system:serviceaccount:coolstore:builder" openshift.logger=registry vars.name="coolstore/cart"

time="2018-03-21T11:51:36.256916033Z" level=debug msg="Origin auth: checking for access to repository:coolstore/cart:push" go.version=go1.8.3 http.request.host="docker-registry.default.svc:5000" http.request.id=89871542-6cd9-4431-aee6-61cc018cf2a6 http.request.method=POST http.request.remoteaddr="10.217.0.1:58498" http.request.uri="/v2/coolstore/cart/blobs/uploads/" http.request.useragent="docker/1.12.6 go/go1.8.3 kernel/
3.10.0-693.el7.x86_64 os/linux arch/amd64 UpstreamClient(go-dockerclient)" instance.id=a92227ef-2c2c-4917-8548-2a23888eeb85 openshift.auth.user="system:serviceaccount:coolstore:builder" openshift.logger=registry vars.name="coolstore/cart"

time="2018-03-21T11:51:36.259894906Z" level=info msg="Using \"docker-registry.default.svc:5000\" as Docker Registry URL" go.version=go1.8.3 instance.id=a92227ef-2c2c-4917-8548-2a23888eeb85 openshift.logger=registry

time="2018-03-21T11:51:36.259997658Z" level=debug msg="(*linkedBlobStore).Writer" go.version=go1.8.3 http.request.host="docker-registry.default.svc:5000" http.request.id=89871542-6cd9-4431-aee6-61cc018cf2a6 http.request.method=POST http.request.remoteaddr="10.217.0.1:58498" http.request.uri="/v2/coolstore/cart/blobs/uploads/" http.request.useragent="docker/1.12.6 go/go1.8.3 kernel/3.10.0-693.el7.x86_64 os/linux arch/amd64
 UpstreamClient(go-dockerclient)" instance.id=a92227ef-2c2c-4917-8548-2a23888eeb85 openshift.auth.user="system:serviceaccount:coolstore:builder" openshift.logger=registry vars.name="coolstore/cart"

time="2018-03-21T11:51:36.305179698Z" level=debug msg="filesystem.PutContent(\"/docker/registry/v2/repositories/coolstore/cart/_uploads/68fe406b-6590-4b24-87e3-b84288f83aa7/startedat\")" go.version=go1.8.3 http.request.host="docker-registry.default.svc:5000" http.request.id=89871542-6cd9-4431-aee6-61cc018cf2a6 http.request.method=POST http.request.remoteaddr="10.217.0.1:58498" http.request.uri="/v2/coolstore/cart/blobs/up
loads/" http.request.useragent="docker/1.12.6 go/go1.8.3 kernel/3.10.0-693.el7.x86_64 os/linux arch/amd64 UpstreamClient(go-dockerclient)" instance.id=a92227ef-2c2c-4917-8548-2a23888eeb85 openshift.auth.user="system:serviceaccount:coolstore:builder" openshift.logger=registry trace.duration=45.072341ms trace.file="/builddir/build/BUILD/atomic-openshift-git-0.8edc154/_output/local/go/src/github.com/openshift/origin/vendor/g
ithub.com/docker/distribution/registry/storage/driver/base/base.go" trace.func="github.com/openshift/origin/vendor/github.com/docker/distribution/registry/storage/driver/base.(*Base).PutContent" trace.id=5ac3aa62-9c46-431b-982e-06e26958c0bd trace.line=95 vars.name="coolstore/cart"

time="2018-03-21T11:51:36.305582362Z" level=error msg="response completed with error" err.code=unknown err.detail="filesystem: mkdir /registry/docker/registry/v2/repositories/coolstore/cart/_uploads/68fe406b-6590-4b24-87e3-b84288f83aa7: no space left on device" err.message="unknown error" go.version=go1.8.3 http.request.host="docker-registry.default.svc:5000" http.request.id=89871542-6cd9-4431-aee6-61cc018cf2a6 http.reque
st.method=POST http.request.remoteaddr="10.217.0.1:58498" http.request.uri="/v2/coolstore/cart/blobs/uploads/" http.request.useragent="docker/1.12.6 go/go1.8.3 kernel/3.10.0-693.el7.x86_64 os/linux arch/amd64 UpstreamClient(go-dockerclient)" http.response.contenttype="application/json; charset=utf-8" http.response.duration=69.390113ms http.response.status=500 http.response.written=242 instance.id=a92227ef-2c2c-4917-8548-2
a23888eeb85 openshift.auth.user="system:serviceaccount:coolstore:builder" openshift.logger=registry vars.name="coolstore/cart"

10.217.0.1 - - [21/Mar/2018:11:51:36 +0000] "POST /v2/coolstore/cart/blobs/uploads/ HTTP/1.1" 500 242 "" "docker/1.12.6 go/go1.8.3 kernel/3.10.0-693.el7.x86_64 os/linux arch/amd64 UpstreamClient(go-dockerclient)"  
```

Message we are looking for:
```
time="2018-03-21T11:51:36.305582362Z" level=error msg="response completed with error" err.code=unknown err.detail="filesystem: mkdir /registry/docker/registry/v2/repositories/coolstore/cart/_uploads/68fe406b-6590-4b24-87e3-b84288f83aa7: no space left on device" err.message="unknown error"
```

To validate this even more we rsh to the registry pod and check mountpoint size:
```
[root@workstation-REPL ~]# oc rsh docker-registry-2-5gktk
sh-4.2$ du -sh /registry    
    10.0G    /registry

# try writing something to it
sh-4.2$ touch /registry/test
touch: cannot touch '/registry/test': No space left on device
```

Now lets check what our registry is using as backend storage:
```
[root@workstation-REPL ~]# oc get pv,pvc
[root@master1 ~]# oc get pv,pvc
NAME                                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                       STORAGECLASS        REASON    AGE
pv/pvc-7f74133e-34de-11e8-9019-2cc26036f58d   10Gi       RWO            Delete           Bound     openshift-metrics/prometheus                glusterfs-storage             9h
pv/pvc-8086f0f7-34de-11e8-9019-2cc26036f58d   10Gi       RWO            Delete           Bound     openshift-metrics/prometheus-alertmanager   glusterfs-storage             9h
pv/pvc-81b01a7f-34de-11e8-9019-2cc26036f58d   10Gi       RWO            Delete           Bound     openshift-metrics/prometheus-alertbuffer    glusterfs-storage             9h
pv/pvc-a537007d-34e2-11e8-ba00-2cc260767803   10Gi       RWO            Delete           Bound     openshift-grafana/grafana                   glusterfs-storage             9h
pv/registry-volume                            30Gi       RWX            Retain           Bound     default/registry-claim                                                    12h

NAME                 STATUS    VOLUME            CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc/registry-claim   Bound     registry-volume   30Gi       RWX                           12h
```

We can see that our `registry-claim` is 10Gi in size, and is provided manually from manually provisioned storage.
In 3.9+ you can use new option on the storageClass `allowVolumeExpansion: true`, which would allow dynamic expansion of the PV's.
User can request larger volume for their PersistentVolumeClaim by simply editing the claim and requesting a larger size. This in turn will trigger expansion of the volume that is backing the underlying PersistentVolume.

In our case we will do this manually.


Check which PV ID was provisioned for the `registry-claim` using Volume name from PVC output:
```
oc get pv registry-volume -o yaml | grep path

#example output:
    path: glusterfs-registry-volume
```

Now lets check what `CNS` can tell us about this volume/brick.
```
oc project glusterfs
oc get pods

#rsh to heketi pod
oc rsh heketi-storage-1-r98zq

sh-4.2# heketi-cli --user admin --secret $HEKETI_ADMIN_KEY volume list
```

Get the ID of the volume we are looking for and check info:
```
sh-4.2# heketi-cli --user admin --secret $HEKETI_ADMIN_KEY volume list | grep  glusterfs-registry-volume
Id:cf7794a72e28b16860d367d19fe192a0    Cluster:8d28e8f000e2835582cc21e45bff7cae    Name:glusterfs-registry-volume


sh-4.2# heketi-cli --user admin --secret $HEKETI_ADMIN_KEY volume info cf7794a72e28b16860d367d19fe192a0   
Name: glusterfs-registry-volume
Size: 30
Volume Id: cf7794a72e28b16860d367d19fe192a0
Cluster Id: 8d28e8f000e2835582cc21e45bff7cae
Mount: 192.168.0.21:glusterfs-registry-volume
Mount Options: backup-volfile-servers=192.168.0.22,192.168.0.23
Block: false
Free Size: 0
Block Volumes: []
Durability Type: replicate
Distributed+Replica: 3
```

We can see that size is 10Gi as observed previously. Lets expand our storage by adding +5Gi.
```
heketi-cli --user admin --secret $HEKETI_ADMIN_KEY volume expand --volume=cf7794a72e28b16860d367d19fe192a0 --expand-size=5
```

Lets see if our action helped to recover our registry.
```
root@workstation-REPL ~]# oc project default
ocNow using project "default" on server "https://console.example.com".
[root@workstation-REPL ~]# oc get pods
[root@workstation-REPL ~]# oc rsh docker-registry-2-5gktk
sh-4.2$ touch /registry/test
touch: cannot touch '/registry/test': No space left on device
```

Surprise :) As in the real world, sometimes things does not work as expected :) . This would be one of those cases were you would need to raise support ticket with Redhat support. But for our lab we already know to which issue we ran into: https://bugzilla.redhat.com/show_bug.cgi?id=1538939
This should be fixed with next heketi release. But what we need to do now (information is in bugzilla) is to rebalance underlaying gluster cluster.

Now rsh to any gluster pod and do rebalancing :
```
oc project glusterfs
oc get pods
oc rsh glusterfs-storage-dt2s5

sh-4.2# gluster volume list           
glusterfs-registry-volume
heketidbstorage
vol_106d888f55fdef210ba3885140f99d18
vol_377905d2401904491c3dcf150f14ca82
vol_4a6169a6be60c4f25431a0561d3b53f6
vol_f02d15415e491acef927da2f7f0c4bce
sh-4.2# gluster volume rebalance glusterfs-registry-volume start
```


Now lets repeat our manual test:
```
oc project default
oc get pods
oc rsh docker-registry-2-5gktk
sh-4.2$ touch /registry/test
```

Container registry behaved as same as any other application, which would be consuming storage. Just it has bigger impact to the all cluster performance, than standalone application. Depending on your storage provider, actions to extend, debug and manage will be different. In our case we used `heketi` to manage our GlusterFS cluster.



When you done with this scenario, execute command on the bastion:

```
lab -s 2 -a solve
```

In addition from 3.9+ you will have access to more Prometheus metrics providing better info about your storage health:
```
kubelet_volume_stats_capacity_bytes
kubelet_volume_stats_inodes
kubelet_volume_stats_inodes_free
kubelet_volume_stats_inodes_used
kubelet_volume_stats_used_bytes
```
And metrics from heketi: https://github.com/heketi/heketi/pull/1068

### Appendix

#### Materials used in the scenario

1. Container Native Storage user guide:
https://access.redhat.com/documentation/en-us/red_hat_gluster_storage/3.3/html/container-native_storage_for_openshift_container_platform/

2. Gluster rebalancing documentation:
https://access.redhat.com/documentation/en-US/Red_Hat_Storage/2.0/html/Administration_Guide/sect-User_Guide-Managing_Volumes-Rebalancing.html
