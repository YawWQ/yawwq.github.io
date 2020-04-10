---
layout: post
title:  "在ARC上使用GPU资源运行Python Tensorflow (University of Leeds)"
date:   2020-04-10"
excerpt: "ARC3, part of the High Performance Computing facilities at the University of Leeds, UK"
tag:
- ARC
- Linux
comments: false
---

# 0. 

用的过程中踩了点坑，总结一下下方便查阅。

本文的示例环境：

- ARC 3

- MAC OS X

- Python 3.6

- Tensorflow-gpu 1.12

一些命令在不同版本中使用时可能会报错，这时需要登录[ARC官网](https://arc.leeds.ac.uk/using-the-systems/getting-started/)查询一下。

# 1. 申请

[ARC申请](https://arc.leeds.ac.uk/apply/getting-an-account/)

申请的过程中照常填写自己的个人信息就可以，除了注意学校提供HPC给学生是为了产出项目与论文等，**所以要填一下自己项目的supervisor，然后请他/她帮忙通过一下申请。**

# 2. 登录

申请好之后就可以在终端通过ssh远程访问了，访问时需要挂上学校vpn，下面是登录命令：

    $ ssh -Y username@arc3.leeds.ac.uk

终端会提示输出密码，之后就会打印ARC的welcome页面（文字）。

# 3. 虚拟环境

在ARC中与DEC-10一样，不可以自己卸载或安装很多东西，所以这时需要通过Anaconda的虚拟环境来调试代码。

用module命令加载Anaconda：

    module load anaconda

有些版本的ARC会报错不存在，这时需要先手动添加一下Anaconda：

    module add anaconda
    
下面使用虚拟环境的命令应该比较熟悉了：

创建环境：

    source activate base

退出环境：

    conda deactivate   
    
值得注意的是，一般刚登录ARC，用户位于home目录下的位置，也就是/home/home0/username，然而这个目录的存储容量很小，只有5G：

![FileSystem](https://yawwq.github.io/assets/img/ARC/filesystem.png)

所以我们需要指定在nobackup下创建虚拟环境，虽然要注意**unused files are expired after 90 days**（但那时我应该已经不在学校了）：

    conda create -p /nobackup/username/envname python=3.6
    
其中username就是我们平时常用的那个，而envname就是你自己指定的环境名。我不记得conda有没有这个权限，要不要先提前创建文件夹了，如果报错就先“cd /nobackup”再“mkdir”创建一下。

查看Anaconda中的虚拟环境：

    conda env list

现在要做的就是安装TensorFlow，然而TF1.5之前的版本cpu与gpu是分开的，
亲测如果没下gpu版本，系统会显示TensorFlow用的是cpu。

安装TensorFlow：

    conda install tensorflow-gpu=1.12
    
    
# 4. 使用GPU资源

可以把bash文件提交给gpu node运行，也可以用qrsh命令直接申请一下session，这里以后者为例。

ARC3中有两个版本的GPU，一个是k80，一个是p100，这里和官网一样以k80为例：

    $ qrsh -l h_rt=2:0:0,coproc_k80=1 -pty y bash
    
其中h_rt=2:0:0是指申请两个小时的session，可以增加，但是增加的话，申请通过的概率将变小（真小气呀）。

详细见
[coproc_k80](https://arc.leeds.ac.uk/?s=coproc_k80)

过了一阵，申请成功的话不会有任何提示，申请失败会提示try again later。

官网教程显示运行代码前要先加载一下cuda：

    module load cuda
    
确认分配成功：

    nvidia-smi -L

还可以查看一下GPU使用率，每3秒更新一次，按ctrl c退出：

    watch -n 3 nvidia-smi
    
如果想确认tensorflow是不是在用GPU跑，可以添加代码查看确认一下：

    sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))

注意观察，GPU信息将会在终端打印输出，包括name，totalMemory，freeMemory等。如果不成功，显示的将会是CPU的信息。

# ARC Acknowledgement

All users of the ARC HPC facilities hosted at Leeds should use the following text to name and acknowledge their use of the facilities in published works. Please use the following statements (depending on the system used).

As part of the terms and conditions associated with undertaking research on the University HPC facilities, it is mandatory to include this acknowledgment text in all formally and informally published works. This includes but is not limited to papers, conference proceedings, presentations, posters, blog and other social media posts.

This work was undertaken on ARC4, part of the High Performance Computing facilities at the University of Leeds, UK.

or

This work was undertaken on ARC3, part of the High Performance Computing facilities at the University of Leeds, UK.


# Reference

https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-ii

https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-ii/solution/tan-xin-suan-fa-by-liweiwei1419-2/
