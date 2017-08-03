---
title: webrtc学习1-全平台编译
date: 2017-08-03 14:31:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - webrtc
---

## 0、前提
因为webrtc很多依赖源是在墙外，需要翻墙工具。


## 1、安装depot tools
是一套脚本，用于管理代码签出和审查。

参考[Install depot_tools](http://dev.chromium.org/developers/how-tos/install-depot-tools)

### Windows
+ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
+ 设置depot_tools到PATH环境变量

### Linux（Android）/Mac（iOS）
+ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
+ 把depot_tools目录加入PATH：export PATH=`pwd`/depot_tools:"$PATH"

<!--more-->

## 2、安装依赖软件

### Windows
+ 安装”Visual Studio 2015 Update 3“，其他版本都不受官方支持。
+ 操作系统必须是Windows 7 x64及以上版本，x86操作系统都不支持。
+ 安装VS2015时必须有下列组件：
    - Visual C++, which will select three sub-categories including MFC
    - Universal Windows Apps Development Tools > Tools
    - Universal Windows Apps Development Tools > Windows 10 SDK (10.0.10586)

### Linux
+ 参考后面

### Android
+ 安装Java OpenJDK
``` bash
$ sudo apt-get install openjdk-7-jdk
$ sudo update-alternatives --config javac
$ sudo update-alternatives --config Java
$ sudo update-alternatives --config javaws
$ sudo update-alternatives --config javap
$ sudo update-alternatives --config jar
$ sudo update-alternatives --config jarsigner
```

### Mac（IOS）
+ 安装最新XCode

## 3、同步源码
参考[Native Code Development](https://webrtc.org/native-code/development/)

先创建源码目录
``` bash
mkdir webrtc-checkout
cd webrtc-checkout
```

### Windows
``` bash
fetch --nohooks webrtc #第一次拉去时使用，后续更新不需要
gclient sync
```

### Linux
``` bash
export GYP_DEFINES="OS=linux"
fetch --nohooks webrtc_android
gclient sync

cd src
./build/install-build-deps.sh
```

### Android
``` bash
export GYP_DEFINES="OS=android"
fetch --nohooks webrtc_android
gclient sync

cd src
. build/install-build-deps-android.sh
```

### Mac
``` bash
export GYP_DEFINES="OS=mac"
fetch --nohooks webrtc_ios
gclient sync
```

### IOS
``` bash
export GYP_DEFINES="OS=ios"
fetch --nohooks webrtc_ios
gclient sync
```

## 4、Working with Release Branches
To see available release branches, run:
``` bash
git branch -r
```

**NOTICE**: If you only see your local branches, you have a checkout created before our switch to Git (March 24, 2015). In that case, first run:

``` bash
cd /path/to/webrtc/src
gclient sync --with_branch_heads
git fetch origin
```

You should now have an entry like this under [remote “origin”] in .git/config:

``` bash
fetch = +refs/branch-heads/*:refs/remotes/branch-heads/*
```

To create a local branch tracking a remote release branch (in this example, the 43 branch):

``` bash
git checkout -b my_branch refs/remotes/branch-heads/43
gclient sync
```

## 5、编译

### 生成ninjia项目

#### Windows/Linux
``` bash
#生成debug版ninja项目文件：
gn gen out/Default
#生成release版ninja项目文件：
gn gen out/Default --args='is_debug=false'
#清空ninja项目文件：
gn clean out/Default
```

#### Android
``` bash
#使用gn生成:
gn gen out/Default --args='target_os="android" target_cpu="arm"'
#生成ARM64版：
gn gen out/Default --args='target_os="android" target_cpu="arm64"'
#生成32位 x86版：
gn gen out/Default --args='target_os="android" target_cpu="x86"'
#生成64位 x64版：
gn gen out/Default --args='target_os="android" target_cpu="x64"'
```

#### Mac
``` bash
#使用gn生成：
gn gen out/Debug-mac --args='target_os="mac" target_cpu="x64" is_component_build=false'
```

#### IOS
``` bash
#生成ARM版：
gn gen out/Debug-device-arm32 --args='target_os="ios" target_cpu="arm" is_component_build=false'
#生成ARM64版：
gn gen out/Debug-device-arm64 --args='target_os="ios" target_cpu="arm64" is_component_build=false'
#生成32位模拟器版：
gn gen out/Debug-sim32 --args='target_os="ios" target_cpu="x86" is_component_build=false'
#生成64位模拟器版：
gn gen out/Debug-sim64 --args='target_os="ios" target_cpu="x64" is_component_build=false'
```

### 编译源码
#### Windows/Linux/Android/Mac/IOS
``` bash
ninja -C out/Default
```

## 6、备注
### windows编译脚本备份
``` bash
set DEPOT_TOOLS_WIN_TOOLCHAIN=0

mkdir webrtc-checkout
cd webrtc-checkout

Windows:
fetch --nohooks webrtc #第一次
gclient sync

cd src

#生成编译ninja项目文件：
#debug:
gn gen out/Default
ninja -C out/Default

#release:
gn gen out/Release "--args=is_debug=false target_cpu=\"x86\""
ninja -C out/Release


#生成vs工程，直接用vs2015打开，方便看源码
gn gen out/msvc --ide="vs2015" ----no-deps
#生成vs工程，x86 release
gn gen out/msvc --ide="vs2015" "--args=is_debug=false target_cpu=\"x86\""
```

### depot_tools更新失败

出现类似如下错误
``` bash
Ensuring CIPD client is up-to-date
GET https://chrome-infra-packages.appspot.com/_ah/api/repo/v1/instance/resolve?version=bccdb9a605037e3dd2a8a64e79e08f691a6f159d&package_name=infra%2Ftools%2Fcipd%2Fwindows-amd64
Failed to fetch https://chrome-infra-packages.appspot.com/_ah/api/repo/v1/instance/resolve?version=bccdb9a605037e3dd2a8a64e79e08f691a6f159d&package_name=infra%2Ftools%2Fcipd%2Fwindows-amd64
```
+ 开启ShadowSocket
+ 访问[URL](https://chrome-infra-packages.appspot.com/_ah/api/repo/v1/instance/resolve?package_name=infra%2Ftools%2Fluci%2Fvpython%2Fwindows-amd64&version=git_revision%3A5cf65fdf804a9b3f3023f79d5b3cab2a88ccd09e)
返回如下内容表示可以正常翻墙
``` json
{
 "status": "SUCCESS",
 "instance_id": "d8a0231b483ecdf618cbfd191ea2dd49c0f9b726",
 "kind": "repo#resourcesItem",
 "etag": "\"pR8slN9LauG_o1VyIXSmbfx7GCI/-JA0lET_3WnwfrRbDzJz7f_ejjI\""
}
```

+ 设置git netsh http https代理
``` bash
#Git的代理设置
git config --global http.proxy http://127.0.0.1:1080
git config --global https.proxy https://127.0.0.1:1080

#winhttp的代理设置（需要管理员权限）
netsh winhttp set proxy 127.0.0.1:1080

#cipd_client代理
set HTTP_PROXY=http://127.0.0.1:1080
set HTTPS_PROXY=https://127.0.0.1:1080
```

+ 还原代理
``` bash
git config --global --unset http.proxy
git config --global --unset https.proxy

netsh winhttp set proxy reset

set HTTP_PROXY=
set HTTPS_PROXY=

```

### gclient sync此应用无法在你电脑上运行

原因是.cipd_client.exe下载错误，删除depot_tools下缓存的.cipd_client.exe文件，重新执行gclient sync。
