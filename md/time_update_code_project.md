# 云原生系列4 批量定时更新本地代码库



![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613571546472-1d437a7d-5bbe-4fae-96fa-91d118122cf0.png#align=left&display=inline&height=584&margin=%5Bobject%20Object%5D&name=image.png&originHeight=584&originWidth=1036&size=1475909&status=done&style=none&width=1036)




图中是一个自动化的机械流水线。


作为一名程序员，每天一定有非常多工作是每天必须重复的，如何消除重复性的工作？也让自己日常重复工作自动化呢？


# 背景和需求


![](https://cdn.nlark.com/yuque/__puml/ff66c63e1875ed7bbbfc7ea24cc2c080.svg#lake_card_v2=eyJjb2RlIjoiQHN0YXJ0dW1sXG5cbmFjdG9yICAgXCLlvIDlj5HkurrlkZhcIiBhcyAgQSAgXG5cbkEgLXVwLT4gKOaLieWPluS7o-eggSlcbkEgLXJpZ2h0LT4gKOiuvuiuoeWunueOsOaWueahiClcbkEgLXJpZ2h0LT4gKOWGmeS7o-eggSlcbkEgLXJpZ2h0LT4gKOa1i-ivleS7o-eggSlcbkEgLWRvd24tPiAo5o-Q5Lqk5Luj56CBKVxuQSAtbGVmdC0-ICjmjqjpgIHku6PnoIEpXG5cblxuXG5AZW5kdW1sIiwidHlwZSI6InB1bWwiLCJtYXJnaW4iOnRydWUsImlkIjoiWFVmb2IiLCJ1cmwiOiJodHRwczovL2Nkbi5ubGFyay5jb20veXVxdWUvX19wdW1sL2ZmNjZjNjNlMTg3NWVkN2JiYmZjN2VhMjRjYzJjMDgwLnN2ZyIsImhlaWdodCI6MzQyLCJjYXJkIjoiZGlhZ3JhbSJ9)







开发人员入职一家新公司，一般会使用git来进行代码的版本管理和协作，负责的代码库随着时间的推移会慢慢增加，最后可能会有1-20个代码工程，有些是新的工程，需要做新的功能特性开发，有的是老的工程做维护开发，而每个工程可能是多人协作的。手工更新多个代码工程的代码，有一些重复性的工作在里面，随着时间的推移，这个时间的消耗会更多，浪费了大量的编码和设计时间。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613573355026-75595437-1e65-4b40-aba2-5810897cae64.png#align=left&display=inline&height=380&margin=%5Bobject%20Object%5D&name=image.png&originHeight=760&originWidth=1648&size=90002&status=done&style=none&width=824)
假如每天花2分钟做拉取代码, 如果你维护20个工程，一年按照正常工作日上班，需要耗费173个小时时间。








# 目标提炼


这个批量更新代码的时间完全可以自动化，即通过定时任务执行脚本的方式，每日定时的批量更新你的代码工程，节约这个每年86个小时的时间，有更多的时间做设计和陪女朋友。


# 实现路径


要点：


1. 列举出你维护的git代码工程，并简单备注名称，类型；

2. 没有则clone代码到本地，有则拉取代码到本地，并做一定扩展；

3. 定时任务执行你的任务，在上班之前执行；







![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613573387068-35b7b64e-c76c-4871-b1d8-3ed6cbe03389.png#align=left&display=inline&height=236&margin=%5Bobject%20Object%5D&name=image.png&originHeight=472&originWidth=1852&size=94323&status=done&style=none&width=926)






## 列举维护的代码工程


文件名： codeProject.text


比如我录入了我放在github上的三个工程代码；

```
git@github.com:carterbrother/springbootpractice.git|springbootpractice|springboot练习代码|backend
git@github.com:carterbrother/COLA.git|cola|cola骨架代码|backend
git@github.com:carterbrother/cat.git|cat|cat服务监控代码|backend
```
## 循环处理代码并可不断扩展


一个shell循环处理即可，同时预留扩展；


比如如果是java后端工程，需要执行mvn clean install到本地；


如果是vue前端工程需要执行类似的操作；






总体的脚本如下：

```shell
#!/usr/bin/env bash
#set -e

function doExtend() {
  serviceType=$1
  appPath=$2
  if [ ${serviceType} == 'backend' ]; then
    cd ${appPath}
    git checkout dev
    git pull
    mvn clean install -Dmaven.test.skip=true
  fi
}

echo '拉取工作维护代码到本地开发机器'

export shPath="${PWD}"
echo "当前路径：${shPath}"

export codeBasePath=~/src/work
echo "你设置存放工作代码的目录是：${codeBasePath}"

if [ ! -d ${codeBasePath} ]; then
  echo "你设置存放工作代码的目录是：${codeBasePath} 它不存在，自动创建它！"
  mkdir -p ${codeBasePath}
fi

export codeProject="codeProject.txt"
echo '按照行来读取您维护的代码工程文件: ${codeProject}'



for line in $(cat "${shPath}/${codeProject}"); do
  echo "line conent: ${line}"
  arr=(${line//|/ })
  repoName=${arr[0]}
  serviceName=${arr[1]}
  serviceTitle=${arr[2]}
  serviceType=${arr[3]}
  echo "服务名称: ${serviceTitle},服务类型：${serviceType} 仓库git地址:${repoName} "

  appPath="${codeBasePath}/${serviceName}"

  if [ ! -d ${appPath} ]; then
    pwd
    echo "代码${serviceName}不存在，需要git clone到本地"
    cd ${codeBasePath}
    git clone "${repoName}"
  else
    cd ${appPath}
    pwd
    echo "代码${serviceName}存在，需要更新 git pull"
    git pull
  fi

  doExtend ${serviceType} ${appPath}
done

```


前提是你需要配置好你的git的ssh公钥信息到你的gitlab库，这里不会配置的话可自行利用搜索引擎。


## 定时任务执行脚本


我使用的是mac电脑，可以使用crontab工具来定时的执行上面的脚本。


命令格式：


```shell
crontab [-u user] file
crontab [-u user] [ -e | -l | -r ]
```


备份和恢复crontab
```shell
# 备份
crontab -l > $HOME/.mycron
# 恢复
crontab $HOME/.mycron
```


把文件放到对应的位置，crontab -e编辑，写入指令即可。
```shell
#每天6点定时拉取代码
* 6 *  *  * sh ~/tool/codetool/pullCode.sh
```


# 小结


一句话概括本篇：使用shell指定和定时任务crontab自动化的批量更新你的代码工程一年可节约86个小时时间。


![批量更新工作代码库.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613577685925-5b599f44-3b75-4466-b452-3b69dca692a7.png#align=left&display=inline&height=1050&margin=%5Bobject%20Object%5D&name=%E6%89%B9%E9%87%8F%E6%9B%B4%E6%96%B0%E5%B7%A5%E4%BD%9C%E4%BB%A3%E7%A0%81%E5%BA%93.png&originHeight=1050&originWidth=3664&size=276102&status=done&style=none&width=3664)




