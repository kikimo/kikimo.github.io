---
title: "Calico IP Migration 失败导致 calico-node 服务启动问题排查"
date: 2020-02-01T18:35:10+08:00
draft: true
---

calico-node 启动时会尝试执行 IP migration 操作，这个步骤如果执行失败会导致 calico-node 启动失败。

给 k8s 部署完 Calico，
发现 新建的 pod 始终分配不到 IP。
检查发现 calico-kube-controller 服务一直在不停的重启，
而 Node 节点上的 calico-node 服务一直处于 init：

```bash
alex@jenny:~/k8s/certs$ kubectl  -n kube-system get pod
NAME                                      READY   STATUS     RESTARTS   AGE
calico-kube-controllers-99bd8767b-dx6c9   0/1     Error      2          17s
calico-node-7lz54                         0/1     Init:0/3   0          30m
```

这应该是导致 calico-kube-controller 不停重启的原因了：
calico-node 启动失败导致 calico-kube-controller 分配不到 IP 进而不停的重启。
检查 calico-node 的详情发现 pod 卡在一个 init container upgrade-ipam 上：

```bash
alex@jenny:~/k8s/certs$ kubectl  -n kube-system describe pod  calico-node-kz8ft
Name:                 calico-node-kz8ft
Namespace:            kube-system
Priority:             2000001000
Priority Class Name:  system-node-critical
Node:                 jenny/192.168.2.108
Start Time:           Sat, 18 Jan 2020 22:18:05 +0800
Labels:               controller-revision-hash=868fbfb8bc
                      k8s-app=calico-node
                      pod-template-generation=1
Annotations:          scheduler.alpha.kubernetes.io/critical-pod: 
Status:               Pending
IP:                   192.168.2.108
Controlled By:        DaemonSet/calico-node
Init Containers:
  upgrade-ipam:
    Container ID:  docker://c5bfeb485cba6d6e18bfeb525d7c5237d4ebbec34812e817f2494447d873ce47
    Image:         calico/cni:v3.11.1
    Image ID:      docker-pullable://calico/cni@sha256:e493af94c8385fdfbce859dd15e52d35e9bf34a0446fec26514bb1306e323c17
    Port:          <none>
    Host Port:     <none>
    Command:
      /opt/cni/bin/calico-ipam
      -upgrade
    State:          Running
      Started:      Sat, 18 Jan 2020 22:18:06 +0800
    Ready:          False
    Restart Count:  0
    Environment:
      KUBERNETES_NODE_NAME:        (v1:spec.nodeName)
      CALICO_NETWORKING_BACKEND:  <set to the key 'calico_backend' of config map 'calico-config'>  Optional: false
    Mounts:
      /host/opt/cni/bin from cni-bin-dir (rw)
      /var/lib/cni/networks from host-local-net-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from calico-node-token-p2kj5 (ro)
  install-cni:
    Container ID:  
    Image:         calico/cni:v3.11.1
    Image ID:      
    Port:          <none>
    Host Port:     <none>
    Command:
      /install-cni.sh
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:
      CNI_CONF_NAME:         10-calico.conflist
      CNI_NETWORK_CONFIG:    <set to the key 'cni_network_config' of config map 'calico-config'>  Optional: false
      KUBERNETES_NODE_NAME:   (v1:spec.nodeName)
      CNI_MTU:               <set to the key 'veth_mtu' of config map 'calico-config'>  Optional: false
      SLEEP:                 false
    Mounts:
      /host/etc/cni/net.d from cni-net-dir (rw)
      /host/opt/cni/bin from cni-bin-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from calico-node-token-p2kj5 (ro)
  flexvol-driver:
    Container ID:   
    Image:          calico/pod2daemon-flexvol:v3.11.1
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /host/driver from flexvol-driver-host (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from calico-node-token-p2kj5 (ro)
Containers:
  calico-node:
    Container ID:   
    Image:          calico/node:v3.11.1
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Requests:
      cpu:      250m
    Liveness:   exec [/bin/calico-node -felix-live -bird-live] delay=10s timeout=1s period=10s #success=1 #failure=6
    Readiness:  exec [/bin/calico-node -felix-ready -bird-ready] delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:
      DATASTORE_TYPE:                     kubernetes
      WAIT_FOR_DATASTORE:                 true
      NODENAME:                            (v1:spec.nodeName)
      CALICO_NETWORKING_BACKEND:          <set to the key 'calico_backend' of config map 'calico-config'>  Optional: false
      CLUSTER_TYPE:                       k8s,bgp
      IP:                                 autodetect
      CALICO_IPV4POOL_IPIP:               Always
      FELIX_IPINIPMTU:                    <set to the key 'veth_mtu' of config map 'calico-config'>  Optional: false
      CALICO_IPV4POOL_CIDR:               10.200.0.0/16
      CALICO_DISABLE_FILE_LOGGING:        true
      FELIX_DEFAULTENDPOINTTOHOSTACTION:  ACCEPT
      FELIX_IPV6SUPPORT:                  false
      FELIX_LOGSEVERITYSCREEN:            info
      FELIX_HEALTHENABLED:                true
    Mounts:
      /lib/modules from lib-modules (ro)
      /run/xtables.lock from xtables-lock (rw)
      /var/lib/calico from var-lib-calico (rw)
      /var/run/calico from var-run-calico (rw)
      /var/run/nodeagent from policysync (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from calico-node-token-p2kj5 (ro)
Conditions:
  Type              Status
  Initialized       False 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  lib-modules:
    Type:          HostPath (bare host directory volume)
    Path:          /lib/modules
    HostPathType:  
  var-run-calico:
    Type:          HostPath (bare host directory volume)
    Path:          /var/run/calico
    HostPathType:  
  var-lib-calico:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/calico
    HostPathType:  
  xtables-lock:
    Type:          HostPath (bare host directory volume)
    Path:          /run/xtables.lock
    HostPathType:  FileOrCreate
  cni-bin-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /opt/cni/bin
    HostPathType:  
  cni-net-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/cni/net.d
    HostPathType:  
  host-local-net-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/cni/networks
    HostPathType:  
  policysync:
    Type:          HostPath (bare host directory volume)
    Path:          /var/run/nodeagent
    HostPathType:  DirectoryOrCreate
  flexvol-driver-host:
    Type:          HostPath (bare host directory volume)
    Path:          /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
    HostPathType:  DirectoryOrCreate
  calico-node-token-p2kj5:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  calico-node-token-p2kj5
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  beta.kubernetes.io/os=linux
Tolerations:     :NoSchedule
                 :NoExecute
                 CriticalAddonsOnly
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/network-unavailable:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/pid-pressure:NoSchedule
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  36m   default-scheduler  Successfully assigned kube-system/calico-node-kz8ft to jenny
  Normal  Pulled     36m   kubelet, jenny     Container image "calico/cni:v3.11.1" already present on machine
  Normal  Created    36m   kubelet, jenny     Created container upgrade-ipam
  Normal  Started    36m   kubelet, jenny     Started container upgrade-ipam
```

init container 是 pod 在启动阶段可以指定运行的一些任务容器，他们跑完就结束了，等他们跑完才会跑 pod 的 container。
检查 upgrade-ipam 这个容器到底在干嘛：

```bash

```
