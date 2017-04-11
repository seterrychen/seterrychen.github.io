title: Vundle安裝
author: Terry Chen
tags:
  - vim
  - vim-plugin
date: 2014-10-04
---

[Vundle](https://github.com/VundleVim/Vundle.vim)是用來管理Vim中的plugin，其優點和安裝方式可參考[Vundle：Vim Plugin 自動下載、安裝、更新與管理工具](http://www.gtwang.org/2014/04/vundle-vim-bundle-plugin-manager.html)。

其中值得注意的事在於`.vimrc`的設置，安裝Vundle必要的plugin有`gmarik/vundle`，其餘的就看個人所需要的plugin再安裝。


# 安裝Vundle

1. clone
```
$ git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

2. 設定`~/.vimrc`
```
set nocompatible              " be iMproved, required
filetype off                  " required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

Plugin 'gmarik/Vundle.vim'

call vundle#end()            " required
filetype plugin indent on    " required
```

3. install
```
$ vim +PluginInstall +qall
```


# 基本使用

## 安裝Plugin

如果要安裝github上的scrooloose/nerdtree，編輯`~/.vimrc`加在**call vundle#begin**和**call vundle#end()**之間加入

```
...(略)
call vundle#begin()

Plugin 'gmarik/Vundle.vim'
Plugin 'scrooloose/nerdtree'   " <--- add one line

call vundle#end()
...(略)
```

儲存並執行`:PluginInstall`，退出Vim，下次再啟Vim就可以使用nerdtree。支援多種不同的plugin來源：

* 加入`Plugin 'L9'` 會從http://vim-scripts.org下載並安裝
* 加入`Plugin 'git://git.wincent.com/command-t.git'` 從git.wincent.com下載並安裝
* 加入`Plugin 'file:///home/gmarik/path/to/plugin'` 安裝本機路徑的plugin

更多指定安裝來源的方法，可參考在[Vundle Github](https://github.com/gmarik/Vundle.vim)中Quick Start第三點裡的示範。

## 移除Plugin

跟安裝相同，將要移除的Plugin從`.vimrc`裡刪掉之後，再執行`:PluginClean`即可。

## 更新Plugin

執行`:PluginUpdate`
