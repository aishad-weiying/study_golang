# 服务器基础环境
服务器：DELL T640
操作系统： Ubuntu 16.04
docker: 18.09
DPU：2080Ti

## 具体配置GPU环境
1. 安装操作系统后做系统配置、ssh配置、网络配置、配置阿里云Ubuntu 基础镜像地址

2. 根据系统环境和显卡型号下载对应的驱动，官网地址：http://www.geforce.cn/drivers
这个是我下载好的2080TI的驱动地址：http://upload.aishad.top/NVIDIA-Linux-x86_64-430.40.run

3. 卸载原有的 nvidia 驱动
```bash
apt-get --purge remove   nvidia-*
apt-get autoremove
```

4. 禁用nouveau驱动和相关的驱动包
```bash
vim  /etc/modprobe.d/blacklist.conf
blacklist rivafb
blacklist vga16fb
blacklist nouveau
blacklist nvidiafb
blacklist rivatv

# 更新内核文件
update-initramfs -u
```

5. 重启服务器验证
```bash
lsmod | grep nouveau
# 如果没有输出，就说明禁用成功
```

6. 安装依赖
```bash
apt update
apt-get dist-upgrade
apt-get install gcc dkms build-essential linux-headers-generic pkg-config xorg
```

7. 安装驱动
```bash
chmod 755  NVIDIA-Linux-x86_64-418.56.run
./NVIDIA-Linux-x86_64-418.56.run -no-x-check -no-nouveau-check -no-opengl-files
```

8. 更新内核文件重启后验证
```bash
update-initramfs -u
reboot

# 验证显卡驱动安装成功
nvidia-smi
# 显示显卡信息安装成功
Fri Sep  6 04:01:51 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.87.00    Driver Version: 418.87.00    CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce RTX 208...  Off  | 00000000:84:00.0 Off |                  N/A |
| 41%   32C    P8    13W / 260W |   2305MiB / 10989MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

## 安装CUDA
CUDA™是一种由NVIDIA推出的通用并行计算架构，该架构使GPU能够解决复杂的计算问题。 它包含了CUDA指令集架构（ISA）以及GPU内部的并行计算引擎。

1. 下载CUDA
官方网站：https://developer.nvidia.com/cuda-downloads
我下载好的文件：http://upload.aishad.top/cuda_10.1.243_418.87.00_linux.run

2. 安装
```bash
bash cuda_10.1.243_418.87.00_linux.run
# 注意：在安装过程中会有一系列提示和确认，其中询问是否安装驱动的选项一定要选择否（如下），其余选择是或默认
#Install NVIDIA Accelerated Graphics Driver for Linux-x86_64 375.26?
#(y)es/(n)o/(q)uit: n
```

3. 配置环境变量
```bash
vim /etc/profile
export PATH=$PATH:/usr/local/cuda/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64
export LIBRARY_PATH=$LIBRARY_PATH:/usr/local/cuda/lib64

# 加载环境变量
source /etc/profile
```

4. 验证是否安装成功
```bash
nvcc -V

nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2019 NVIDIA Corporation
Built on Sun_Jul_28_19:07:16_PDT_2019
Cuda compilation tools, release 10.1, V10.1.243

# 验证cuda能否使用GPU
cd /usr/local/cuda-10.1/samples/1_Utilities/deviceQuery

root@ubuntu:/usr/local/cuda-10.1/samples/1_Utilities/deviceQuery# make
/usr/local/cuda/bin/nvcc -ccbin g++ -I../../common/inc  -m64    -gencode arch=compute_30,code=sm_30 -gencode ampute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=comp deviceQuery.cpp
/usr/local/cuda/bin/nvcc -ccbin g++   -m64      -gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,c_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_7
mkdir -p ../../bin/x86_64/linux/release
cp deviceQuery ../../bin/x86_64/linux/release

