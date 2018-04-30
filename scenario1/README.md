### Problem to solve

```
Complexity: Medium
Length: 20-30 min
Dashboard: Labs Generic
```

In this scenario we learn how to detect and identify application storage problems. The backing storage for this lab is Container Native Storage (CNS), running in the same Openshift Container Platform cluster. We will investigate a few issues, and use troubleshooting steps to help identify storage problems.

:heavy_check_mark: GlusterFS metrics will be available in Prometheus from 3.9+. This is outside of the scope of this Lab.*


To start the lab scenario, execute the following on the bastion host.

*This might take a minute or two*

```
lab -s 1 -a init
```

Now try kicking off a build within the OpenShift Container Platform cluster (there are pre-deployed projects and applications that can be used).

... or inspect any of the builds in the `ci-cd` project/namespace:

```
oc project ci-cd
oc logs -f summit-labs-fe-5-build
...
Pushing image docker-registry.default.svc:5000/ci-cd/summit-labs-fe:uat-summit-labs-fe-bake.2
Registry server Address:
Registry server User Name: serviceaccount
Registry server Email: serviceaccount@example.org
Registry server Password: <<non-empty>>
error: build error: Failed to push image: received unexpected HTTP status: 500 Internal Server Error
```

The builds should fail. Continue to check some of the other builds for additional failures as those builds are being executed periodically (part of the lab noise making).

If no failed builds can be found, check the build that was just launched automatically:

```
oc project build-test
oc get pods
```
and inspect the logs.

Next, we will debug the issue with the build(s).


:exclamation: To be proactive on this type of issue we recommend to follow "failed build" metrics `openshift_build_total` in prometheus. This provides a healthy view on what is happening in the cluster. At the same time it is recommended to execute some "smoke test" builds on a periodic basis so issues can be detected early.

The actual logs in the build itself are typically not very helpful.

In this case it is not clear what happened. To troubleshoot this, inspect the registry logs as the build failed during the "push" phase of the newly build image.


Task 1: Identify why the build failed.

Task 2: Solve the issue. and confirm that the solution worked.

Useful commands:
```
oc log <pod>                                        # get logs from the pod
oc rsh <pod>                                        # rsh into pod
heketi-cli --user admin --secret $HEKTI_ADMIN_KEY   # execute heketi commands
gluster                                             # gluster commands
```

### Solution

First, inspect the integrated container registry logs to look for any indications on the problem:

```
oc project default
oc get pods

# get logs of the registry pod.
oc logs -f  docker-registry-2-5gktk
```

In this case, because the internal container registry is running in HA, it can be hard to find out which replica was serving failed request. If the registry is running with 1-3 pods, it is easy to check them all. However, in the event that the deployment have more than 3, it can be easier to scale down the deployment to 1 replica with command `oc scale dc/docker-registry --replicas=1` and rerun build again to get the right pod and logs.

After investigating logs of the registry, the correct error is found:

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

It may take a trained eye to spot the failure, but in this case the message of interest is:
```
time="2018-03-21T11:51:36.305582362Z" level=error msg="response completed with error" err.code=unknown err.detail="filesystem: mkdir /registry/docker/registry/v2/repositories/coolstore/cart/_uploads/68fe406b-6590-4b24-87e3-b84288f83aa7: no space left on device" err.message="unknown error"
```

To validate this even further, access the registry pod with `rsh` and check the mountpoint size:
```
oc rsh docker-registry-2-5gktk

du -sh /registry    
    10.0G    /registry

# try writing something to it
touch /registry/test
touch: cannot touch '/registry/test': No space left on device
```

Next, check what the container registry is using as its backend storage:
```
oc get pv,pvc
NAME                                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                       STORAGECLASS        REASON    AGE
pv/pvc-7f74133e-34de-11e8-9019-2cc26036f58d   10Gi       RWO            Delete           Bound     openshift-metrics/prometheus                glusterfs-storage             9h
pv/pvc-8086f0f7-34de-11e8-9019-2cc26036f58d   10Gi       RWO            Delete           Bound     openshift-metrics/prometheus-alertmanager   glusterfs-storage             9h
pv/pvc-81b01a7f-34de-11e8-9019-2cc26036f58d   10Gi       RWO            Delete           Bound     openshift-metrics/prometheus-alertbuffer    glusterfs-storage             9h
pv/pvc-a537007d-34e2-11e8-ba00-2cc260767803   10Gi       RWO            Delete           Bound     openshift-grafana/grafana                   glusterfs-storage             9h
pv/registry-volume                            10Gi       RWX            Retain           Bound     default/registry-claim                                                    12h

NAME                 STATUS    VOLUME            CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc/registry-claim   Bound     registry-volume   10Gi       RWX                           12h
```

