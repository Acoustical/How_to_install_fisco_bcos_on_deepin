# 统信UOS / Deepin上搭建 FISCO-BCOS 2.6.0 ，控制台及区块链浏览器

此方法已经在ARM的 统信UOS 实体机上实验成功

### 1. 环境搭建

##### （1）安装基础依赖

```shell
apt-get install git openssl cmake flex patch bison gcc g++ automake
```

##### （2）cmake 版本过低问题

<p style="color:red">注意：此处 cmake 版本不要选择过大或过小，过小的版本（低于3.10）无法编译 fisco-bcos，过大的版本（高于3.14）依赖于C++11，而 FISCO-BCOS 源码编译要求 gcc 和 g++ 版本不能高于 8，为了不重复安装多版本，故 cmake 版本选择 3.13.4</p>

i. 查看当前 cmake 版本

```shell
cmake -version
```

> cmake version 3.9.5

如果此处的 cmake 版本小于 3.10，则需要重新安装大于 3.10 版本的 cmake

ii. 卸载之前版本的 cmake

```shell
apt-get remove cmake
```

iii. 下载 cmake 源码并解压

```shell
wget https://github.com/Kitware/CMake/releases/download/v3.13.4/cmake-3.13.4.tar.gz
tar -xzvf cmake-3.13.4.tar.gz -C /opt/
```

iv. 编译安装

```shell
cd /opt/cmake-3.13.4
./bootstarp
make -j4
make install
cd ~
```

v. 创建软链接

```shell
ln -s /opt/cmake-3.13.4/bin/* /usr/bin/
```

vi. 查看是否安装成功

```shell
cmake -version
```

##### （2）解决 openssl 出现 libssl.so.1.0.0 找不到的问题

i. 上述基础依赖安装完成后，输入以下命令检查 openssl 安装是否成功

```shell
openssl version
```

如果出现以下回显，则 openssl 安装成功：

> OpenSSL 1.1.0j  20 Nov 2018

如果出现如下 libssl.so.1.0.0 缺失的问题，则需要通过编译重新安装 openssl

> libssl.so.1.0.0: no version information available (required by openssl)

ii. 下载 openssl 源码并解压

```shell
wget https://www.openssl.org/source/openssl-1.1.1a.tar.gz
tar -xzvf openssl-1.1.1a.tar.gz && cd openssl-1.1.1a
```

iii. 新建编译配置文件

```shell
vim openssl.ld
```

输入：

```json
OPENSSL_1.0.0 { 
global: 
*; 
};
```

iv. 设置编译选项

```shell
./config --prefix=/usr/local --openssldir=/usr/local/openssl shared -Wl,--version-script=/home/openssl.ld -Wl,-Bsymbolic-functions zlib-dynamic enable-camellia -DOPENSSL_NO_HEARTBEATS
```

v. 编译

```shell
make
make install
```

切记将编译好的文件放到系统的可执行目录中（或配置环境变量），一般是<code>/usr/bin</code>,<code>/usr/local/bin</code>下

##### （3）安装 java

i. 下载解压 java 源码

在 [ Oracle官网 ](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html) 上下载 jdk-1.8安装包，并解压

```shell
tar -xzvf jdk-8u261-linux-arm64-vfp-hflt.tar.gz  -C /usr/local/
```

ii. 设置环境变量

```shell
vim /etc/profile
```

在文档中输入：

```shell
export JAVA_HOME=/usr/local/jdk1.8.0_261
export PATH=$JAVA_HOME/bin:$PATH
```

保存退出，然后使文档生效：

```shell
source /etc/profile
```

查看 java 是否安装成功

```shell
java -version
```

> java version "1.8.0_261"
> Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
> Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)

### 2. FISCO-BCOS 源码编译安装

##### （1）源码下载

选择某个目录作为源码保存位置，我这里保存在了`~/fisco-bcos`下。

