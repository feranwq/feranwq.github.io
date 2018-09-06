title: bash_profile
date: 2018-09-06 23:30:18
tags: 
mp3: 
cover: http://wx1.sinaimg.cn/large/77b6881dgy1fmc4f1yj15j20xc0rsdgl.jpg
---
做个记录，不断补充吧


```
# histroy

HISTCONTROL=ignoredups  #忽略重复命令
HISTCONTROL=erasedups #清除重复命令， 多个终端写history的时候还是有可能重复
HISTFILESIZE=1000000000 #命令历史文件大小
HISTSIZE=10000  # 保存历史命令条数
PROMPT_COMMAND="history -a"
export HISTSIZE PROMPT_COMMAND

# git
find_git_branch () {

local dir=. head

until [ "$dir" -ef / ]; do

if [ -f "$dir/.git/HEAD" ]; then

head=$(< "$dir/.git/HEAD")

if [[ $head = ref:\ refs/heads/* ]]; then

git_branch=" (${head#*/*/})"

elif [[ $head != '' ]]; then

git_branch=" → (detached)"

else

git_branch=" → (unknow)"

fi

return

fi

dir="../$dir"

done

git_branch=''

}

alias gitl='git log --color --graph --decorate -M --pretty=oneline --abbrev-commit -M --all '
alias gits='git status'

PROMPT_COMMAND="find_git_branch; $PROMPT_COMMAND"
export LS_OPTIONS='--color=auto' # 如果没有指定，则自动选择颜色
export CLICOLOR='Yes' #是否输出颜色
export LSCOLORS='ExfxcxdxbxegedabagGxGx' #指定颜色
black=$'\[\e[1;30m\]'

red=$'\[\e[1;31m\]'

green=$'\[\e[1;32m\]'

yellow=$'\[\e[1;33m\]'

blue=$'\[\e[1;34m\]'

magenta=$'\[\e[1;35m\]'

cyan=$'\[\e[1;36m\]'

white=$'\[\e[1;37m\]'

normal=$'\[\e[m\]'

PS1="$white[$white@$green\h$white:$cyan\W$yellow\$git_branch$white]\$ $normal"

```