The output above shows that the `registry-claim` is 10Gi in size, and is provided from a manually provisioned volume.

Starting with OpenShift Container Platform 3.9, a new option is available for the storageClass - `allowVolumeExpansion: true` - which allows dynamic expansion of the PV's.

The users can request larger volume for their PersistentVolumeClaim by simply editing the claim and requesting a larger size. This in turn will trigger expansion of the volume that is backing the underlying PersistentVolume.

For this lab scenario, the 10Gi PV will be manually expanded:

Check which PV ID was provisioned for the `registry-claim` using the Volume name from the PVC output:
```
oc get pv registry-volume -o yaml | grep path

# example output:
    path: glusterfs-registry-volume
```

Next, check what information `CNS` provides for this volume/brick.
```
oc project glusterfs
oc get pods

# rsh to heketi pod
oc rsh heketi-storage-1-r98zq

heketi-cli --user admin --secret $HEKETI_ADMIN_KEY volume list
```

Get the ID of the volume and check its info:
```
heketi-cli --user admin --secret $HEKETI_ADMIN_KEY volume list | grep  glusterfs-registry-volume

Id:cf7794a72e28b16860d367d19fe192a0    Cluster:8d28e8f000e2835582cc21e45bff7cae    Name:glusterfs-registry-volume


heketi-cli --user admin --secret $HEKETI_ADMIN_KEY volume info cf7794a72e28b16860d367d19fe192a0   

Name: glusterfs-registry-volume
Size: 10
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

As observed previously in OpenShift, its size is 10Gi. Next, expand the storage by adding +5Gi.
```
heketi-cli --user admin --secret $HEKETI_ADMIN_KEY volume expand --volume=cf7794a72e28b16860d367d19fe192a0 --expand-size=5
```

Next see if this helped with the recovery of the registry:
```
oc project default
Now using project "default" on server "https://console.example.com".

oc get pods

oc rsh docker-registry-2-5gktk

touch /registry/test
touch: cannot touch '/registry/test': No space left on device
```

Surprise :). This would be one of those cases were you would need to raise a support ticket with Red Hat support. For our lab we already know which issue was hit: https://bugzilla.redhat.com/show_bug.cgi?id=1538939

This should be fixed with next heketi release. To work around this issue, (information is in bugzilla) the underlying gluster cluster needs to be re-balanced.

Use `rsh` to access any of the gluster pods and perform the re-balancing :
```
oc project glusterfs

oc get pods

oc rsh glusterfs-storage-dt2s5

gluster volume list           

glusterfs-registry-volume
heketidbstorage
vol_106d888f55fdef210ba3885140f99d18
vol_377905d2401904491c3dcf150f14ca82
vol_4a6169a6be60c4f25431a0561d3b53f6
vol_f02d15415e491acef927da2f7f0c4bce

gluster volume rebalance glusterfs-registry-volume start
```


Next, repeat the manual test:
```
oc project default

oc get pods

oc rsh docker-registry-2-5gktk

touch /registry/test
```

The container registry behaves the same as any other application consuming storage, but its impact may be bigger as it is a vital part of the OpenShift Container Platform cluster and its operation. Depending on the storage provider, actions to extend, debug and manage the backing volumes will be different. In our case we used `heketi` to manage our Gluster (CNS) cluster.


To complete this scenario, execute the following command on the bastion:

```
lab -s 1 -a solve
```

Starting with OpenShift Container Platform 3.9, additional Prometheus metrics will provide better info about the OpenShift Container Platform storage health:
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



### [**-- HOME --**](https://rht-labs-events.github.io/summit-lab-2018-doc/)