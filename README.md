---
description: >-
  this article describe how  to debug tetragon which writed for observe
  container security based on ebpf
---

# tetragon debug

## 调试依赖

tetragon github说明调试都在virtualbox中进行，所以开始调试之前需要安装vagrant和virtualbox。

并且源码中有提供Vagrant，通过Vagrant启动virtualbox时，会自动安装所有的依赖和工具。

详情参考tetragon源码。

## 本地编译

* 首先安装llvm和libbpf的依赖

```bash
make tools-install
```

* 然后编译bpf程序和go程序

```bash
sudo LD_LIBRARY_PATH=$(realpath ./lib) ./tetragon --bpf-lib bpf/objs
```

然后可以看到本地生成了tetragon和tetra程序：

```bash
```



## docker编译



## k8s 调试

