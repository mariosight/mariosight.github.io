---
title: ndnSIM仿真安装以及使用
tags:
  - ndn
date: 2024-10-10T11:43:11+08:00
draft: false
---

### ndnSIM 搭建以及使用

##### 安装准备

```bash
sudo apt install build-essential libsqlite3-dev libboost-all-dev libssl-dev git python-setuptools castxml
sudo apt install python-dev python-pygraphviz python-kiwi python-gnome2 ipython libcairo2-dev python3-gilibgirepository1.0-dev python-gi python-gi-cairo gir1.2-gtk-3.0 gir1.2-goocanvas-2.0 python-pip
pip install pygraphviz pycairo PyGObject pygccxml
sudo apt-get install graphviz libgraphviz-dev graphviz-dev pkg-config
pip install pygraphviz
```

##### 下载源码

现在任意文件创建个 ndnsim 文件夹

```bash
mkdir ndnSIM2.7
```

然后进入该文件夹下，进行源码的下载

```C
cd ndnSIM2.7
git clone https://github.com/named-data-ndnSIM/ns-3-dev.git ns-3
git clone https://github.com/named-data-ndnSIM/pybindgen.git pybindgen
git clone https://github.com/named-data-ndnSIM/ndnSIM.git ns-3/src/ndnSIM
```

##### 修改对应版本号

这里的命令依靠 git 实现，能到这一步肯定都安装好了，就不多说了

首先进入到 ndnSIM 的核心源码地带

```bash
cd ns-3/src/ndnSIM
```

通过 git checkout 修改版本号

```bash
git checkout ndnSIM-2.7
```

也可以通过 git tag 查看所有的版本号，然后修改为想要安装的版本

然后安装 NFD 和 ndn-cxx 模块

```bash
git submodule update --init
```

接下来进入到 ndnSIM2.7/ns-3 文件下进行版本的修改

```bash
git checkout ndnSIM-ns-3.29
```

接下对 pybindgen 的版本进行修改

```bash
cd ndnSIM2.7/pybindgen
git checkout 0.19.0
```

然后在此目录下安装安装该 python 模块

```bash
sudo python setup.py install
```

##### 可视化出现问题

```bash
PyViz visualizer              : not enabled (Python Bindings are needed but not enabled)
Python Bindings               : not enabled (PyBindGen missing)
```

这个问题其实很好解决，执行一下

```bash
pip install pybindgen
```

注:以下代码若不知道存放路径,运行 pip show pybindgen 即可，或者重新运行一遍上面的代码

```bash
./waf -d debug configure --with-pybindgen=存放路径//ex：./waf -d debug configure --with-pybindgen=/home/antl417/anaconda3/lib/python3.8/site-packages
```

```bash
#在此虚拟机中应该是下面的
./waf -d debug configure --with-pybindgen=/home/yin/.local/lib/python3.8/site-packages
```

然后执行

```bash
./waf configure  --with-pybindgen=/home/yin/.local/lib/python3.8/site-packages --enable-examples
./waf
./waf --run ndn-simple --vis
```

正常情况直接编译完成，出现错误的话请看下面的修改提示

![image-20231211144052079](https://cdn.jsdelivr.net/gh/mariosight/image/picture/image-20231211144052079.png)

进入到可视化模块下将‘file=’删除

```bash
cd  ndnSIM2.7/ns-3/src/visualizer/visualizer
```

进入 base.py 文件，修改保存即可
![](https://cdn.jsdelivr.net/gh/mariosight/image/picture/202312111100622.png)
成功编译：
![](https://cdn.jsdelivr.net/gh/mariosight/image/picture/202312111100623.png)
第二种错误可能出现在运行时加上–vis 可视化模块时，如下图所示
![](https://cdn.jsdelivr.net/gh/mariosight/image/picture/Pasted%20image%2020231211140335.png)
这种情况下，还是进入刚才的那个文件夹，修改 hub.py 文件  
![](https://cdn.jsdelivr.net/gh/mariosight/image/picture/202312111100624.png)
将 from.import 注释，修改为 import core，再次运行就可以了

第三个错误也可能是出现在加上-vis 的情况，如下图所示：
![](https://cdn.jsdelivr.net/gh/mariosight/image/picture/202312111100625.png)
安装 gi.cairo 即可解决：

```bash
sudo apt-get install gi.cairo
```

##### 成功启动

然后执行

```bash
./waf configure  --with-pybindgen=/home/yin/.local/lib/python3.8/site-packages --enable-examples
./waf
./waf --run ndn-simple --vis
```

启动之后的 simple 如下图所示：
![](https://cdn.jsdelivr.net/gh/mariosight/image/picture/202312111100626.png)

##### 参考教程

使用教程：[ndnSIM 的使用教程-CSDN 博客](https://blog.csdn.net/qq1187239259/article/details/115296651)

安装流程：[在 Ubuntu 安装 ndnSIM_ndnsim20 安装教程-CSDN 博客](https://blog.csdn.net/qq_44001007/article/details/107575203)

可视化出现问题解决方法：[NS3 可视化问题及解决办法\_ns3.34 python binding-CSDN 博客](https://blog.csdn.net/m0_49448331/article/details/117454644)
