---
description: >-
  this article describe how  to debug tetragon which writed for observe
  container security based on ebpf
---

# tetragon debug

## 调试依赖

tetragon github说明调试都在virtualbox中进行，所以开始<mark style="color:red;">调试之前需要安装vagrant和virtualbox</mark>。

tetragon源码中有提供Vagrant，通过Vagrant启动virtualbox时，会自动安装所有的依赖和工具。详情参考tetragon源码。

## 启动虚拟机

通过下面的命令启动并登陆到virtualbox虚拟机：

```bash
vagrant up
vagrant ssh
```

在部分vagrant老版本上，vagrant up可能会报错，可以删除disk配置再试

```git
diff --git a/Vagrantfildiffe b/Vagrantfile
--- a/Vagrantfile
+++ b/Vagrantfile
@@ -1,6 +1,5 @@
 Vagrant.configure("2") do |config|
   config.vm.box = "ubuntu/impish64"
-  config.vm.disk :disk, size: "50GB"
   config.vm.provision :docker
```

## 本地编译

* 首先安装llvm和libbpf的依赖

```bash
make tools-install
```

* 然后编译bpf程序和go程序

```bash
LD_LIBRARY_PATH=$(realpath ./lib) make
```

此命令会编译所有的可执行程序和Test程序，调试过程中为了提高效率可以只编译<mark style="color:red;">tetra和tetragon。</mark>

```bash
LD_LIBRARY_PATH=$(realpath ./lib) make tetra  tetragon
```

然后可以看到本地生成了tetragon和tetra程序，后续会说明吗这两个程序的作用。

```bash
 ls  tetra*
tetra  tetragon  tetragon-alignchecker
```

## 本地调试

### 启动tetragon

启动的时候需要指定ebpf .o文件在那个路径下，并且指定tetragon库。

```bash
sudo LD_LIBRARY_PATH=$(realpath ./lib) ./tetragon --bpf-lib bpf/objs
```

tetragon启动之后，会加载ebpf程序，并且在指定端口（默认为<mark style="color:red;">localhost:54321</mark>）等待listener的链接，listener连接之后，tetragon会把监听到的事件转发给listener。

tetragon启动log如下：

