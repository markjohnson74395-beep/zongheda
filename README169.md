# 国产系统装完 WPS 打开一片方框：deb 与 rpm 怎么挑、缺的 3 个依赖和 1 套字体怎么补

上周三下午，一个做档案数字化的朋友抱着笔记本过来，说单位新发的机器跑统信 UOS，他自己下载装上 WPS，双击一份 `.docx` 合同，屏幕上是一排排空心方框——标题行糊成一团，数字能看，汉字全变成一个口一个口的形状，页码位置也歪了。他以为文件坏了，发到手机上打开却正常。

这就把问题定死了：文件没坏，是这台机器上**没有文档要求的那套字体**，加上安装时包选得不对、依赖补齐得不彻底，几个毛病叠在一起。国产系统上装 WPS Linux 版，麻烦基本集中在这三处：包格式、依赖、字体。这篇按我修过的几台机器（UOS、银河麒麟桌面版、openEuler 服务器带桌面、Ubuntu 22.04）的顺序写，界面路径和命令都可以对着 [WPS Linux 安装与字体配置说明](https://www.jackofalltraining.com) 一条条核。

## 那台 UOS 笔记本到底缺了什么

先复盘那台机器。`dpkg -l | grep wps` 能看到包装上了，版本是 11.1.0.11723，架构 `amd64`，看着没错。但 `ldd /opt/kingsoft/wps-office/office6/wps | grep "not found"` 一跑，冒出来一行 `libssl.so.1.1 => not found`。这台机器是 UOS 20，系统里只有 `libssl3`，而这一版 WPS 是链着 `libssl1.1` 编的。启动器能点开、界面能出来，所以很多人以为"装成功了"，但文档里的加密、签章、部分导入导出模块在调用到这个库时会静默失败。

再看字体：`fc-list :lang=zh | wc -l` 只有 11 条，全是文泉驿和 Noto 的几款。那份合同里指定的是宋体、仿宋、黑体三样，一个都没有。WPS 找不到就退回到系统默认中文字体，字形宽度不一样，表格列宽全被撑歪，标题的字重也掉了一档；遇到连替代字体都覆盖不到的生僻字，就直接吐方框。

所以"乱码"这个词得拆开。它在这个场景里其实是两件事：**缺字形**（方框、豆腐块）和**缺字体导致版式错位**（能看但不像原样）。修法完全不同。

## 四个问题定包型，别凭名字猜

WPS 官方对 Linux 只发两种包，`.deb` 和 `.rpm`。选错的典型后果不是装不上，而是装上一个半残的版本：能启动，某些功能报错，或者干脆依赖链一路红。

判断顺序是这样四条：

1. **系统家族**。Debian、Ubuntu、UOS、Deepin、银河麒麟桌面版走 `.deb`；RHEL、CentOS、Rocky、Fedora、openEuler、银河麒麟服务器版走 `.rpm`。
2. **CPU 架构**。这个最容易漏。飞腾、鲲鹏是 `arm64`，海光、兆芯是 `x86_64`，龙芯是 `loongarch64`。包文件名里带 `amd64`、`arm64` 的必须和 `uname -m` 对得上，拿 `amd64` 的包装到飞腾机器上，装得上也起不来。
3. **桌面环境**。WPS 依赖 GTK 一系，纯命令行最小化安装的系统要先补桌面运行库，否则启动时只报一句 `cannot open display`。
4. **系统自带的软件源能不能用**。`apt update` 或 `dnf makecache` 报错的机器，后面补依赖会一路卡，先把源修好再装 WPS。

| 系统 | 包格式 | 安装命令 | 常见依赖坑 | 中文字体现状 |
| --- | --- | --- | --- | --- |
| Ubuntu 22.04 / 24.04 | deb | `sudo apt install ./wps-office_*.deb` | 缺 `libssl1.1`、`libjpeg8` | 只有 Noto CJK，无宋体仿宋 |
| Debian 12 | deb | `sudo apt install ./wps-office_*.deb` | 缺 `libssl1.1`，源里已无此包 | 需自行补中文字体 |
| 统信 UOS / Deepin | deb | `sudo dpkg -i` 后 `apt install -f` | 缺 `libssl1.1`、部分 GTK 模块 | 自带字体偏少 |
| 银河麒麟桌面 V10 | deb | `sudo dpkg -i` 后 `apt install -f` | 与 UOS 类似，注意 arm64 包 | 自带字体偏少 |
| openEuler / CentOS / Rocky | rpm | `sudo dnf install ./wps-office-*.rpm` | 缺 `libXScrnSaver`、`libXss.so.1` | 需装 `wqy-*` 或自备 |
| 银河麒麟服务器版 | rpm | `sudo rpm -ivh` 后补依赖 | 无桌面环境时缺 GTK 一整套 | 基本无中文字体 |

## 装之前先把依赖翻一遍，能省半小时

包还没装的时候就能看出它要什么，这一步九成人跳过，然后卡在报错上。

deb 包：

```
dpkg -I wps-office_11.1.0.11723.XA_amd64.deb | grep -i depends
```

rpm 包：

```
rpm -qpR wps-office-11.1.0.11723-1.x86_64.rpm
```

输出里 `libssl`、`libXss`、`libjpeg` 开头的条目，就是后面可能红的点。装完再跑一次 `ldd` 看实际链接，比等报错再回头查快。

遇到的三条高频报错，大致长这样：

- `wps: error while loading shared libraries: libssl.so.1.1: cannot open shared object file: No such file or directory` —— 新版系统上跑老包，缺 OpenSSL 1.1。
- `错误：依赖检测失败： libXss.so.1()(64bit) 被 wps-office-11.1.0.11723-1.x86_64 需要` —— rpm 系缺 `libXScrnSaver`。
- `dpkg: 依赖关系问题使得 wps-office 的配置工作不能继续` —— deb 装到一半，依赖没落地，包处于未配置状态。

## 两套装法，命令别混着用

**deb 系（Ubuntu / Debian / UOS / 麒麟桌面）**

1. 先 `cd` 到包所在目录，用 `sudo apt install ./wps-office_11.1.0.11723.XA_amd64.deb`。注意前面那个 `./`，少了它 apt 会去源里找同名包而不是装本地这个文件。
2. 如果已经用 `dpkg -i` 装了一半报错，接着跑 `sudo apt install -f`，让包管理器自己把缺的依赖拉下来。
3. 还卡在 `libssl1.1` 上，就从 Ubuntu 20.04 的旧源里单独取这个库补上，再 `sudo dpkg -i` 一次。
4. 收尾跑 `sudo dpkg --configure -a`，把状态为"未配置"的包全部配置完，然后 `wps` 命令行启动一次看有没有报错输出。

**rpm 系（openEuler / CentOS / Rocky / 麒麟服务器）**

1. 用 `sudo dnf install ./wps-office-11.1.0.11723-1.x86_64.rpm`，dnf 会自动解依赖，比 `rpm -ivh` 省事。
2. 走 `rpm -ivh` 报依赖失败时，先 `sudo dnf install libXScrnSaver libX11 libXext` 补齐，再重跑。
3. 系统里没有图形界面的话，WPS 装了也用不了，先确认 `echo $DISPLAY` 有输出。

卸载各走各的：deb 系 `sudo apt remove wps-office`，rpm 系 `sudo rpm -e wps-office`，别互相借命令。

## 字体：方框、问号、整页乱码是三种病

字体是这类机器上最容易误判的一环。

**方框（豆腐块）**：文档指定的字体本机没有，连替代字体也覆盖不到那些字。生僻地名用字、公文里的特殊符号最先中招。

**版式错位**：字体被替换成了字形宽度不同的中文字体。字都能看，但行距、列宽、页码全跑，打印出来和原件对不上。判断办法是 `fc-match SimSun`，如果输出 `wqy-zenhei.ttc: "WenQuanYi Zen Hei"`，说明系统把宋体整体映射成了文泉驿黑体——黑体替代宋体，字重和宽度差得很明显。

**整页乱码**：这才轮到编码问题，通常是 GBK 存的老文件被按 UTF-8 打开。在 WPS 文字的打开对话框里换一次编码重试即可，跟字体无关。

修字体的步骤：

1. 建目录：`sudo mkdir -p /usr/share/fonts/truetype/wps-custom`。
2. 把宋体、仿宋、黑体、楷体、微软雅黑这几个字体文件拷进去。没有的话，用开源替代方案：`sudo apt install fonts-wqy-zenhei fonts-wqy-microhei` 或 `sudo dnf install wqy-zenhei-fonts`，再用思源宋体、思源黑体覆盖公文场景。
3. 刷新缓存：`sudo fc-cache -fv`。
4. 验证：`fc-list :lang=zh | grep -iE "simsun|simhei|noto serif cjk"`，能列出来就说明系统认到了。
5. 重启 WPS（不是关窗口，是退出进程），再打开那份出问题的文档。

文档已经打开、又不想重排版式的，可以在 WPS 文字里按 `Ctrl+D` 调出字体对话框，看「中文字体」那一栏当前指向谁，把它改成机器上确实装了的字体，勾选「应用于所有文字」再确定，先让内容能读，回头再统一。对外交付的文档建议直接换成思源系字体，字体授权上也不容易扯皮。

## 一张表对上症状和命令

| 现象 | 最可能的原因 | 先跑这条 |
| --- | --- | --- |
| 启动就报 `libssl.so.1.1` 缺失 | 系统只有 libssl3，包按 1.1 链接 | `ldd /opt/kingsoft/wps-office/office6/wps \| grep "not found"` |
| rpm 安装提示 `libXss.so.1()(64bit)` | 缺 X 屏保扩展库 | `sudo dnf install libXScrnSaver` |
| 界面中文全是方框 | 缺中文字形 | `fc-list :lang=zh \| wc -l` |
| 字能看但表格列宽全歪 | 宋体被映射成别的字体 | `fc-match SimSun` |
| 打开是老文件的整页乱码 | GBK/UTF-8 编码判断错 | 打开对话框里换编码重试 |
| 双击文档没反应、终端里报 `cannot open display` | 没有桌面会话或权限 | `echo $DISPLAY` |
| 装完启动器里没有图标 | 桌面缓存没刷新 | `sudo update-desktop-database` |

## 装完先做这三项验证，别急着关终端

第一，命令行起一次 `wps`，看有没有库报错刷屏，这一步能抓出所有静默失败的依赖。

第二，打开三份真实文件：一份带复杂表格的 `.docx`、一份 `.xlsx`、一份老 `.doc`。重点看表格列宽、页眉页码、公式结果，这三样是字体和渲染出问题时最先崩的地方。

第三，按 `Ctrl+P` 走一遍打印预览，再看「文件 → 输出为 PDF」导出一份，和原件逐页比对。预览正常、导出正常，这套环境才算能交付。

## 把顺序固定下来

国产系统上折腾 WPS，我现在的固定动作是四步：先看 `uname -m` 和系统家族定包型，再用 `dpkg -I` 或 `rpm -qpR` 把依赖翻一遍，装完立刻补中文字体并 `fc-cache -fv`，然后用真实文件跑三项验证。四步按顺序走，半小时能收工；顺序反了，就会像那台 UOS 一样，先花两小时怀疑文件坏了。

各发行版对应的包格式、依赖清单和字体目录，在 [WPS Office 下载与安装指南](https://www.jackofalltraining.com) 里是按系统分开列的，换机器时照着对一遍比记命令可靠。至于机器上到底装没装对，最快的判断不是看菜单有没有出现，而是 `fc-match SimSun` 那一行输出——它映射到谁，你的文档就会长成谁的样子。