i. 首先下载编译时的大文件（本步骤可选，但是如果在国内还是建议执行这一步）（这一步的功能主要是提前下载编译中需要用到的几个大文件，以防编译下载时下载文件过慢，因为这几个源都在国外，所以建议执行这一步）

```shell
cd ~
mkdir fisco-bcos && cd fisco-bcos
git clone https://github.com/FISCO-BCOS/LargeFiles.git
```

如果 github 过慢，也可以使用码云 gitee 的源

```shell
git clone https://gitee.com/FISCO-BCOS/LargeFiles.git
```

ii. 下载 FISCO-BCOS 源代码

```shell
git clone https://github.com/FISCO-BCOS/FISCO-BCOS.git
cd FISCO-BCOS
```

这一份文件同样有码云的地址，不赘述

iii. 查看本机 gcc 版本

```shell
gcc -v
```

> gcc version 6.3.0 20170516 (Debian 6.3.0-18+deb9u1) 

如果这里的 gcc 版本号大于等于 9，需要把分支切换至 dev 分支（dev分支不建议用于生产环境，如果要用于请降级 gcc 版本）

```shell
# 如果 gcc 版本号不大于 9
git checkout master

# 如果 gcc 版本号大于 9
git checkout dev
```

##### （2）编译安装

i. 执行预编译

```shell
mkdir -p build && cd build
cmake .. -DARCH_NATIVE=on
```

ii. 复制依赖包到相应目录

```shell
cp ~/fisco-bcos/LargeFiles/libs/* ~/fisco-bcos/FISCO-BCOS/deps/src
```

iii. 执行编译

```shell
make -j4
```

本步不建议使用 `make` 进行编译，不然会出现 evmc 没有编译出来的问题，建议使用四核选项编译

iv. GroupSigLib 报错

> [ 24%] Performing configure step for 'GroupSigLib'
> -- GroupSigLib configure command succeeded.  See also /root/fisco-bcos/FISCO-BCOS/deps/src/GroupSigLib-stamp/GroupSigLib-configure-*.log
> [ 25%] Performing build step for 'GroupSigLib'
> CMake Error at /root/fisco-bcos/FISCO-BCOS/deps/src/GroupSigLib-stamp/GroupSigLib-build-RelWithDebInfo.cmake:49 (message):
>   Command failed: 2
>
>    'make'
>
>   See also
>
> /root/fisco-bcos/FISCO-BCOS/deps/src/GroupSigLib-stamp/GroupSigLib-build-*.log
>
>
> make[2]: *** [CMakeFiles/GroupSigLib.dir/build.make:115：../deps/src/GroupSigLib-stamp/GroupSigLib-build] 错误 1

如果编译时出现以上问题（用非 CentOS/Ubuntu/Suse 发行版，肯定会出现这个问题），那么则是编译文件生成的 config.guess 文件无法获知系统架构或内核信息，我们需要借用以下 automake 中的 config.guess 文件

等待编译停止后，按如下步骤执行：

```shell
ls /usr/share/ | grep automake-
```

> automake-1.15

找到 automake 文件夹后，运行一下 automake 的 config.guess 文件，查看结果

```shell
/usr/share/automake-1.15/config.guess
```

统信 UOS 实体机上显示：

> aarch64-unknown-linux-gnu

Deepin 15 虚拟机上显示：

> x86_64-pc-linux-gnu

若该文件可以如此成功显示内核版本，则执行以下命令替换编译生成的 config.guess 文件

```shell
cp /usr/share/automake-1.15/config.guess ~/fisco-bcos/FISCO-BCOS/deps/src/GroupSigLib/deps/src/pbc_sig/config.guess
```

若不能正常显示内核版本，则需要根据本机信息手动修改 config.guess 文件源码，然后再将编译文件中的 config.guess 替换掉

v. 待编译完成后检查是否安装成功

```shell
cd bin
./fisco-bcos -v
```

> FISCO-BCOS Version : 2.6.0
> Build Time         : 20201017 13:10:51
> Build Type         : Linux/g++/RelWithDebInfo
> Git Branch         : master
> Git Commit Hash    : a2c2cd3f504a101fbc5e97833ea0f4443b68098e

