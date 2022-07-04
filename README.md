# 				移动端AI模型部署相关

# 一、Android 内存零拷贝(8450 GPU/DSP)

## 1、FastRPC

FastRPC.md

## 2、数据卸载（零拷贝）资料

### ION & OpenCL

https://www.khronos.org/registry/OpenCL/extensions/qcom/cl_qcom_ion_host_ptr.txt

https://www.khronos.org/registry/cl/extensions/qcom/cl_qcom_ext_host_ptr.txt

[The Android ION memory allocator](https://lwn.net/Articles/480055/)

https://www.e-learn.cn/topic/3149907

```
// 移动端优化
https://zhuanlan.zhihu.com/p/429890621
// 签名
https://www.pudn.com/news/62971879bf399b7f354ba50d.html
// dsp blog
https://blog.csdn.net/u010658879?type=blog
// FastRPC基本使用
https://bbs.elecfans.com/jishu_1651374_1_1.html
// dsp芯片漏洞挖掘
https://www.4hou.com/posts/9GPD
// FastRPC 负载转移
https://www.elecfans.com/rengongzhineng/778673.html
```



#### 1、ION：

OpenCL中三种内存创建image的效率对比： [OpenCL中三种内存创建image的效率对比 - willhua - 博客园](https://www.cnblogs.com/willhua/p/10123398.html)

ION OpenCL: [高通Adreno使用ION buffer的OpenCL扩展 - 辰宸的备忘录](https://www.licc.tech/article?id=74)



#### 2、OpenCL:

```C%2B%2B
// 下载
// https://www.intel.com/content/www/us/en/developer/tools/opencl-sdk/choose-download.html
//安装
./install.sh
// 调试 https://blog.csdn.net/qq_28483731/article/details/68235383
gcc opencl_hello.c -lOpenCL 
```

#### 3、PRCMEM:

RPCMEM API 提供了一组 API，用于从 Android 端分配内存，目的是通过 FastRPC 调用与 DSP 共享内存。根据应用程序的需要，可以使用不同类型的内存和缓存选项。

与 Android ION 内存分配器相比，使用 RPCMEM API 的优势在于它既分配内存又将缓冲区注册到 FastRPC 框架。

rpcmem.h : file:///opt/wood/work/work_project/daotong/Hexagon_SDK/4.5.0.3/docs/doxygen/rpcmem/rpcmem_8h_source.html

```Groovy
blog:
//将任务加载到DSP上的主要机制称为FastRPC。将任务放到DSP上处理的过程，就是从CPU->FastRPC->DSP的调用过程，代码也是三部分：CPU代码 -> FastRPC idl代码 -> DSP代码
https://icode.best/i/02710137590345

// 有关Snapdragon Flight™平台（开发者版）的一些信息，
https://github.com/ATLFlight/ATLFlightDocs

// example:如何为由 AppsProc 修改并由 aDSP 修改以返回到 AppsProc 的缓冲区完成此操作：
https://developer.qualcomm.com/forum/qdn-forums/hardware/snapdragon-flight/33750

// src/fastrpc_apps_user.c - platform/external/fastrpc
https://android.googlesource.com/platform/external/fastrpc/+/8a30933a85303af3c4f7747d58b3ae85e0f55df4/src/fastrpc_apps_user.c
// 宏
// 分配与 ION_FLAG_CACHED 标志具有相同属性的内存
#define     RPCMEM_DEFAULT_FLAGS   1
// FastRPC 库尝试将使用此标志分配的缓冲区映射到所有当前和新 FastRPC 会话的远程进程。
#define     RPCMEM_TRY_MAP_STATIC   0x04000000
// 使用未缓存的内存
#define     RPCMEM_FLAG_UNCACHED   0
// 使用缓存内存
#define     RPCMEM_FLAG_CACHED   RPCMEM_DEFAULT_FLAGS
 
 // 枚举
// 支持的 RPCMEM 堆 ID
enum    rpc_heap_ids { 
RPCMEM_HEAP_ID_SECURE = 9, // 内存仅用于安全用例。
RPCMEM_HEAP_ID_CONTIG = 22, // 连续物理内存：可用内存非常有限 (< 8 MB)；推荐用于没有 SMMU 的子系统（sDSP 和 mDSP）
RPCMEM_HEAP_ID_SYSTEM = 25 // 非连续的系统物理内存：推荐用于不需要使用特定堆的所有用例；与带有 SMMU 的子系统（cDSP 和 aDSP）一起使用
}

// RPCMEM API 函数
/**
 * void     rpcmem_init (void)
 * void     rpcmem_deinit (void)
 * void *   rpcmem_alloc (int heapid, uint32 flags, int size)
 * static void *    rpcmem_alloc_def (int size)
 * void     rpcmem_free (void *po)
 * int  rpcmem_to_fd (void *po)
 * */


// rpcmem_alloc()
/**
 * 通过 ION 分配缓冲区并将其注册到 FastRPC 框架
 * heapid Heap ID 用于内存分配
 * flags ION 用于内存分配的标志
 * size Buffer 要分配的大小。
 * return 成功时指向缓冲区的指针；失败时为 NULL。
 * */
// example:
// 默认内存属性：2KB
rpcmem_alloc(RPCMEM_HEAP_ID_SYSTEM, RPCMEM_DEFAULT_FLAGS, 2048);
rpcmem_alloc_def(2048);

// 堆 22，未缓存，1 KB
rpcmem_alloc(22, 0, 1024);
rpcmem_alloc(22, RPCMEM_FLAG_UNCACHED, 1024);

// 堆 21，缓存，2 KB
rpcmem_alloc(21, RPCMEM_FLAG_CACHED, 2048);
#include <ion.h>
rpcmem_alloc(21, ION_FLAG_CACHED, 2048);

// 默认内存属性，但来自堆 18，4 KB
rpcmem_alloc(18, RPCMEM_DEFAULT_FLAGS, 4096);

// rpcmem_alloc_def()
/**
 * 使用默认设置分配缓冲区。
 * size 要分配的缓冲区的大小。
 * return 指向分配的内存缓冲区的指针。
 * */

//rpcmem_deinit()
/**
 * 取消初始化 RPCMEM 库。
 * 仅在不再需要 RPCMEM 库时调用此函数一次。
 * */

// rpcmem_free() 释放缓冲区并忽略无效缓冲区。

// rpcmem_init()
/**
 * 初始化 RPCMEM 库。
 * 在使用 RPCMEM 库之前只调用一次这个函数。
 * */


// rpcmem_to_fd() 返回关联的文件描述符。
/**
 * po RPCMEM 分配的缓冲区的 poData 指针。
 * return: 缓冲区文件描述符。
 * */
```



#### 4、IDL：

https://icode.best/i/02710137590345

IDL 语言是描述两个处理器之间接口的通用语言。 SDK 中支持的 IDL 版本经过调整，专门描述了应用处理器和使用 FastRPC 进行通信的 Hexagon DSP 之间的接口。

定义这些 CPU-DSP 接口的 IDL 文件使用 QAIC 工具编译以生成 C 头文件以及链接到 CPU 和 DSP 模块的存根和框架源文件，以使它们能够通过 RPC 调用进行通信。

将任务卸载到dsp流程：

1、在IDL定义一个对数据执行操作的API；（后期编译生成头文件和stub.c库、skel.c库）；

2、实现接口code（dsp）, 接口的 Hexagon 端实现的源代码，并被编译为共享对象。；

3、cpu端main函数调用1的接口。（申请ion 缓冲区， 调用 rpcmem api）



## 3、Hexagon SDK 4.5.0.3 数据卸载到DSP

### 3-1、Hexagon SDK 3.5.2 examples

qpm：高通包管理工具

文档：https://createpoint.qti.qualcomm.com/tools/#document/8643/1336631

下载安装：https://createpoint.qti.qualcomm.com/tools/#suite/7382/63942

```SQL
sudo dpkg -l | grep qpm
# 登录
qpm-cli --login
# 打开
qpm-cli --catalog-refres
```

#### 3-1-1、calculator例子

### 

```Python
# cd
/root/Qualcomm/Hexagon_SDK/3.5.2/examples/common/calculator

# 交叉编译 
make tree V=android_Release_aarch64 CDSP_FLAG=1
make tree V=hexagon_Release_dynamic_toolv83_v66 VERBOSE=1
/*** printf
---Allocate 1024 bytes from ION heap
---Creating sequence of numbers from 0 to 255

---Compute sum locally
---Sum = 32640

---Allocate 1024 bytes from ION heap
---Creating sequence of numbers from 0 to 255

---Compute sum on the DSP---ifdef __hexagon
---Sum = 32640
############################################################
Summary Report 
############################################################
Pass: 2
Undetermined: 0
Fail: 0
Did not run: 0
***/

# 实机测试:push Android 端测试程序
adb root
adb wait-for-device root
adb remount
adb push android_Release_aarch64/ship/calculator /vendor/bin/
adb shell chmod +x /vendor/bin/calculator
# push DSP 端算法库文件
adb push hexagon_Release_dynamic_toolv83_v66/ship/libcalculator_skel.so /vendor/lib/rfsa/adsp/

# 运行测试
adb shell
cd /vendor/bin
./calculator 1 255
/*** result:
---Starting calculator test

---Allocate 1020 bytes from ION heap
---Creating sequence of numbers from 0 to 254

---Compute sum locally
---Sum = 32385
---Success
***/
./calculator 0 255
/*** result:
---Starting calculator test

---Allocate 1020 bytes from ION heap
---Creating sequence of numbers from 0 to 254

---Compute sum on the DSP
---Error: compute on DSP failed, nErr = 44

---Usage: ./calculator <1/0 run locally> <uint32 size>
***/
```



### 3-2、Hexagon SDK 4.5.0.3

#### 3-2-1、安装

```python
# 登录
qpm-cli --login
# 查看信息
qpm-cli --info hexagonsdk4.x
# 激活license
pm-cli --license-activate hexagonsdk4.x
# 下载安装对应版本
qpm-cli --install hexagonsdk4.x --version 4.5.0.3
/** result:
* SUCCESS: Installed JRE4.x.AddOn at /local/mnt/workspace/Qualcomm/Hexagon_SDK/4.5.0.3
* ...
**/
```

```
source setup_sdk_env.source
```

#### 3-2-2、4.5.0.3 - asyncdspq_example

交叉编译：

```
make android BUILD=Debug
make hexagon BUILD=Debug DSP_ARCH=v69
```

push到设备：

```c
adb push android_Debug_aarch64/ship/fcvqueuetest /vendor/bin/
adb shell chmod 777 /vendor/bin/fcvqueuetest
adb push android_Debug_aarch64/libasyncdspq.so /vendor/lib64/
// adb shell mkdir -p /vendor/lib/rfsa/dsp/sdk
adb push hexagon_Debug_toolv85_v69/ship/* /vendor/lib/rfsa/dsp/sdk/
adb push hexagon_Debug_toolv85_v69/libasyncdspq_skel.so /vendor/lib/rfsa/dsp/sdk/
```

环境变量：

```
export LD_LIBRARY_PATH=/vendor/lib64/:$LD_LIBRARY_PATH 
export ADSP_LIBRARY_PATH="/vendor/lib/rfsa/dsp/sdk;"
```

代码解析：

```c
fcvqueue.c: 
CPU 端的实现。本节的其余部分主要涉及此文件中的函数。
对应调用 fcvqueue_dsp.idl的接口。（init, destroy, map, unmap）
fcvqueue_dsp_imp.c: 
以 DSP 端实现为例。包含从 fcvqueue.c 调用的 FastRPC 方法和消息队列客户端代码。

fcvqueuetest.c: 
测试应用。从这里开始遵循调用顺序。可在这找到调用 fastcv_dilate3x3等图像处理方法
fcvqueuetest_imp.c
dsp端图像处理实现；
```



### 3-3、cal_w

```Bash
make android BUILD=Debug

make hexagon BUILD=Debug DSP_ARCH=v68

# 模拟器测试
$DEFAULT_HEXAGON_TOOLS_ROOT/Tools/bin/hexagon-sim -mv68 --simulated_returnval --usefs hexagon_Debug_toolv85_v68 --pmu_statsfile hexagon_Debug_toolv85_v68/pmu_stats.txt hexagon_Debug_toolv85_v68/cal_w_q --
```

在模拟器上模拟可以不需要到编译android,直接 make hexagon BUILD=Debug DSP_ARCH=v68， 然后模拟器测试。编译android端需要编写Android 可执行文件的源代码，如本例中的cal_w_main.c。

Code：

```Swift
cal_w/inc:
----cal_w.idl // idl 接口代码
cal_w/src:
----verify.h   //
----cal_w_test.h // cal_w_test() 函数定义 cal_w_test(int runLocal, int domain_id, int num, bool is_unsignedpd_enabled)
----cal_w_test.c // cal_w_test() 函数实现，HLOS 端测试函数的源代码， 使用fastrpc分配共享内存缓冲区，通过idl接口调用实现将计算卸载到dsp
----cal_w_imp.c   //Hexagon 端实现的源代码
----cal_w_test_main.c  //模拟器调用 cal_w_test()

----cal_w_main.c // Android 可执行文件的源代码，它调用 HLOS 端的计算器存根以将计算任务卸载到 DSP 上。
```

设备签名：

file:///local/mnt/workspace/Qualcomm/Hexagon_SDK/4.5.0.3/docs/tools/sign.html#use-signerpy

adb -s 85c6ccb wait-for-device push ./testsigs/testsig-0x583925b9.so /vendor/lib/rfsa/adsp

push到设备测试：

```C
adb push android_Debug_aarch64/ship/cal_w /vendor/bin/
adb shell chmod 777 /vendor/bin/cal_w

adb push android_Debug_aarch64/ship/libcal_w.so /vendor/lib64/
adb push hexagon_Debug_toolv85_v68/ship/libcal_w_skel.so /vendor/lib/rfsa/dsp/sdk/

// cpu 成功
./cal_w -d 0 -U 0 -r 1 -n 256
// dsp 失败： 解决办法，仔细按照官方文档设置环境变量。
./cal_w -d 0 -U 0 -r 0 -n 256
```



### 3-4、image_w

Code:

```c
// 配置文件: 修改配置文件，去掉queuetest、queueperf相关配置
image_w/
--all.mak
--android_deps.min
--android.min
--CMakeLists.txt
--cpu.min
--hexagon_deps.min
--hexagon.min
--Makefile
--UbuntuARM.min  // 有以上配置文件make hexagon 可以成功, make android 也可以成功


image_w/inc:
----fcvqueuetest.idl // idl 接口代码 fastcv_dilate3x3/fastcv_gaussian3x3/fastcv_median3x3/set_clocks
----fcvqueue_dsp.idl  // idl 接口 init/destroy/map/unmap
----fcvqueue.h    // 基于异步 FastCV 的图像处理操作队列示例。主 API，仅在应用程序 CPU 上可用。
----fcvqueue_common.h  // 基于异步 FastCV 的图像处理操作队列示例。常见的应用程序(apps)/DSP 定义。

image_w/src:
----fcvqueue.c // 基于异步 FastCV 的图像处理操作队列示例 - CPU 端实现
/**
* fill_random(in, size);  
* FCVQUEUE_OP_DILATE3X3
*/
----fcvqueuetest.c // 示例异步基于 FastCV 的图像处理操作队列 - 测试应用程序 int main() -- open_session() -- while ( loops-- ) 
----fcvqueuetest_imp.c   //
----fcvqueue_dsp_imp.c  // 示例异步基于 FastCV 的图像处理操作队列 - DSP 端实现   fcvqueue_internal_t
```

环境变量：

```Python
export LD_LIBRARY_PATH=/vendor/lib64/:$LD_LIBRARY_PATH ADSP_LIBRARY_PATH="/vendor/lib/rfsa/dsp/sdk;"
```

push到设备运行：

```Apache
make android BUILD=Debug
adb push android_Debug_aarch64/ship/fcvqueuetest /vendor/bin/
adb shell chmod 777 /vendor/bin/fcvqueuetest
adb push android_Debug_aarch64/libasyncdspq.so /vendor/lib64/

#  hexagon 在该板子上需要编译 v69 版本，相关需要的so文件可参考5-2
make hexagon BUILD=Debug DSP_ARCH=v69
adb push hexagon_Debug_toolv85_v69/ship/* /vendor/lib/rfsa/dsp/sdk/
adb push hexagon_Debug_toolv85_v69/libasyncdspq_skel.so /vendor/lib/rfsa/dsp/sdk/
```

动态链接库的编译：

```Makefile
# cpu.min
BUILD_LIBS = fcvqueuetest $(SHARED)

# hexgon.min 尝试过添加如下：
## fcvqueuetest test
#libfcvqueuetest_QAICIDLS += inc/fcvqueuetest
#libfcvqueuetest_C_SRCS += $(OBJ_DIR)/fcvqueuetest_skel \
#                               src/fcvqueuetest
#libfcvqueuetest_DLLS += libasyncdspq_skel libfcvqueue_dsp_skel libfcvqueuetest_skel
```



### 3-5、dspmenman

1、编译(先删除设备上原有相关so库) 与可执行文件测试：

```c
source setup_sdk_env.source

adb root
adb wait-for-device
adb remount

make android BUILD=Debug
adb push android_Debug_aarch64/ship/fcvqueuetest /vendor/bin/
adb shell chmod 777 /vendor/bin/fcvqueuetest
adb push android_Debug_aarch64/libasyncdspq.so /vendor/lib64/

# 如无 hexagon 编译得到的so文件
/** error:
DSP domain is not provided. Retrieving DSP information using Remote APIs.
Attempting to run on unsigned PD on domain 3
ERROR 0x80000406: unable to create fastrpc session on CDSP
ERROR 0x80000406: Test FAILED
**/

#  hexagon 在该板子上需要编译 v69 版本，相关需要的so文件可参考3-2
make hexagon BUILD=Debug DSP_ARCH=v69
adb push hexagon_Debug_toolv85_v69/ship/* /vendor/lib/rfsa/dsp/sdk/

./fcvqueuetest -d 3 -t 1 -U 1 -i /vendor/bin/input1920-1080.yuv -o output1920-1080.yuv
./fcvqueuetest -d 3 -t 1 -U 1 -i "/vendor/bin/input1920-1080.yuv" -o "output1920-1080.yuv"
```



# 二、FastCV库交叉编译

## 1、前置：

dspCV 库：*从 SDM660 开始不再推荐使用这些远程* *API**。仍然支持它们以实现向后兼容性。*

FastCV: FastCV库包含许多图像处理 API，针对 Hexagon DSP 进行了优化

3.5.2 版本examples: 

file:///root/Qualcomm/Hexagon_SDK/3.5.2/docs/FastCV/Applications_Computer%20Vision.html

相关资料：

```markdown
[Qualcomm Computer Vision SDK Tools & Resources](https://developer.qualcomm.com/software/qualcomm-computer-vision-sdk/tools)

https://developer.qualcomm.com/software/qualcomm-computer-vision-sdk 

https://developer.qualcomm.com/forums/software/qualcomm-computer-vision

SNPE 中使用fastcv: 包括 fastcv.h 和 libfastcv.a

file:///home/**/HDD/wood/work/work_project/daotong/Hexagon_SDK/4.5.0.3/addons/compute/docs/fastcv/fastcv.html

Git RB5 examples: https://github.com/quic/sample-apps-for-Qualcomm-Robotics-RB5-platform/tree/master/FastCV-Samples

```



## 2、环境配置：

下载sdk： [Qualcomm Computer Vision SDK v1.7.1 for Android - Linux 安装程序](https://developer.qualcomm.com/download/fastcv/fastcv-sdk-linux.bin?referrer=node/7332)

- ubuntu20.04未安装成功，ubuntu18可成功安装, 后将安装的文件拷贝到ubuntu20.04上。

CMakeLists.txt:

```bash
cmake_minimum_required(VERSION 3.14.3)

project(fastcv_snpe C CXX)

set(CMAKE_DEBUG_TARGET_PROPERTIES
        INCLUDE_DIRECTORIES
        COMPILE_DEFINITIONS
        POSITION_INDEPENDENT_CODE
        CONTAINER_SIZE_REQUIRED
        LIB_VERSION
    )

set(ANDROID_CMAKE_ROOT /local/mnt/workspace/Qualcomm/Hexagon_SDK/4.5.0.3/tools/android-ndk-r19c/build/cmake)
include(${ANDROID_CMAKE_ROOT}/android.toolchain.cmake)

set(ANDROID_TOOLCHAIN /local/mnt/workspace/Qualcomm/Hexagon_SDK/4.5.0.3/tools/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin)
set(CMAKE_CXX_COMPILER ${ANDROID_TOOLCHAIN}/armv7a-linux-androideabi21-clang++)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/inc)

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

add_executable(fastcv_snpe src/test_fastcv.cpp)

# 链接库文件
TARGET_LINK_LIBRARIES(fastcv_snpe 
	${CMAKE_CURRENT_SOURCE_DIR}/lib/libfastcv.a
    log)
```

交叉编译&调试：

```bash
export NDK_HOME=/local/mnt/workspace/Qualcomm/Hexagon_SDK/4.5.0.3/tools/android-ndk-r19c
# cmake .. -DDEBUG=ON -DANDROID_NDK=$NDK_HOME
cmake ..
make

adb push bin/fastcv_snpe /vendor/bin/
adb shell chmod 777 /vendor/bin/fastcv_snpe
adb push lib/libfastcv.a /vendor/lib64/
adb pull /vendor/bin/dst960-540.yuv ./src/
```

## 3、fastcv api:

```css
FastCV resize/downscale APIs are for greyscale images only. You can extract the color channels (fcvChannelExtract...), resize individual channels, and then combine the channels (fcvChannelCombine...) again into a color image, but the performance will not be optimal. You can experiment to see if it meets your needs.
FastCV 调整大小/缩小 API 仅适用于灰度图像。您可以提取颜色通道 (fcvChannelExtract...)，调整各个通道的大小，然后将通道 (fcvChannelCombine...) 再次组合成彩色图像，但性能不会最佳。您可以进行实验，看看它是否满足您的需求

Mat 与 数组转换： https://wenku.baidu.com/view/9dfb20aacf22bcd126fff705cc17552707225e67.html
从rgb转yuv出现问题 ：https://www.csdn.net/tags/OtDaYgzsMDQxNzYtYmxvZwO0O0OO0O0O.html
YUV420 与Mat互转 ： https://blog.csdn.net/zhizhengguan/article/details/108767736

```



# 三、SNPE库交叉编译

## 1、本机环境配置：

用于模型转换、snpe工具测试。创建虚拟环境env_snpe

前置环境：Ubuntu20.04 安装 Anaconda3； conda环境（python=3.6）； pytorch; 

~~~SQL
# 1、Anaconda3
Anaconda3-5.3.1-Linux-x86_64.sh
下载地址:https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/
bash Anaconda3-5.3.1-Linux-x86_64.sh   https://blog.csdn.net/qq_48020679/article/details/118391084
一路yes,直到提示信息“Do you wish to proceed with the installation of Microsoft VSCode? [yes|no]”，输入no；
环境变量：echo 'export PATH="~/anaconda3/bin":$PATH' >> ~/.bashrc
source ~/.bashrc

# 2、 创建conda环境：
```
conda create -n env_snpe python=3.6
conda activate env_snpe
sudo apt-get update
sudo apt-get install python3.6 python3.6-dev python3-pip
python3 -m pip install --upgrade pip
```

# 3、安装pytorch：
pip install torch==1.9.1+cpu torchvision==0.10.1+cpu torchaudio==0.9.1 -f https://download.pytorch.org/whl/torch_stable.html
ipython问题：https://blog.csdn.net/yuebowhu/article/details/102706068, 用conda安装:conda install ipython
~~~

SNPE： snpe-1.59.0.3230

```SQL
pip install onnx==1.6.0 -i https://mirrors.bfsu.edu.cn/pypi/web/simple

source snpe-X.Y.Z/bin/dependencies.sh
source snpe-X.Y.Z/bin/check_python_depends.sh
source bin/envsetup.sh -o /root/anaconda3/envs/env_snpe/lib/python3.6/site-packages/onnx

pip3 install onnx-simplifier
snpe-onnx-to-dlc --input_network test/vgg16.onnx  --output_path test/vgg.dlc
# https://developer.qualcomm.com/sites/default/files/docs/snpe/tutorial_onnx.html
wget https://s3.amazonaws.com/model-server/inputs/kitten.jpg
wget https://s3.amazonaws.com/onnx-model-zoo/synset.txt

python create_VGG_raws.py -i ../../../../test/data/ -d ../../../../test/data/cropped

python create_file_list.py --help
python create_file_list.py -i ../../../../test/data/cropped/ -o ../../../../test/data/raw_list.txt -e *.raw

snpe-net-run --input_list ../test/data/raw_list.txt --container ../test/vgg.dlc --output_dir ../test/data/output

python models/VGG/scripts/show_vgg_classifications.py -i ../test/data/raw_list.txt -o ../test/data/output/ -l ../test/data/synset.txt 
```

## 2、SNPE tools

模型转换： snpe-onnx-to-dlc：batch 必须静态

模型可视化：snpe-dlc-viewer

模型量化：snpe-dlc-quantize

模型运行测试：snpe-net-run

```bash
snpe-onnx-to-dlc --input_network mobilenetv2_100_1_3_224_224.onnx --input_dim inputs 1,3,224,224 --output_path ./mobilenetv2_100_1_3_224_224.onnx.dlc

snpe-dlc-viewer -i mobilenetv2_100_1_3_224_224.onnx.dlc -s mobilenetv2_100_1_3_224_224.onnx.dlc.html

snpe-onnx-to-dlc --input_network mobilenetv2_100_1_3_224_224.onnx --input_dim inputs 2,3,224,224 --output_path ./mobilenetv2_100_2_3_224_224.onnx.dlc

snpe-dlc-viewer -i mobilenetv2_100_2_3_224_224.onnx.dlc -s mobilenetv2_100_2_3_224_224.onnx.dlc.html

snpe-dlc-quantize --input_dlc mobilenetv2_100_1_3_224_224.onnx.dlc --input_list ../../test/data/raw_list.txt
snpe-dlc-viewer -i mobilenetv2_100_1_3_224_224.onnx_quantized.dlc -s mobilenetv2_100_1_3_224_224.onnx_quantized.dlc.html
python ../../snpe-1.59.0.3230/models/inception_v3/scripts/create_inceptionv3_raws.py -i back_pack/ -d data_raw/ -s 224

python ../../snpe-1.59.0.3230/models/inception_v3/scripts/create_file_list.py -i data_raw/ -o raw_list.txt -e *.raw

snpe-net-run --input_list raw_list.txt --container ../models/mobilenetv2_100_1_3_224_224.onnx.dlc --output_dir ./output
python ../../test/show_inceptionv3_classifications.py -i raw_list.txt  -o ./output/ -l synset.txt 

# quan
snpe-net-run --input_list raw_list.txt --container ../models/mobilenetv2_100_1_3_224_224.onnx_quantized.dlc --output_dir ./output_quan
python ../../test/show_inceptionv3_classifications.py -i raw_list.txt  -o ./output_quan/ -l synset.txt 

# bs=2
snpe-net-run --input_list raw_list.txt --container ../models/mobilenetv2_100_2_3_224_224.onnx.dlc --output_dir ./output_bs2
python ../../test/show_inceptionv3_classifications.py -i raw_list.txt  -o ./output_bs2/ -l synset.txt
```

## 3、api

```
file:///root/wood/SNPE/snpe-1.59.0.3230/doc/html/cplus_plus_tutorial.html#cpp_tutorial_load_input_itensors
```



# 四、OpenCV4.6.0 库交叉编译



