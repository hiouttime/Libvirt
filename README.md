# Libvirt-Manager
该项目来自于kasuganosoras的[Libvirt](https://github.com//Libvirt-Manager)
在此项目之上进行修改、重构。使其更易用。
## 介绍
这是一个由PHP所写的Libvirt管理器。
它可以简单地帮你完成创建虚拟机、获取虚拟机信息
管理虚拟机硬盘等绝大多数自动出售KVM所需要的功能
且使用起来轻松简单。
## 依赖
控制Libvirt由SSH2模块完成，您需要安装SSH2模块
并使用一个有root权限的用户来使用。
如您不会安装SSH2模块可以参考我的[一篇文章](https://imrbq.cn/exp/62.html)

## 安装 Libvirt-Manager
直接下载`libvirt.php`到您的项目之中并引用即可。

## 使用
### 连接到服务器
在使用之前需要与服务器建立SSH连接。请确保程序处于能够正常连接到要进行虚拟化的机器上，且账号具有root权限。

```php
require_once __DIR__ . '/libvirt.php';
use libvirt_manager\Libvirt;
$Libvirt = new Libvirt();
$Libvirt->setHost("目标地址", 目标端口, "/data/vmdata/", "/data/vmdata/datapath");
//第三与第四个参数分别为虚拟机的存储位置以及数据盘的存储位置
$Libvirt->connect("root", "密码");
```
请尽量将该文件部署在虚拟化主机内，以保证执行效率。
SSH连接过程可能会较为缓慢。

### 创建虚拟机示例代码

下面的示例代码将会创建一个名为 Test 的虚拟机，并且为它分配 2 CPU 核心，2GB 内存。

```php
$Libvirt->createDisk("虚拟机磁盘名", "qcow2", "30G");
//在自动化售卖的过程中，创建系统磁盘的过程应省略，直接复制已经做好的系统模板即可。
$Libvirt->createVMXML("虚拟机名", 核心数, 内存大小(KiB), "虚拟机系统盘位置",数据盘大小(G) "要挂载的光盘", "hd", "network", "default", $Libvirt->randomMac(), "virbr0", 0, 0, VNC端口,$uuid);
$Libvirt->define("/data/libvirt/虚拟机名.xml");
$Libvirt->setPermission("虚拟机名");
$Libvirt->start("虚拟机名");
//启动虚拟机
```
创建部分有更多的选项，建议查看源码的 createVMXML 类来定义需求。
实际的售卖过程中应尽量减少耗时的操作。
数据盘如大于0则会自动创建一个数据盘并挂载好。
默认是适配2个socket,以适应Windows7这类家用系统仅支持2个socket的问题。
请不要在挂载数据盘的同时又挂载一个光盘，可能会导致无法启动(bus)
注意内存单位是KiB,2G内存=1024\*1024\*2
带宽限制单位是Kib,1MBps=8192Kbps=8Mbps 需要注意一下

### 获取虚拟机的信息
需要注意的是，如果系统语言为中文，那么返回的也会是中文
```php
$Libvirt->getInfo("虚拟机名");
```
同时库内提供了获取内网IP、VNC端口，密码的功能
本人写了一个简单的整理代码，可以方便地将信息输出为json串
```php
$getData = $Libvirt->getInfo(虚拟机名);
if (!empty($getData['名称'])) { //判断是不是中文
    $temp = array();
    $temp['Id'] = $getData['Id'];
    $temp['Name'] = $getData['名称'];
    $temp['UUID'] = $getData['UUID'];
    $temp['OS_Type'] = $getData['OS 类型'];
    $temp['status'] = $getData['状态'];
    switch ($temp['status']) {
        case '关闭':
            $temp['status'] = "stopped";
            break;
    }
    $temp['CPU_cores'] = $getData['CPU'];
    $temp['CPU_time'] = $getData['CPU 时间'];
    $temp['MaxMemory'] = $getData['最大内存'];
    $temp['UsedMemory'] = $getData['使用的内存'];
    $temp['KeepIt'] = ($getData['持久'] == "是"?true:false);
    $temp['AutoBoot'] = ($getData['自动启动'] == "禁用"?false:true);
    $temp['SavedMachine'] = ($getData['管理的保存'] == "否"?false:true);
    //是否已被保存(类似于Vmware的存盘)
    $temp['SafeMode'] = $getData['安全性模式'];
    $temp['SafeDOI'] = $getData['安全性 DOI'];
    $getData = $temp;
}
if (!empty($getData)) {
    $getData['vncInfo'] = $Libvirt->getVncInfo("虚拟机名");
    $getData['ipInfo'] = $Libvirt->getIP("虚拟机名");
}
```
$getData即为最后整理出来的虚拟机信息。
在写这段的时候并没有去找一台英文系统查阅，所以都是按照自己的思想进行的英文翻译
如有问题还请各位帮忙指出，感激不尽!
这段代码遵守GPL协议。

#### 启动虚拟机
```php
String start ( Name )
```
#### 正常停止虚拟机
```php
String shutdown ( Name )
```
#### 强制停止虚拟机
如果你的虚拟机出现了问题导致不能使用 shutdown 方法停止时，可以使用此方法强制结束虚拟机，但是可能会丢失数据。
```php
String destroy ( Name )
```
#### 获得虚拟机列表
你可以使用此方法获得所有已注册的虚拟机列表，它将会返回一个数组。
```php
String getList ()
```
#### 获得虚拟机信息
你可以使用此方法获得任何已注册的虚拟机信息，它将会返回一个数组。
```php
String getInfo ( Name )
```
#### 导出虚拟机的 XML 配置文件
此方法可以读取虚拟机的 XML 配置文件并返回
```php
String dumpxml ( Name )
```
### 设置虚拟机网卡
此方法可以设置虚拟机的网卡

第三个参数是布尔型的，如果赋值是 true，将会启用网卡，如果赋值是 false，将会禁用网卡。
```php
String setNetwork ( Server, Network name, Status )
```

#### 涉及磁盘操作的也许需要消耗很长时间，具体视磁盘性能而定，建议加一行代码 `set_time_limit(120)` 以防止脚本超时。

### 您可以在`Libvirt.php`文件内找到更多的信息。
也欢迎您对该项目进行优化修改~
本人学识不高，还请多多指教。
