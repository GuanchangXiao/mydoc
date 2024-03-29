## 使用 Alibaba 的 Homebrew 镜像源进行加速

平时我们执行 brew 命令安装软件的时候，跟以下 3 个仓库地址有关：

1. brew.git
2. homebrew-core.git
3. homebrew-bottles

通过以下操作将这 3 个仓库地址全部替换为 Alibaba 提供的地址

### 1. 替换 / 还原 brew.git 仓库地址

 替换成阿里巴巴的 brew.git 仓库地址: 

```shell
cd "$(brew --repo)" 
git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git
```

 还原为官方提供的 brew.git 仓库地址

```shell
cd "$(brew --repo)" 
git remote set-url origin https://github.com/Homebrew/brew.git
```

### 2. 替换 / 还原 homebrew-core.git 仓库地址

 替换成阿里巴巴的 homebrew-core.git 仓库地址: 

```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core" 
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git 
```

还原为官方提供的 homebrew-core.git 仓库地址 

```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core" 
git remote set-url origin https://github.com/Homebrew/homebrew-core.git
```



### 3. 替换 / 还原 homebrew-bottles 访问地址

替换 homebrew-bottles 访问 URL: 

```shell
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.bash_profile 
source ~/.bash_profile 

```

还原为官方提供的 homebrew-bottles 

```shell
vi ~/.bash_profile 
# 删除 HOMEBREW_BOTTLE_DOMAIN 这一行配置 
source ~/.bash_profile
```

