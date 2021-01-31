---
date: 2020-06-05 12:47:44
tags:
- 随笔
---

为了了解volatility的运作机制和插件情况（以实现开发新插件的计划），代码审计如下：
<!-- more -->
#### 0x00 volatility主线程

审计核心代码`vol.py`，源码：

```python
def main():

    # 打印版本信息
    sys.stderr.write("Volatility Foundation Volatility Framework {0}\n".format(constants.VERSION))
    sys.stderr.flush()

    # 初始化debug模块
    debug.setup()
    # Load up modules in case they set config options
    registry.PluginImporter()

    ## Register all register_options for the various classes
    registry.register_global_options(config, addrspace.BaseAddressSpace)
    registry.register_global_options(config, commands.Command)

    if config.INFO:
        print_info()
        sys.exit(0)

    ## Parse all the options now
    config.parse_options(False)
    # Reset the logging level now we know whether debug is set or not
    debug.setup(config.DEBUG)

    module = None
    # 获取插件的字典 插件名：插件体
    cmds = registry.get_plugin_classes(commands.Command, lower = True)
    for m in config.args:
        # 找到参数里的第一个插件名
        if m in cmds.keys():
            # module即为插件名 比如Linux_arp
            module = m
            break

    if not module: # 插件不存在
        config.parse_options()
        debug.error("You must specify something to do (try -h)")

    if module in cmds.keys():
        # 插件存在，则获取插件对象，并将config赋值module这个插件的_config变量
        command = cmds[module](config)
        
        # hook 上help函数
        config.set_help_hook(obj.Curry(command_help, command))
        config.parse_options()

        if not config.LOCATION:
            debug.error("Please specify a location (-l) or filename (-f)")

        # 开始执行插件
        command.execute()
```

总结的流程图如下：

![avatar](https://k1ng0fic3.github.io/images/vola1.png)

上面主要是主线程的启动流程。

1、vol从自身配置中获取插件字典，该字典的格式是[插件名:插件体]
2、主线程从命令行参数里获取第一个长得像是插件名的参数，就像我们输入的linux_arp一样的参数
3、参数：

    1、不存在像是插件名的字符串则报错并退出
    2、存在则以插件名、配置来初始化插件

4、在插件上hook help函数
5、开始执行插件

注意的：`get_plugin_classes`函数用于获取一切以`commands.Command`为父类的派生类字典。

#### 0x01 插件的类派生

![avatar](https://k1ng0fic3.github.io/images/vola2.png)

如上图，volatility中的插件以`commands`的`Command`类为父类，分别派生出`AbstractWindowsCommand`类、`AbstractMacCommand`类以及`AbstractLinuxCommand`类，这分别代表三个平台插件的基类。

这三类为不同平台的插件定义了一些共同的函数，以供具体的插件继承调用，也方便了我们自定义插件时复用这些函数。

#### 0x02 插件运行机理

![avatar](https://k1ng0fic3.github.io/images/vola3.png)

如上是插件的运行机理：

首先获取所有已注册的Profile的派生类集合。

```python
def execute(self):
    # 获取Profile类的子类
    profs = registry.get_plugin_classes(obj.Profile)
```

接着除了kdbgscan和imageinfo两个插件可以不提供profile而使用默认的profile--WinXPSP2x86以外，其他插件没有提供profile参数就不给运行的机会。

```python
    if plugin_name != "mac_get_profile":
        if self._config.PROFILE == None:
            if plugin_name in ["kdbgscan", "imageinfo"]:
                self._config.update("PROFILE", "WinXPSP2x86")
            else:
                debug.error("You must set a profile!")
         
            if self._config.PROFILE not in profs:
                debug.error("Invalid profile " + self._config.PROFILE + " selected")
            if not self.is_valid_profile(profs[self._config.PROFILE]()):
                debug.error("This command does not support the profile " + self._config.PROFILE)
```

接着执行插件的calculate函数，这个函数我们在自定义插件的时候一般会实现，作为vol和我们的自定义插件之间的调用接口，即插件的执行。

```python
    # # 首先执行calculate
    data = self.calculate()
```

随后基于输出格式确定渲染函数。

```python
    function_name = "render_{0}".format(self._config.OUTPUT)
```

若非sqlite输出且存在输出的目标文件，则进行写文件。否则将结果输出至标准输出。

```python
    if not self._config.OUTPUT == "sqlite" and self._config.OUTPUT_FILE:
        ...
    else:
        outfd = sys.stdout
```

其中默认以时间戳_插件名的命名方式来创建输出文件，若指定了输出文件，则使用用户自定义的。

```python
    out_file = '{0}_{1}.txt'.format(time.strftime('%Y%m%d%H%M%S'), plugin_name) if self._config.OUTPUT_FILE == '.' else self._config.OUTPUT_FILE
    if os.path.exists(out_file):
        debug.error("File " + out_file + " already exists.  Cowardly refusing to overwrite it...")
    print 'Outputting to: {0}'.format(out_file)
    outfd = open(out_file, 'wb')
```

接着执行插件对应的渲染函数`render_output`

```python
    func = getattr(self, function_name)
```

最后将结果输出到输出对象里。

```python
    func(outfd, data)
```

以上