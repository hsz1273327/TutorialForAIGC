# ollama作为推理框架

正常llama.cpp都不会被直接安装使用,毕竟它主要是推理框架,推理的模型我们还得自己维护.ollama就是这个可以管理模型的工具.
作为llama.cpp的上层管理工具,ollama自然是可以顺利执行的.它在设计上充分参考了docker--一样的c/s结构,一样的用systemd管理服务,一样定义了一种打包方式用于专门打包模型,一样的有一个中心化的ollma hub用于上传和分化打包好的模型,对习惯docker的用户来说就相当好上手.

## 安装与配置

安装只需要挂上代理常规安装即可

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

在linux下默认安装的版本是cpu和是Nvidia显卡的版本.如果你有Nvidia显卡,它会为你配置环境;如果你是amd显卡或核显,在rocm装好的情况下这个脚本会安装[ollama-linux-amd64-rocm.tgz](https://github.com/ollama/ollama/releases/tag/v0.5.4)这样的rocm专用版本;

在windows下ollama需要运行在WSL2中,你先得配好WSL2然后它才能正确安装.默认安装的版本是cpu和是Nvidia显卡的版本,如果你有Nvidia显卡它会为你配置环境;如果你是amd显卡或核显,这个脚本就只能让你运行在cpu中了,要想能在amd的显卡或核显上运行它你需要参考[likelovewant/ollama-for-amd](https://github.com/likelovewant/ollama-for-amd)这个项目,安装hip环境,然后下载它提供的ollama替换原本的脚本中安装的ollama可执行文件即可

ollama本质上是一个go程序,正常情况下会被安装到`/usr/local`,同时会配置`systemd`到`/etc/systemd/system/ollama.service`.由于我们是780m,要让igpu成为首选就需要做如下设置:

1. 先停掉`ollama`

    ```bash
    sudo systemctl stop ollama.service
    ```

2. 进入systemd的设置页设置ollama.service的启动环境(一般文件都还没有,需要创建)

    ```bash
    sudo su
    cd /etc/systemd/system/
    mkdir ollama.service.d
    cd ollama.service.d
    nano override.conf
    ```

    填入如下内容

    ```bash
    [Service]
    Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0" # 780m
    Environment="OLLAMA_MAX_LOADED_MODELS=1" # 仅加载一个模型
    Environment="OLLAMA_NUM_PARALLEL=1" # 仅允许一个并发
    ```

    当然了如果有其他要设置的也在这里设置,设置项可以用`ollama serve --help`查看

3. 重新加载ollama.service的设置,并重启

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart ollama.service
    ```

当然你如果是直接下载的ollama可执行文件,手动配置的环境,你也可以不走systemd,要用的时候`ollama serve`启动ollama服务即可.

ollama启动后服务端默认会在`127.0.0.1:11434`监听.我们可以通过环境变量`OLLAMA_HOST`来修改这个监听位置

## 本地模型管理

ollama的模型默认根据操作系统不同放在不同的位置

+ linux: `/usr/share/ollama/.ollama/models`
+ mac: `~/.ollama/models`
+ windows: `:\Users\%username%\.ollama\models`

我们可以修改环境变量`OLLAMA_MODELS`来改变它的查找路径.ollama维护模型的方式类似docker维护镜像,是将模型分层()切割为多个文件(`blobs`),拿其hash值命名,然后通过描述文件(`manifests`)记录不同模型不同版本的层次类型等元信息.

我们可以通过`ollama pull <模型名:标签>`来从ollama的模型仓库拉取指定模型到本地;`ollama list`查看本地有哪些已经下好的模型;用`ollama show <模型名:标签>`来确认模型信息;通过`ollama rm 模型名:标签>`来删除指定模型.

## 聊天

用ollama聊天默认就是在terminal中进行,使用

```bash
ollama run [flags] <模型名:标签>
```

其中比较重要的标签包括

+ `--verbose` 可以在回答结果后面带上推理的过程信息,其中会有速度指标,一般打开用来检查推理速度
+ `--format string` 返回的结构,


当我们要运行一个大模型时,ollama会根据`vram`的大小来判断是否要让cpu参与推理,参与的程度就是

```bash
(模型大小-vram):vram=cpu:gpu
```

要使用某个模型也很简单



## 配置私有模型

## 模型分享