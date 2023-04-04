# VSCode插件生成编号、目录、文件目录树
- [1. 安装VS Code](#1-安装vs-code)
- [2. Markdown自动生成编号和目录](#2-markdown自动生成编号和目录)
- [3. 生成文件目录树](#3-生成文件目录树)
  - [3.1. 场景一：生成目录树链接](#31-场景一生成目录树链接)
  - [3.2. 场景二：生成目录树视图](#32-场景二生成目录树视图)
## 安装VS Code

> 官方地址：https://code.visualstudio.com/

- 下载安装完毕后在**扩展**中安装中文插件**Chinese (Simplified)**

## Markdown自动生成编号和目录

- 安装插件**Markdown All in One**
- 配置插件，将目录起始级别由**1**改为2

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202201233538.png" alt="image-20220220123317493" width="600" />

- 打开MD文档，**右键**--**命令面板**，搜索**markdown**，点击**添加/更新章节序号**

> 注：章节有变化后都需手动更新

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202201236286.png" alt="image-20220220123649250" width="400" />



- 创建目录后，点击**创建目录**，创建后保存时会自动更新，无需手动更新

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202201238790.png" alt="image-20220220123858737" width="600" />

## 生成文件目录树

### 场景一：生成目录树链接

- 安装插件**File Tree to Text Generator**
- 在文件列表右击，点击**Generate Filetree**，选择Markdown，4级目录。
- 使用全局替换，将`.\<根目录>\`替换为`\`，去掉根目录
- 再将路径中的`\`全局替换为`/`，即可在github中实现链接跳转。

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202201517606.png" alt="image-20220220151744559" width="500" />

### 场景二：生成目录树视图

- 安装插件**project-tree**
- 打开MD文档，**右键**--**命令面板**，搜索**project-tree**，点击即在README文档中生成目录树视图
- 可在`.gitignore`文件中配置不需要显示的文件

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202201518234.png" alt="image-20220220151855182" width="500" />