```log
time="2022-05-29T10:37:56Z" level=info msg="Starting tetragon" version=v0.8.0-106-g5c3fd60
time="2022-05-29T10:37:56Z" level=info msg="config settings" 
     config="map[bpf-lib:bpf/objs btf: cilium-bpf: config-dir: config-file: debug:false enable-cilium-api:false enable-export-aggregation:false enable-k8s-api:false enable-process-ancestors:true enable-process-cred:false enable-process-ns:false export-aggregation-buffer-size:10000 export-aggregation-window-size:15s export-allowlist: export-denylist: export-file-compress:false export-file-max-backups:5 export-file-max-size-mb:10 export-file-rotation-interval:0s export-filename: export-rate-limit:-1 force-small-progs:false ignore-missing-progs:false kernel: log-format:text log-level:info metrics-server: netns-dir:/var/run/docker/netns/ process-cache-size:65536 procfs:/proc/ run-standalone:false server-address:localhost:54321 verbose:0]"
time="2022-05-29T10:37:56Z" level=info msg="Available sensors" sensors=
time="2022-05-29T10:37:56Z" level=info msg="Registered tracing sensors" sensors="kprobe sensor, tracepoint sensor"
time="2022-05-29T10:37:56Z" level=info msg="Registered probe types" types="tracepoint sensor, kprobe sensor"
time="2022-05-29T10:37:56Z" level=info msg="Disabling Kubernetes API"
time="2022-05-29T10:37:56Z" level=info msg="Disabling Cilium API"
time="2022-05-29T10:37:56Z" level=info msg="Starting process manager" enableCilium=false enableEventCache=false enableProcessCred=false enableProcessNs=false
time="2022-05-29T10:37:56Z" level=info msg="Exporter configuration" enabled=false fileName=
time="2022-05-29T10:37:56Z" level=info msg="Using metadata file" metadata=
time="2022-05-29T10:37:56Z" level=info msg="Loading sensor" name=__main__
time="2022-05-29T10:37:56Z" level=info msg="Loading kernel version 5.13.19"
time="2022-05-29T10:37:56Z" level=info msg="Starting gRPC server" address="localhost:54321"
time="2022-05-29T10:37:56Z" level=info msg="tetragon, map loaded." map=execve_map path=/sys/fs/bpf/tcpmon/execve_map sensor=__main__
time="2022-05-29T10:37:56Z" level=info msg="tetragon, map loaded." map=execve_map_stats path=/sys/fs/bpf/tcpmon/execve_map_stats sensor=__main__
time="2022-05-29T10:37:56Z" level=info msg="tetragon, map loaded." map=names_map path=/sys/fs/bpf/tcpmon/names_map sensor=__main__
time="2022-05-29T10:37:56Z" level=info msg="tetragon, map loaded." map=tcpmon_map path=/sys/fs/bpf/tcpmon/tcpmon_map sensor=__main__
time="2022-05-29T10:37:56Z" level=info msg="BPF prog was loaded" label=tracepoint/sys_exit prog=bpf/objs/bpf_exit.o
time="2022-05-29T10:37:56Z" level=info msg="BPF prog was loaded" label=kprobe/wake_up_new_task prog=bpf/objs/bpf_fork.o
time="2022-05-29T10:37:56Z" level=info msg="Load probe" Program=bpf/objs/bpf_execve_event_v53.o Type=execve
time="2022-05-29T10:37:57Z" level=info msg="Read ProcFS /proc/ appended 74/218 entries"
time="2022-05-29T10:37:57Z" level=warning msg="Procfs execve event pods/ identifier error" error="open /proc/0/cgroup: no such file or directory"
time="2022-05-29T10:37:57Z" level=info msg="BPF prog was loaded" label=tracepoint/sys_execve prog=bpf/objs/bpf_execve_event_v53.o
time="2022-05-29T10:37:57Z" level=info msg="Loaded BPF maps and events for sensor successfully" sensor=__main__
time="2022-05-29T10:37:57Z" level=info msg="Listening for events..."
```

从log中可以看到<mark style="color:orange;">tetragon启动的config参数</mark>（第二行），还有tetragon的version等等。

### 修改默认log等级

从config参数中可以看到，默认的log级别是info，如果修改log等级为debug，需要在启动时添加-<mark style="color:red;">-log-level=debug</mark>参数。

```bash
sudo LD_LIBRARY_PATH=$(realpath ./lib) ./tetragon --bpf-lib bpf/objs --log-level=debug
```

### 启动listerner：tetra

tetra是tetragon的一个CLI调试工具，tetra的具体参数和用法可以通过直接执行tetra查看。

```bash
Tetragon CLI

Usage:
  tetra [flags]
  tetra [command]

Available Commands:
  bugtool         Produce a tar archive with debug information
  getevents       Print events
  help            Help about any command
  sensors         Manage sensors
  stacktrace-tree Manage stacktrace trees
  status          Print health status
  tracingpolicy   Manage tracing policies
  version         Print version

Flags:
  -d, --debug                   Enable debug messages
  -h, --help                    help for tetra
      --server-address string   gRPC server address (default "localhost:54321")

Use "tetra [command] --help" for more information about a command.
```

通过<mark style="color:red;">tetra getevents</mark>可以获取tetragon发送的ebpf事件。tetra默认以json格式输出，例如：

