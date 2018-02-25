title: Shell script debug技巧
author: Terry Chen
tags:
  - shell
  - debugging
date: 2018-02-25 12:53
---

個人在工作時候大量地使用Shell scripts，透過它將服務所需要的環境自動化設定並帶起該服務，很多時候需要針對script執行失敗的原因進行修正。為了達到這個目的，單單使用`echo`指令印出執行時期的結果來找出失敗的原因是不夠的。以下記錄個人覺得非常實用的debug技巧跟大家分享。

## Shell script前言

> sh (or the Shell Command Language) is a programming language described by the [POSIX standard](http://pubs.opengroup.org/onlinepubs/009695399/utilities/xcu_chap02.html). It has many implementations (ksh88, dash, ...). bash can also be considered an implementation of sh (see below).

以上那段話是出自[shell - Difference between sh and bash - Stack Overflow](https://stackoverflow.com/questions/5725296/difference-between-sh-and-bash)，用來說明`sh`是一個[標準](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_06_01)，而實作這個標準的有ash(busybox)、bash、dash、zsh等等。(而也有很多人使用的[fish shell](https://github.com/fish-shell/fish-shell)目前並不相容與POSIX standard，故以下技巧不一定適用於fish shell) 

## 技巧
### xtrace功能

在POSIX standard裡Shell Command Language中的[Special Built-In Utilities](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_14)定義shell須要支援built-in utilities在shell中執行。其中[set utility](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#set)是用來在shell執行時期設定shell的options和parameters。以下為部份shell options：

```
NAME
    set - set or unset options and positional parameters

DESCRIPTION

    ..........(skip)

    -o  option
        This option is supported if the system supports the User Portability
        Utilities option. It shall set various options, many of which shall be
        equivalent to the single option letters. The following values of option
        shall be supported:

        ..........(skip) 

        verbose
            Equivalent to -v.
        xtrace
            Equivalent to -x.

    -x
        The shell shall write to standard error a trace for each command after
        it expands the command and before it executes it. It is unspecified
        whether the command that turns tracing off is traced.

    ..........(skip)

```

xtrace的option功能為shell展開指令之後將結果輸出到standard error，然後再執行該指令。以下用簡單的script範例以及使用`bash`來展示其效果。

#### 範例

* 直接帶入`-x`option

    ```
    root@4837ac34d286:/# cat hello_world.sh
    words="Hello world"
    echo $words
    
    root@4837ac34d286:/# bash -x hello_world.sh
    + words='Hello world'
    + echo Hello world
    Hello world
    ```

* 使用set utility

    ```
    root@4837ac34d286:/# cat hello_world.sh
    set -x
    words="Hello world"
    echo $words
    
    root@4837ac34d286:/# bash hello_world.sh
    + words='Hello world'
    + echo Hello world
    Hello world
    ```

可以看到xtrace的效果為將每行指令展開後加上預設前輟字串'**+ **'，將結果輸出到standard error。xtrace也能展開parameters：

```
root@4837ac34d286:/# cat hello.sh
set -x
name=$1
echo "Hello ${name}, welcome to the world!"

root@4837ac34d286:/# bash hello.sh Terry
+ name=Terry
+ echo 'Hello Terry, welcome to the world!'
Hello Terry, welcome to the world!

root@4837ac34d286:/# bash hello.sh Terry Chen
+ name=Terry
+ echo 'Hello Terry, welcome to the world!'
Hello Terry, welcome to the world!
```

當該範例的parameters帶入Terry Chen時，shell解釋該指令會展開成parameter 1($1)=Terry，parameter 2($2)=Chen，其原因是IFS預設值為`<space>,<tab>和<newline>`，需要加上雙引號`"Terry Chen"`，shell才會解釋成為一個parameter，此為常見的錯誤之一。

```
root@4837ac34d286:/# bash hello.sh "Terry Chen"
+ name='Terry Chen'
+ echo 'Hello Terry Chen, welcome to the world!'
Hello Terry Chen, welcome to the world!
```

另外xtrace也包含對function的展開，對於debug複雜的script時非常有幫助。首先我們加上`export`更改前輟字串(PS4 variable)變為`"\$BASH_SOURCE \$LINENO> "`，讓它展開指令時顯示其所在的檔案和行數(BASH_SOURCE只作用於bash shell, 而LINENO則是看該shell有沒有支援，像[dash在0.5.9.1才有支援此variable](https://github.com/infertux/bashcov/issues/24#issuecomment-272637205))。例如：

```
root@4837ac34d286:/# cat main.sh
set -x

export PS4="\$BASH_SOURCE \$LINENO> "

. ./functions
hello $1

root@4837ac34d286:/# cat functions
hello() {
    name=$1
    echo "Hello ${name}, welcome to the world!"
}

root@4837ac34d286:/# bash main.sh
+ export 'PS4=$BASH_SOURCE $LINENO> '
+ PS4='$BASH_SOURCE $LINENO> '
main.sh 5> . ./functions
main.sh 6> hello
./functions 2> name=
./functions 3> echo 'Hello , welcome to the world!'
Hello , welcome to the world!
```

### exec utility

有時候問題發生在某些特定的情況之下，例如你所寫的script是在作業系統啟動時被呼叫，或是你的script提供功能讓其他的程式來呼叫等等情境所產生的。對於這類無法經由重新執行script本身來重現的問題，我會開啟xtrace option並使用exec utility重新定向standard error到檔案，再透過該特定情境再製問題後，就有log檔可以分析。

簡單地說明一下exec的作用，在[POSIX standard](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#exec)裡提到它是用來執行指令並同時可以用來操作file descriptors(簡稱為fd)和將file descriptors重新導向到其他地方。**當使用exec時未指定指令時**，其對fd操作時將會變為對目前的shell fd進行操作。接下來就來看使用exec所帶來的效果。

#### 範例

```
root@d8bad16f2fc3:/# cat hello_world.sh
set -x
exec >> /tmp/log 2>&1
words="Hello world"
echo $words

root@d8bad16f2fc3:/# bash hello_world.sh
+ exec

root@d8bad16f2fc3:/# cat /tmp/log
+ words='Hello world'
+ echo Hello world
Hello world
```

注意在此例中`exec`的前後，所輸出的訊息分別導到螢幕(+ exec)和/tmp/log檔案。

## 其他

在其他這個部份想要說明的是良好的習慣可減少你花在debug的時間，像是總是在script開頭使用shebang語法指定所使用的script直譯器，使用工具來檢查script語法是否正確等等。

* Shebang

  [Shebang](https://en.wikipedia.org/wiki/Shebang_%28Unix%29)指的是置於script檔案的開頭由`#!`所組成的字串，用來指定script直譯器來執行該檔案。例如：`#!/bin/bash`指定用bash來執行該script。由於實作shell script的直譯器眾多，各自的語法有些微的不同，為了避免執行時期可能產生不同的結果，個人覺得我們應該總是使用shebang來防止意料之外的錯誤發生。

* ShellCheck

  Shell script相較於其他的programming language，有一個很不一樣的特性，當它執行時發生錯誤時並不會中斷執行，而是會一直繼續往下執行下一個指令，直到整個script執行結束。在這樣的特性之下，有可能造成資料或系統的損害。為了減少錯誤的發生，我們可以使用靜態檢查工具來防範語法錯誤所帶來的危害。shell script本身有提供語法檢查的功能，使用option `-n`此時意味著`noexec`，透過不執行指令的方式來檢查。

  這邊提供一個功能更強的語法檢查工具：[ShellCheck](https://github.com/koalaman/shellcheck)，它可以安裝在很多作業系統上，也有線上檢查的服務[ShellCheck – shell script analysis tool](https://www.shellcheck.net/)。透過ShellCheck可以發現可能造成問題的code，而每一個rule都有清楚的附上可能有問題的原因以及怎樣才是比較好的寫法，例如：[rule SC2115 Use "${var:?}" to ensure this never expands to /* .](https://github.com/koalaman/shellcheck/wiki/SC2115)

  > Problematic code: `rm -rf "$STEAMROOT/"*`
  > Correct code: `rm -rf "${STEAMROOT:?}/"*`
  >
  > Rationale:
  > If `STEAMROOT` is empty, this will end up deleting everything in the system's root directory.
  >
  > Using `:?` will cause the command to fail if the variable is null or unset. Similarly, you can use `:-` to set a default value if applicable.


* Strict Mode

  由於上述提到的shell script特性，有人則是提出使用strict mode，當指令回傳失敗時即中斷該script的方式來避免錯誤，出自[Use the Unofficial Bash Strict Mode (Unless You Looove Debugging)](http://redsymbol.net/articles/unofficial-bash-strict-mode/)：

  ```
  #!/bin/bash
  set -euo pipefail
  IFS=$'\n\t'
  ```

  之所以該文章標題為`Bash Strct Mode`，是因為`set -o pipefail`並不適用於POSIX standard shell，`pipefail`是用於來指定當使用pipeline來串接指令時，只要有一個指令回傳錯誤，整條的指令將視為錯誤而不是依據pipeline的最後一個指令來決定，就像該文章中舉到的例子：

  > ```
    % grep some-string /non/existent/file | sort
    grep: /non/existent/file: No such file or directory
    % echo $?
    0
    ```
  > ```
    % set -o pipefail
    % grep some-string /non/existent/file | sort
    grep: /non/existent/file: No such file or directory
    % echo $?
    2
    ```

  然而使用Strict Mode時更影響Shell script的撰寫方式，例如有時是透過指令回傳的exit code結果來決定接下來的要如何執行指令：

  ```
  if some-command some-argument; then
      do something
  else
      excute error handler
      exit 1
  fi
  ```

  在Strict Mode情況之下，就無法達到該目的，而該篇作者也提出針對在Strict Mode之下的不同情況因應辦法(Issues & Solutions段落)，針對上述的問題的解決方案為：

  ```
  set +e
  if some-command some-argument; then
      do something
  else
      excute error handler
      exit 1
  fi
  set -e
  ```

  個人在工作時沒有使用Strict Mode，主要的原因為引入Strict Mode將會大大的改變團隊撰寫程式的方式，也有人認為更好的方法為[良好的error檢查](https://news.ycombinator.com/item?id=11315144)。在Hacker News則有一篇關於該篇文章的討論[Use the Unofficial Bash Strict Mode (Unless You Love ... - Hacker News](https://news.ycombinator.com/item?id=11313928)，供各位參考。個人雖然沒有使用Strict Mode，但很推薦大家看該篇文章，相信有助於對shell script的瞭解。


## 相關文章
* [Shell Scripts Matter - Dev.To](https://dev.to/thiht/shell-scripts-matter)
