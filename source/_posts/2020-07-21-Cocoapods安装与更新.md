---
title:      "一、Cocoapods安装与更新" 
date:       2020-07-21
tags:
    - iOS
    - Cocoapods
categories:
    - iOS
    - Cocoapods
---

1.首先查看 `gem source` 源
```
gem sources -l
```

2.删除源
```
gem sources --remove https://rubygems.org/
```

3.添加新的源
```
gem sources --add https://gems.ruby-china.com/
```

4.升级ruby

查看ruby版本：`ruby -v`

安装homebrew
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

可以使用`rvm`和`brew`来安装最新的ruby

 - rvm:

    `rvm install 2.5.3`

    `rvm use 2.5.3 --default`

 - bew:

    `brew install ruby`

5.安装cocoapods

`sudo gem install -n /usr/local/bin cocoapods`

`pod setup`由于网络原因，会执行的很慢

可以使用如下命令

`git clone https://mirrors.tuna.tsinghua.edu.cn/git/CocoaPods/Specs.git  ~/.cocoapods/repos/trunk`

