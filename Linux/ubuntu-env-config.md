#windows/wsl #code-env/lib  #linux/ubuntu #c-cpp 
## 2022.1.20

## 换源

```bash
$ sudo mv /etc/apt/sources.list /etc/apt/sources.list.bak
$ sudo touch /etc/apt/sources.list
$ sudo vi /etc/apt/sources.list
# https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/
```
## 更新并安装库

```bash
$ sudo apt-get update && sudo apt-get --assume-yes upgrade
$ sudo apt-get install build-essential -y
$ sudo apt-get install man -y
$ sudo apt-get install gdb -y
$ sudo apt-get install git -y
$ sudo apt-get install libreadline-dev -y
$ sudo apt-get install libsdl2-dev -y
$ sudo apt-get install llvm -y
$ sudo apt-get install make -y
```

## 配置git

```bash
$ git config --global user.name "xac"      
$ git config --global user.email "jiangfengxi.c@gmail.com"
$ rm -rf ~/.ssh
$ ssh-keygen -t rsa # 按三个回车确保没有密码
$ cat ~/.ssh/id_rsa.pub
```

add ssh on github setting page

## 在windows上访问wsl

在文件管理器中输入`\\wsl$`即可访问

## Lazygit

**Install latest git**
```bash
$ sudo add-apt-repository ppa:git-core/ppa
$ sudo apt update
$ sudo apt install git
```

```bash
$ LAZYGIT_VERSION=$(curl -s "https://api.github.com/repos/jesseduffield/lazygit/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
$ curl -Lo lazygit.tar.gz "https://github.com/jesseduffield/lazygit/releases/latest/download/lazygit_${LAZYGIT_VERSION}_Linux_x86_64.tar.gz"
$ tar xf lazygit.tar.gz lazygit
$ sudo install lazygit /usr/local/bin
```

## Latest gcc

```bash
$ sudo add-apt-repository ppa:ubuntu-toolchain-r/test
$ sudo apt update
$ sudo apt install g++-13
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 60 --slave /usr/bin/g++ g++ /usr/bin/g++-13
```