### 3. 搭建1群组4节点联盟链

i. 依照当前 FISCO-BCOS 版本下载自动搭链脚本

```shell
cd ~/fisco-bcos
curl -LO https://github.com/FISCO-BCOS/FISCO-BCOS/releases/download/`curl -s https://api.github.com/repos/FISCO-BCOS/FISCO-BCOS/releases | grep "\"v2\.[0-9]\.[0-9]\"" | sort -u | tail -n 1 | cut -d \" -f 4`/build_chain.sh && chmod u+x build_chain.sh
```

ii. 运行搭链脚本

```shell
./build_chain.sh -l 127.0.0.1:4 -p 30300,20200,8545 -e FISCO-BCOS/build/bin/fisco-bcos
```

> Checking fisco-bcos binary...
> Binary check passed.
> &#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;
> Generating CA key...
> &#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;
> Generating keys and certificates ...
> Processing IP=127.0.0.1 Total=4 Agency=agency Groups=1
> &#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;
> Generating configuration files ...
> Processing IP=127.0.0.1 Total=4 Agency=agency Groups=1
> &#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;
> [INFO] FISCO-BCOS Path : bin/fisco-bcos
> [INFO] Start Port      : 30300 20200 8545
> [INFO] Server IP       : 127.0.0.1:4
> [INFO] Output Dir      : /root/fisco-bcos/nodes
> [INFO] CA Path         : /root/fisco-bcos/nodes/cert/
> &#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;
> [INFO] Execute the download_console.sh script in directory named by IP to get FISCO-BCOS console.
> e.g.  bash /root/fisco-bcos/nodes/127.0.0.1/download_console.sh -f
> &#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;&#61;
> [INFO] All completed. Files in /root/fisco-bcos/nodes

iii. 启动节点

```shell
cd nodes/127.0.0.1/
./start_all.sh
```

iv. 查看节点状态

```shell
ps -aux | grep fisco
```

> root     17241  1.1  0.7 729792 31168 pts/1    Sl   17:32   0:00 /root/nodes/127.0.0.1/node2/../fisco-bcos -c config.ini
> root     17242  1.0  0.7 729792 31360 pts/1    Sl   17:32   0:00 /root/nodes/127.0.0.1/node0/../fisco-bcos -c config.ini
> root     17243  1.0  0.7 729792 31168 pts/1    Sl   17:32   0:00 /root/nodes/127.0.0.1/node3/../fisco-bcos -c config.ini
> root     17244  1.1  0.7 729408 31296 pts/1    Sl   17:32   0:00 /root/nodes/127.0.0.1/node1/../fisco-bcos -c config.ini

v. 查看共识状态

```shell
tail -f node*/log/* | grep +++
```

> info|2020-10-18 17:34:19.452001|[g:1][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,nodeIdx=2,hash=40f8c3c5...
> info|2020-10-18 17:34:18.449782|[g:1][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,nodeIdx=1,hash=85ab01ec...
> info|2020-10-18 17:34:17.446976|[g:1][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,nodeIdx=0,hash=d7d2cfa2...
> info|2020-10-18 17:34:20.454172|[g:1][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,nodeIdx=3,hash=87efbeb5...
> info|2020-10-18 17:34:21.456586|[g:1][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,nodeIdx=0,hash=4a9c4f2d...
> info|2020-10-18 17:34:22.459794|[g:1][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,nodeIdx=1,hash=d1dd4738...

### 4. 搭建 FISCO-BCOS 控制台

i. 下载控制台源码

```shell
cd ~/fisco-bcos
git clone https://github.com/FISCO-BCOS/console.git
```

ii. 编译安装

```shell
cd console
git checkout release-1.1.1
./gradlew build -x test
```

> &gt; Task :compileJava
> 注: /root/console/src/main/java/console/web3j/Web3jImpl.java使用或覆盖了已过时的 API。
> 注: 有关详细信息, 请使用 -Xlint:deprecation 重新编译。
> 注: /root/console/src/main/java/console/common/ContractClassFactory.java使用了未经检查或不安全的操作。
> 注: 有关详细信息, 请使用 -Xlint:unchecked 重新编译。
>
> Deprecated Gradle features were used in this build, making it incompatible with Gradle 6.0.
> Use '--warning-mode all' to show the individual deprecation warnings.
> See https://docs.gradle.org/5.6.2/userguide/command_line_interface.html#sec:command_line_warnings
>
> BUILD SUCCESSFUL in 1h12m35s
> 3 actionable tasks: 3 executed

注：这一步比较慢的话是网络原因，可以先将需要下载的文件拷贝下来，再执行该步

iii. 修改配置文件

```shell
cd dist/conf
cp applicationContext-sample.xml applicationContext.xml

// 如果节点端口没有使用20200，需要编辑文件修改端口
vim applicationContext.xml
```

iv. 拷贝证书

```shell
cp ~/fisco-bcos/nodes/127.0.0.1/sdk/* .
```

v. 启动控制台

```shell
cd ~/fisco-bcos/console/dist/
./shart.sh
```

> ```
> =============================================================================================
> Welcome to FISCO BCOS console(1.1.0)!
> Type 'help' or 'h' for help. Type 'quit' or 'q' to quit console.
>  ________ ______  ______   ______   ______       _______   ______   ______   ______
> |        |      \/      \ /      \ /      \     |       \ /      \ /      \ /      \
> | $$$$$$$$\$$$$$|  $$$$$$|  $$$$$$|  $$$$$$\    | $$$$$$$|  $$$$$$|  $$$$$$|  $$$$$$\
> | $$__     | $$ | $$___\$| $$   \$| $$  | $$    | $$__/ $| $$   \$| $$  | $| $$___\$$
> | $$  \    | $$  \$$    \| $$     | $$  | $$    | $$    $| $$     | $$  | $$\$$    \
> | $$$$$    | $$  _\$$$$$$| $$   __| $$  | $$    | $$$$$$$| $$   __| $$  | $$_\$$$$$$\
> | $$      _| $$_|  \__| $| $$__/  | $$__/ $$    | $$__/ $| $$__/  | $$__/ $|  \__| $$
> | $$     |   $$ \\$$    $$\$$    $$\$$    $$    | $$    $$\$$    $$\$$    $$\$$    $$
>  \$$      \$$$$$$ \$$$$$$  \$$$$$$  \$$$$$$      \$$$$$$$  \$$$$$$  \$$$$$$  \$$$$$$
> 
> =============================================================================================
> [group:1]>
> ```

vi. 测试控制台

```shell
[group:1]> getNodeVersion
{
    "Build Time":"20201018 09:43:15",
    "Build Type":"Linux/g++/RelWithDebInfo",
    "Chain Id":"1",
    "FISCO-BCOS Version":"2.6.0",
    "Git Branch":"master",
    "Git Commit Hash":"a2c2cd3f504a101fbc5e97833ea0f4443b68098e",
    "Supported Version":"2.6.0"
}
```

vii. 部署示例合约

```shell
[group:1]> deploy HelloWorld
contract address: 0xd22aa109bc0708ad016391fa5188e18d35b16434

[group:1]> call HelloWorld 0xd22aa109bc0708ad016391fa5188e18d35b16434 set "nice"
transaction hash: 0x72f4f8c980fd0d63d57bdbcc89d6b82dda79e301f25a65f0f49726105184b596

[group:1]> call HelloWorld 0xd22aa109bc0708ad016391fa5188e18d35b16434 get
nice
```

### 5. 搭建 FISCO-BCOS 区块链浏览器

##### （1）安装配置 MySQL 数据库

i. 安装 MySQL

```shell
apt-get install mariadb*
```

ii. 启动 MySQL

```shell
systemctl start mariadb.service
```

iii. 配置 MySQL

```shell
mysql_sercure_installation
```

输入该命令后，之后的步骤依次为

* 为root用户设置密码
* 删除匿名账号（建议删除）
* 取消root用户远程登录（建议取消）
* 删除test库和对test库的访问权限（建议删除）
* 刷新授权表使修改生效（建议加载）

也就是说除了设置root用户密码那块需要输入几遍密码

iv. 登陆MySQL

```shell
mysql -u root
```

v. 配置账户

```mysql
CREATE 'test'@'localhost' IDENTIFIED by '123456';
# 其中 test 为账户名，123456 为密码，可以根据需要修改
```

vi. 使用新用户登陆并创建数据库

```shell
mysql -utest -p123456 -h 127.0.0.1 -P 3306
```

```mysql
SHOW DATABASES;
USE test;
CREATE DATABASE db_browser;
```

##### （2）安装 gradle

i. 在 [gradle官网](https://gradle.org/releases/)下载gradle（版本至少为5）

ii. 解压缩并安装

```shell
mkdir /opt/gradle
unzip -d /opt/gradle gradle-6.7-bin.zip
```

iii. 配置环境变量

```shell
# 可以把这两句写进 /etc/profile
export GRADLE_HOME=/opt/gradle/gradle-6.7
export PATH=$GRADLE_HOME/bin:$PATH
```

##### （3）下载并编译源码（服务端）

i. 拉取源码

```shell
git clone https://github.com/FISCO-BCOS/fisco-bcos-browser.git
```

ii. 编译源码

```shell
cd fisco-bcos-browser/server/fisco-bcos-browser
gradle build
```

iii. 修改配置文件

```shell
cp dist/conf_template dist/conf -r
cd dist/conf
vim application.yml
```

修改第8、9行为自己的数据库账户名和密码

```shell
username: test
password: 123456
```

iv. 启动服务

```shell
cd ~/fisco-bcos/fisco-bcos-browser/server/fisco-bcos-browser/dist
```

```shell
# 启动服务
sh start.sh
# 停止服务
sh stop.sh
# 服务状态
sh status.sh
```

v. 日志检查

如果搭建过程中发生错误，可用命令查看日志

```shell
tail -f ~/fisco-bcos/fisco-bcos-browser/server/fisco-bcos-browser/distlog/fisco-bcos-browser.log
```

##### （4）编译安装源码（界面端）

i. 安装nginx并备份nginx配置文件

```shell
apt-get install nginx
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```

ii. 将前端界面备份，移出

```shell
cd ~/fisco-bcos/fisco-bcos-browser
mkdir /opt/fisco-bcos-web
cp -r ./web/fisco-bcos-browser-front/dist /opt/fisco-bcos-web/
# 不要移动到用户根目录，之后会提示没有操作权限（即使权限调成了777）
```

iii. 拷贝bcos的nginx文件并编辑

```shell
cp ./web/fisco-bcos-browser-front/docs/nginx.conf /etc/nginx/
vim /etc/nginx/nginx.conf
```

修改配置文件：

```nginx
server {
    listen       5100 default_server;   # nginx 监听端口，稍后访问页面需要的端口
    server_name  127.0.0.1;           	# 前端访问地址，可以配置域名（如果需要非本机访问不要配置本地回环地址）
    location / {
        root    /opt/fisco-bcos-web/dist;   # 前端文件路径（就是刚刚拷贝出去的 dist 文件）
        index  index.html index.htm;
        try_files $uri $uri/ /index.html =404;
    }

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

        location /api {
        proxy_pass    http://127.0.0.1:5101/;    # 后端服务(fisco-bcos-browser server)地址及端口
        proxy_set_header		Host				$host;
        proxy_set_header		X-Real-IP			$remote_addr;
        proxy_set_header		X-Forwarded-For		$proxy_add_x_forwarded_for;
	}
}
```

iv. 启动nginx

```shell
systemctl start nginx
```

v. 访问前端

使用浏览器访问 127.0.0.1:5000