# 如果显示一些关于GPU的信息，结果为PASS，则说明安装成功。
root@ubuntu:/usr/local/cuda-10.1/samples/1_Utilities/deviceQuery# ./deviceQuery 
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "GeForce RTX 2080 Ti"
  CUDA Driver Version / Runtime Version          10.1 / 10.1
  CUDA Capability Major/Minor version number:    7.5
  Total amount of global memory:                 10989 MBytes (11523260416 bytes)
  (68) Multiprocessors, ( 64) CUDA Cores/MP:     4352 CUDA Cores
  GPU Max Clock rate:                            1635 MHz (1.63 GHz)
  Memory Clock rate:                             7000 Mhz
  Memory Bus Width:                              352-bit
  L2 Cache Size:                                 5767168 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  1024
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 3 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device supports Compute Preemption:            Yes
  Supports Cooperative Kernel Launch:            Yes
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 132 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 10.1, CUDA Runtime Version = 10.1, NumDevs = 1
Result = PASS
```

## 安装CUDNN
CUDA(ComputeUnified Device Architecture)，是显卡厂商NVIDIA推出的运算平台。 CUDA是一种由NVIDIA推出的通用并行计算架构，该架构使GPU能够解决复杂的计算问题。

1. 下载
我下载好的地址：http://upload.aishad.top/cudnn-10.1-linux-x64-v7.6.2.24.tgz

2. 解压并拷贝文件
```bash
tar xvf cudnn-10.1-linux-x64-v7.6.2.24.tgz 
cp cuda/include/cudnn.h /usr/local/cuda/include/
cp cuda/lib64/libcudnn* /usr/local/cuda/lib64/ -d
chmod a+r /usr/local/cuda/include/cudnn.h
chmod a+r /usr/local/cuda/lib64/libcudnn*
```

3. 更新链接库
```bash
ldconfig
```

## 安装docker
```bash
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update

# 查看指定版本的docker和docker-cli
apt-cache madison docker-ce
apt-cache madison docker-ce-cli

# 安装指定版本的docker和docker-cli
apt-get install docker-ce=5:18.09.9~3-0~ubuntu-bionic docker-ce-cli=5:18.09.9~3-0~ubuntu-bionic containerd.io
```

## 安装nvidia-docker2

1. 如果之前有安装nvidia-docker1.0，需要先卸载之前的旧版本
```bash
docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
apt-get purge -y nvidia-docker
```

2. 添加存储库及更新索引包
```bash
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey |  apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list |  tee /etc/apt/sources.list.d/nvidia-docker.list
apt-get update
```

3. 安装并重新载入docker daemon
```bash
pt-get install -y nvidia-docker2
pkill -SIGHUP dockerd
```

4. 验证是否安装成功，如果显示显卡信息，则安装成功
```bash
docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
Unable to find image 'nvidia/cuda:latest' locally
latest: Pulling from nvidia/cuda
35c102085707: Pull complete 
251f5509d51d: Pull complete 
8e829fe70a46: Pull complete 
6001e1789921: Pull complete 
9f0a21d58e5d: Pull complete 
47b91ac70c27: Pull complete 
a0529eb74f28: Pull complete 
23bff6dcced5: Pull complete 
2137cd2bcba9: Pull complete 
Digest: sha256:68efc9bbe07715c54ff30850aeb2e6f0d0b692af3c8dd40f13c0b179bfc0bc15
Status: Downloaded newer image for nvidia/cuda:latest
Fri Sep  6 02:23:11 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.87.00    Driver Version: 418.87.00    CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce RTX 208...  Off  | 00000000:84:00.0 Off |                  N/A |
| 31%   45C    P0     1W / 260W |      0MiB / 10989MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## 安装docker-compose
```bash
# 方法一:
curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# 方法二:
apt install python-pip
pip install docker-compose==1.21.2

# 修改docker daemon文件
root@ubuntu:~# vim /etc/docker/daemon.json 

{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```