# 下载及安装

[下载地址](https://golang.google.cn/dl/)

按照版本所需下载即可


安装一般推荐直接使用安装包安装.

> 环境变量配置

**MAC**

1. 打开.bash_profile文件即`vi ~/.bash_profile`
2. 写入如下内容
   ```
    export GOROOT=/usr/local/go
    export PATH=$PATH:$GOROOT/bin
    export GOPATH=$HOME/yourpath
   ```
3. 重写加载`source ~/.bash_profile`

注意,其中的GOROOT是go的bin文件所在的根目录
GOPATH是go项目的工作目录,这个也类似eclipse中的workspace的概念,可以有多个.后期使用goLand开发也可以忽略这个点,在开发工具中可以随意创建指定即可


# hello world

1. 创建文件以`.go`结尾
2. 写入代码
   ```
    package main
    import "fmt"
    func main(){
        fmt.Println("hello world")
    }
   ```
3. 执行,使用`go run test.go`执行即可

