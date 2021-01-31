---
date: 2020-06-14 10:47:44
tags:
- 随笔
---

## DigDog安装与简易运行演示

<!-- more -->

以纯净安装的Ubuntu 16.04系统进行演示

首先在源代码目录下，运行`install.sh`
>bash ./install.sh

![avatar](https://k1ng0fic3.github.io/images/digdog1.png)

出现是否ssh连接github的选项，同意之，保证检测报告可以上传至云端

![avatar](https://k1ng0fic3.github.io/images/digdog2.png)

待安装完成后，将系统配套的可执行文件，移入下图所示目录下运行
>DigDog\DigDog\App\Controller\

![avatar](https://k1ng0fic3.github.io/images/digdog3.png)

接下来就可以运行本产品了，选择一个内存文件并选择正确的profile值后，选择产品中自带的MLP模型进行扫描与检测

![avatar](https://k1ng0fic3.github.io/images/digdog4.png)

随后将自动打开报告页面，用户也可以选择手动打开 http://digdog-report.cn/archives/ 进行检测报告查看

更多操作，如开发者模式及自定义模型训练教程，请在系统配套的说明书中进行查阅

注：如果出现`no module named xxx`类报错，请检查pip是否正常工作，并根据源代码目录下的`requirements.txt`自行安装即可。