```bash
 $ ./tetra getevents | jq "."
{
  "process_exec": {
    "process": {
      "exec_id": "OjI0Njk4Mjk4ODYxNDI6MTI0NDE=",
      "pid": 12441,
      "uid": 0,
      "cwd": "/run/containerd/io.containerd.runtime.v2.task/k8s.io/25ecffd7e2ee8332923500bd930eaebe2725bb643e7e9e2206b611f8666abac0/",
      "binary": "/usr/local/sbin/runc",
      "arguments": "--root /run/containerd/runc/k8s.io --log /run/containerd/io.containerd.runtime.v2.task/k8s.io/c7c9f733b618c7b46f0bd86e054fc5d940fbd7eb627cf009d6d9cd0584c846a6/log.json --log-format json exec --process /tmp/runc-process789407863 --detach --pid-file /run/containerd/io.containerd.runtime.v2.task/k8s.io/c7c9f733b618c7b46f0bd86e054fc5d940fbd7eb627cf009d6d9cd0584c846a6/4c9ea3805e0d2a23de4c0cdb983c52db832689aa821d3363ba8954b1b76359d9.pid c7c9f733b618c7b46f0bd86e054fc5d940fbd7eb627cf009d6d9cd0584c846a6",
      "flags": "execve clone",
      "start_time": "2022-05-29T10:54:19.452Z",
      "auid": 4294967295,
      "parent_exec_id": "OjMzMzgwMDAwMDAwOjI0NzM=",
      "refcnt": 1
    },
    "parent": {
      "exec_id": "OjMzMzgwMDAwMDAwOjI0NzM=",
      "pid": 2473,
      "uid": 0,
      "cwd": "/run/containerd/io.containerd.runtime.v2.task/k8s.io/25ecffd7e2ee8332923500bd930eaebe2725bb643e7e9e2206b611f8666abac0",
      "binary": "/usr/local/bin/containerd-shim-runc-v2",
      "arguments": "-namespace k8s.io -id 25ecffd7e2ee8332923500bd930eaebe2725bb643e7e9e2206b611f8666abac0 -address /run/containerd/containerd.sock",
      "flags": "procFS auid",
      "start_time": "2022-05-29T10:13:43.002Z",
      "auid": 0,
      "parent_exec_id": "OjIyODIwMDAwMDAwOjExOTE=",
      "refcnt": 4294967265
    }
  },
  "time": "2022-05-29T10:54:19.452Z"
}
```

通过<mark style="color:red;">tetra getevents --output compact</mark> <mark style="color:blue;">能获取到更友好的输出。</mark>

```bash
./tetra getevents --output compact
🚀 process  /usr/sbin/iptables -w 5 -W 100000 -S KUBE-KUBELET-CANARY -t mangle 
💥 exit     /usr/sbin/iptables -w 5 -W 100000 -S KUBE-KUBELET-CANARY -t mangle 0 
🚀 process  /usr/sbin/ip6tables -w 5 -W 100000 -S KUBE-KUBELET-CANARY -t mangle 
💥 exit     /usr/sbin/ip6tables -w 5 -W 100000 -S KUBE-KUBELET-CANARY -t mangle 0 
```

## docker中编译

docker中编译的命令如下，在docker中编译默认会打包docker镜像，详细参考Makefile。

```bash
# Build Tetragon agent and operator images
LD_LIBRARY_PATH=$(realpath ./lib) make LOCAL_CLANG=0 image image-operator

# Bootstrap the cluster
contrib/localdev/bootstrap-kind-cluster.sh

# Install Tetragon
contrib/localdev/install-tetragon.sh --image cilium/tetragon:latest --operator cilium/tetragon-operator:latest
```

整个过程因为会按照kind工具，并且通过kind 安装单节点k8s cluster，所以会比较慢。

编译之后，tettagon会被部署在k8s cluster中，可以通过kubectl命令查看。

```
kubectl get pods -n kube-system
```

## k8s cluster中调试

当tetragon在k8s cluster中部署之后，通过下面的命令查查看事件上报，输出的log比较多（json格式），可以通过jq命令进行日志过滤。

```
kubectl logs -n kube-system ds/tetragon -c export-stdout -f
```
