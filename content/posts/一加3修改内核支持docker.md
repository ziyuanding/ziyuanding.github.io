---
title: "一加3修改内核支持docker"
date: 2024-11-21
---

成品 `boot.img` 见 https://github.com/ziyuanding/oneplus3_kernel_with_docker_support

# 准备工作
1. 手机连上电脑
2. 准备使用ADB: `cd platform-tools`
  
    ADB 应该有这样的结果
    ```
    (base) PS     platform-tools> ./adb devices
    List of devices attached
    d32159b3        device
    ```

    手动进入fastboot模式，试试看 `fastboot devices`
    ```
    tools> ./fastboot devices
    d32159b3        fastboot
    ```
看到了类似以上的输出就好了。

# rec
lineageos的rec我用不习惯（虽然也有adb sideload和安装zip的功能），还是去https://dl.twrp.me/oneplus3/ 下载了twrp的rec, 最新的是 `twrp-3.7.0_9-0-oneplus3.img`



手机连上电脑后，刷twrp的rec

`./fastboot flash recovery ..\twrp-3.7.0_9-0-oneplus3.img`

刷好rec后，可以手动操作(电源+音量加/减)进入rec. 

# os
进入rec界面后，把data格式化，然后cache也都清一清

高级 -》 ADB Sideload

然后电脑执行

`
.\adb.exe sideload ..\lineage-18.1-20240305\lineage-18.1-20240305-nightly-oneplus3-signed.zip
`

# 刷boot.img
系统原装的boot.img可以在 `lineage-18.1-20240305-nightly-oneplus3-signed.zip`里找到，直接解压出来即可。

刷入boot很简单，也不会丢数据，唯一的风险是卡开机。

`fastboot flash boot boot.img`

需要注意的是，刷完boot.img后，root权限会丢失。

# 刷root
进入rec后，找到 `Magisk-v26.4.apk.zip` (在“手动存入”里)，刷入。

# 在安卓上运行docker
## 编译内核
主要参考 https://gist.github.com/FreddieOliveira/efe850df7ff3951cb62d74bd770dce27
和 https://ivonblog.com/posts/how-to-compile-custom-android-kernel/
https://ivonblog.com/posts/run-docker-natively-on-android/
https://www.jianshu.com/p/c719e6a459ab

采用的主要是最后一篇来自 @fanxcv-简书 的资料。

``` bash
# 安装编译工具依赖
apt update 
apt install git gnupg flex bison gperf build-essential zip curl zlib1g-dev libc6-dev-i386 x11proto-core-dev libx11-dev libgl1-mesa-dev libxml2-utils xsltproc unzip bc gcc-aarch64-linux-gnu

export CUS_KERNEL_HOME=~/customkernel
mkdir -p $CUS_KERNEL_HOME/tools
cd $CUS_KERNEL_HOME

# git clone 前请先检查自己的git是否会修改代码的换行符 git config --global core.autocrlf 如果输出是true, 则说明 Git 会将 Unix 风格换行符转换为 Windows 风格
# 可以通过 git config --global core.autocrlf input 修改，然后重写clone

# 检出编译工具 clang
git clone https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86 --depth=1 -b android-vts-11.0_r8 $CUS_KERNEL_HOME/tools/clang

# 检出安卓GCC
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9 --depth=1 -b android-cts-9.0_r20 $CUS_KERNEL_HOME/tools/aarch64-linux-android-4.9
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9 --depth=1 -b android-cts-9.0_r20 $CUS_KERNEL_HOME/tools/arm-linux-androideabi-4.9

# 编写环境变量脚本
cat <<EOF > $CUS_KERNEL_HOME/env.sh
#!/usr/bin/env bash
base=$CUS_KERNEL_HOME/tools
export PATH=${base}/aarch64-linux-android-4.9/bin:${base}/arm-linux-androideabi-4.9/bin:${base}/clang/clang-r383902b/bin:$PATH
# 这个config需要符合你的机型
export config=lineageos_oneplus3_defconfig

export args="-j 32 O=out CC=clang ARCH=arm64 SUBARCH=arm64 CLANG_TRIPLE=aarch64-linux-gnu- CROSS_COMPILE=aarch64-linux-android- CROSS_COMPILE_ARM32=arm-linux-androideabi-"
EOF
source env.sh

# 下载内核源码
git clone --depth 1 https://github.com/LineageOS/android_kernel_oneplus_msm8996.git $CUS_KERNEL_HOME/src
cd src

# 清理(可反复执行)
make clean && rm -rf out
make mrproper

# 以默认内核配置做模板生成新配置，会生成到out/.config
make ${args} ${config}

# 对生成的默认内核配置进行修改，如果不使用args, 那么会使用默认的x64配置，那就和手机完全不一样了
# 这个命令会修改上一条生成的out/.config 并备份旧的
make ${args} menuconfig

# 依照 https://gist.github.com/FreddieOliveira/efe850df7ff3951cb62d74bd770dce27 里提供的termux端 检查脚本check-config.sh 的缺失项修改配置完成后，即可进行内核编译：
make ${args}
```

内核编译成功后，能在日志最后一行看到：

`
CAT   arch/arm64/boot/Image.gz-dtb
`
这就是成功了，我们需要去这个目录找到Image.gz-dtb，并用它替换官方boot.img里的组件。

我使用的工具是 `Android Image Kitchen Win32` ，简称 **AIK**, 这个工具也有linux版本的，但是我运行不成功。

将官方的boot.img复制到AIK目录下，执行unpackimg.bat，它会解压boot.img并生成split_img文件夹，进入该文件夹，把你的Image.gz-dtb改名为boot.img-kernel并覆盖掉原本的boot.img-kernel。

然后执行repackimg.bat，此时生成 boot-new.img, 我们就用这个正常刷就行了。

## 刷入内核后，Termux端
RunC requires support for devices cgroup support in kernel.

If CONFIG_CGROUP_DEVICE was enabled during compile time,
you need to run the following commands (as root) in order
to use the RunC:

sudo  mount -t tmpfs -o mode=755 tmpfs /sys/fs/cgroup
sudo  mkdir -p /sys/fs/cgroup/devices
sudo  mount -t cgroup -o devices cgroup /sys/fs/cgroup/devices

If you got error when running commands listed above, this
usually means that your kernel lacks CONFIG_CGROUP_DEVICE.

### 方法一
修改dockerd配置
```bash
$ mkdir -p $PREFIX/etc/docker
$ cat << "EOF" > $PREFIX/etc/docker/daemon.json
{
    "data-root": "/data/docker/lib/docker",
    "exec-root": "/data/docker/run/docker",
    "pidfile": "/data/docker/run/docker.pid",
    "hosts": [
        "unix:///data/docker/run/docker.sock"
    ],
    "storage-driver": "overlay2" # 我这里必须改成vfs才行
}
EOF
```

然后，运行docker时强制指定docker.sock
```sudo DOCKER_HOST=<the docker.sock location spit out when you start dockerd> docker xxxx```



### 方法二（最终使用的）
直接使用vfs，不过这可能会是一种低效的空间利用方案

```bash
cat << "EOF" > $PREFIX/etc/docker/daemon.json
{
    "storage-driver": "vfs"
}
EOF
```
使用docker时无需指定位置，正常使用即可。

## 其他注意事项
FreddieOliveira提供的检测脚本是检测 `/proc/config.gz` 的，有的手机没有这个文件。这个文件的生成也是受内核控制的，你可以先不用管docker的内核选项，先去General Options -> Kernel Config gz 把这个启用后，正常编译内核，正常刷入。然后在termux里执行测试脚本就能看到docker所需要的内核选项了。