---
layout: post
title:  "打造Mac个人开发环境"
date:   2015-01-10 20:33:25
categories: mac 
tags: mac
---

自从Windows系统转到Mac系统以来，越来越觉得基于Unix系统的Mac是对程序员更为友好的使用伙伴。自己根据使用的场景对Mac开发环境也做了一些定制，将主要过程记录下来，将来也可以再次使用。

#   1.iterm2的安装和配置

苹果自带的term很难用，iterm2实际上是程序员通往外界的一把钥匙。安装iterm很简单，直接去官网下载安装就可以了。后面需要对iterm进行配色，下载solarized到~/Code/tools，然后通过Finder来到~/Code/tools/solarized/iterm2-colors-solarized/目录，双击下面的两个Solarized Dark和Solarized Light就可以直接加载到iterm2中。

```bash
mkdir -p ~/Code/tools/
git clone https://github.com/altercation/solarized
```

然后，打开iterm，进入Preference->Colors，有一个 Load Presets的按钮，选择Solaried Dark这个主题就可以了。

最后通过 Perfernces->Window的Window Appearance可以调整Transparency和Blur。通过设置一定的Transparency，可以在term中看到打开的网页，方面在iterm中输入命令。

#   2.dircolor的配置

在term中使用最多的shell及时ls了，如果ls显示不带任何颜色是非常难看的。Mac自带的ls显示颜色效果一般，为了使用solarized主题显示ls的颜色效果，需要用到Mac下安装GNU 软件的利器 --- Homebrew.

首先我们可以来到Homebrew的官网来安装Homebrew。
[Homebrew](http://brew.sh "homebrew")

使用系统自带的ruby来安装homebrew  

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

这里不推荐使用git clone的方式自己搞，因为有一大队文件夹得权限问题，而安装脚步帮我们自动搞定了。

安装完homebrew之后就可以安装coreutils以及grep等GNU 软件了。

```
brew install coreutils
brew install grep
brew install wget
brew install git
```
看到所有GNU 软件会被装到/usr/local/bin下面，更早的homebrew版本默认安装到/usr/local/homebrew/bin下面。我会把/usr/local目录默认为homebrew安装GNU 软件的目录。

我们需要把/usr/local/bin在PATH环境变量里面提前，这样coreutils里面的ls等命令会覆盖原系统自带的。具体做法是在~.bash_profile里面加入：

```
export PATH="/usr/local/opt/coreutils/libexec/gnubin:/usr/local/bin:$PATH"
```

最后一步就是给ls进行配色了，需要下载最新的dircolors-solarized。

```
cd ~/Code/tools/
git clone https://github.com/seebi/dircolors-solarized
```
我们需要在~.bash_profile里面加载某个dircolors方案，我选择的是solarized dark。并且打开ls的颜色开关。

```
eval `gdircolors -b $HOME/Code/tools/dircolors-solarized/dircolors.ansi-dark`
alias ls='ls -F --show-control-chars --color=auto'
alias ll='ls -al -F --show-control-chars --color=auto'
```

#   3.Bash prompt
在bashrc加入

```
 PS1=$'\[\e]2;[bash]   \h::\]$PWD\[\a\]\[\e]1;\]$(basename "$(dirname "$PWD")")/\W\[\a\]bash@\w\$ 🚀  '
```

在bash_profile加入

```
source ~/.bashrc
```

也可以进行更为复杂的pipeline的配置，在term里显示详细的git信息如branch等。有兴趣的可以Google。

#   4.vim安装和配置
还是使用homebrew安装最新的vim  

```
brew install vim
```

在/usr/local/bin中建立vi的链接，否则vi还会使用系统自带的  

```
cd /usr/local/bin
ln -sf ./Cellar/vim/7.4.1401/bin/vim vi
```

我使用了[spf13-vim](http://vim.spf13.com/ "spf13-vim")来进行vim的插件管理和相关配置，事实上,spf13-vim使用的是[Vundle](https://github.com/gmarik/vundle.git" "Vundle")这个插件管理器来管理所有的vim plugins。  

```
cd $HOME
git clone git@github.com:froghui/froghui-dotfiles.git .froghui-dotfiles
cd .froghui-docfiles
bash bootstrap.sh
```

#   5.ruby和jekyll安装
首先使用homebrew安装rbenv  

```
brew install rbenv
rbenv install --list
rbenv install 2.0.0-p481
rbenv rehash
rbenv install 2.3.0
rbenv rehash
rbenv list
rbenv global 2.3.0
```

之后在bash_profile里面初始化rbenv，以便通过rbenv安装的shims可以生效。  

```
# Ruby
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
```

最后就是安装bundler以及各种需要的gems。   

```
gem sources --remove https://rubygems.org/
gem sources --remove http://rubygems.org/
gem sources -a https://ruby.taobao.org/
gem sources -l
gem update --system
gem install bundler
gem install jekyll
gem install jekyll-paginate
gem install rdiscount
```

下面就可以使用jekyll写markdown以及个人blog了。我比较喜欢使用Sublime写markdown，当然,需要装一些插件。

#   6.Sublime及插件安装
首先去官网安装Sublime Text 3。
之后安装package control，[Package Control安装](https://packagecontrol.io/installation#st3"package control install")
有两种方式，一种是代码直接安装，另外一种是手动安装。

安装OmniMarkupPreviewer和MarkDownEditing两款插件。具体安装就不说了，通过以下快捷方式唤出Install Package即可。  

```
Ctrl + Shifit + P
```


#   7.Java和Maven安装
可以安装多个版本的Java，在bash_profile中加入这段可以选择需要的Java版本。  

```export JAVA_HOME=`/usr/libexec/java_home -v 1.7`
```

推荐使用IDEA (IntelliJ)做为Java开发的IDE。
