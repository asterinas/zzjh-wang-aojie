# 说明书

## 前置准备

### 1. 下载 go 语言编译器

```cmd
wget https://go.dev/dl/go1.24.4.linux-amd64.tar.gz
tar -xf go1.24.4.linux-amd64.tar.gz
export GOROOT=`pwd`/go
export PATH=$GOROOT/bin:$PATH
```

### 2. 编译 Syzkaller

```cmd
cd syzkaller-pkvm
make
```

### 3. 配置 qemu

1. 安装依赖

   ```cmd
   sudo apt install build-essential zlib1g-dev pkg-config libglib2.0-dev binutils-dev libboost-all-dev autoconf libtool libssl-dev libpixman-1-dev virtualenv flex bison ninja-build
   ```

2. 获取源码

   ```cmd
   wget https://download.qemu.org/qemu-6.2.0.tar.xz
   ```

   > 由于 `sudo apt install qemu` 安装的版本较低，而过高的版本又要求更高版本的其他依赖，因此这里选择 qemu-6.2.0 版本

3. 编译安装

   ```cmd
   tar -xf qemu-6.2.0.tar.xz
   cd qemu-6.2.0
   mkdir build && cd build
   QEMU_DST_DIR=/usr/local/qemu  # 指定安装 qemu 的路径
   sudo mkdir $QEMU_DST_DIR
   ../configure --prefix=$QEMU_DST_DIR
   make -j4
   sudo make install
   vim ~/.bashrc
   export PATH=$PATH:/usr/local/qemu/bin  # 将该内容添加到 ~/.bashrc 文件
   ```

## Syzkaller 配置文件

针对 pKVM-IA 的 Syzkaller 的配置文件如下：

```json
{
    "target": "linux/amd64",
    "http": "127.0.0.1:56741",
    "rpc": "127.0.0.1:0",
    "workdir": "/path/to/syzkaller-pkvm/workdir-pkvm",
    "kernel_obj": "/path/to/tee/pKVM-IA",
    "syzkaller": "/path/to/syzkaller-pkvm",
    "image": "/path/to/tee/workdir/ubuntu.qcow2",
    "enable_syscalls": ["openat$kvm", "ioctl$KVM_CREATE_VM_PKVM", "ioctl$KVM_CREATE_VCPU", "syz_kvm_setup_syzos_vm$x86", "syz_kvm_add_vcpu$x86", "ioctl$KVM_GET_VCPU_MMAP_SIZE", "mmap$KVM_VCPU", "pkvm_fuzz"],
    "procs": 1,
    "type": "qemu",
    "vm": {
        "count": 1,
        "qemu": "/usr/local/qemu/bin/qemu-system-x86_64",
        "qemu_args": "-machine q35,accel=kvm,kernel-irqchip=split -cpu host,kvm-pv-unhalt=off,kvm-pv-ipi=off,kvm-pv-sched-yield=off -device intel-iommu,aw-bits=48 -name debug-threads=on -vga none -pidfile /home/wangaojie/tee/workdir/qemu.pid -monitor unix:/home/wangaojie/tee/workdir/qemu-monitor-socket,server,nowait -gdb tcp:127.0.0.1:1234",
        "cmdline": "root=/dev/sda2 console=ttyS0,115200n8 earlyprintk=ttyS0,115200n8 kvm-intel.pkvm=1 nokaslr systemd.mask=NetworkManager-wait-online.service systemd.mask=systemd-networkd-wait-online.service",
        "network_device": "virtio-net-pci",
        "cpu": 8,
        "mem": 16384,
        "kernel": "/path/to/tee/workdir/bzImage"
    }
}
```

需要注意，`workdir`、`kernel_obj`、`syzkaller` 、`image`、`qemu`、`kernel` 字段需修改为实际环境下的路径：

* `workdir`：Syzkaller 工作路径，包含虚拟机运行日志、崩溃报告等；
* `kernel_obj`：内核源代码路径，需要包含编译得到的原始的 `vmlinux` 文件；
* `syzkaller` ；Syzkaller 的源代码以及编译产物所在路径；
* `image`：磁盘镜像文件所在路径；
* `qemu`：qemu 二进制文件所在路径；
* `kernel`：待测内核镜像所在路径。

## pKVM-IA 内核镜像编译

直接在 pKVM-IA 路径下执行：

```cmd
make -j32
cp arch/x86/boot/bzImage ../workdir/bzImage
```

内核源代码相关改动以及配置文件改动已同步。

## 整体框架

整体的相关文件树状框架如下：

```txt
.
├── go1.24.4
├── qemu-6.2.0
├── syzkaller-pkvm
│   ├── ...
│	└── pkvm.cfg
└── tee
	├── boot.sh
	├── pKVM-IA
	└── workdir
		├── bzImage
		└── ubuntu.qcow2 # 该文件不在项目内，需要自行构建
```

## 运行 Fuzzing

```
./bin/syz-manager -config=pkvm.cfg
```

可视化报告通过浏览器访问 `127.0.0.1:56741` 转发到的端口来查看。

