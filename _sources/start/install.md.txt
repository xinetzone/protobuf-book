# 安装

如果之前安装过 Protobuf，一定要卸载干净：

```sh
sudo apt-get remove libprotobuf-dev
```

然后输入以下命令。删除 `protoc`：

```sh
which protoc
```

另外还要删除

```sh
sudo rm /usr/local/bin/protoc  # 执行文件
sudo rm -rf /usr/local/include/google # 头文件
sudo rm -rf /usr/local/lib/libproto* # 库文件
sudo rm -rf /usr/lib/protoc
```

在输入以下命令，发现 `protoc` 已经被删除干净了

```sh
which protoc
```

安装最新 `protobuf`：

```sh
git clone https://github.com/protocolbuffers/protobuf.git
cd protobuf
git submodule update --init --recursive

# 安装需要的软件
sudo apt-get install autoconf automake libtool curl make g++ unzip
```

## 编译

```sh
cd protobuf
./autogen.sh # 如果下载的对应语言的包则不需要
./configure
make -j4
sudo make install
sudo ldconfig # 刷新共享库缓存
protoc --version # 查看版本号
```

详细内容见：[安装](https://daobook.github.io/protobuf/src/README.html)。