# mac
#### 비번 설정 초기화
~~~bash
$ pwpolicy -clearaccountpolicies
~~~ 

# intellij
#### plugins
* Rainbow Brackets
* Key Promoter X

#### 설정
* Preferences - Editor - Live Templates
* Preferences - Editor - Inspections - Serialization class without serialVersionUID

# Visual Studio Code
#### Font size
* control + shift + p 
  * Preferences: Open Settings (JSON)
  ~~~json
  {
      "window.zoomLevel": 1.2
  }
  ~~~

#### IntelliJ IDEA Keybindings
* https://marketplace.visualstudio.com/items?itemName=k--kato.intellij-idea-keybindings

# SourceTree
- git ui

# git

#### git 자동 완성

~~~sh
$ brew install hub
$ mkdir -p ~/.zsh/completions
$ cp ~/Download/hub.zsh_completion ~/.zsh/completions/_hub

$ vim ~/.zshrc
fpath=(~/.zsh/completions $fpath) 
autoload -U compinit && compinit

$ source ~/.zshrc
zsh compinit: insecure directories, run compaudit for list.
Ignore insecure directories and continue [y] or abort compinit [n]? 
compinit: initialization aborted

$ ls /usr/local/share
$ sudo chmod -R 755 zsh
~~~

* https://github.com/github/hub/tree/master/etc#zsh

# sdkman
* https://sdkman.io/install

~~~bash
$ curl -s "https://get.sdkman.io" | bash
$ source "$HOME/.sdkman/bin/sdkman-init.sh"
$ sdk install java 8-open /Library/Java/JavaVirtualMachines/adoptopenjdk-8.jdk/Contents/Home 
~~~

