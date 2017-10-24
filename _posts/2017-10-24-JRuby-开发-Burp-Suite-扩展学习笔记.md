---
layout: post
title: "JRuby 开发 Burp Suite 扩展学习笔记"
date: 2017-10-24
categories: Note
---
# JRuby 开发 Burp Suite 扩展

这两天学了下 Burp Suite 的扩展开发，因为比较偏爱 Ruby 所以就选择了 JRuby 作为开发语言。

首先我要从吐槽开始。。。。我先在在网上搜了一圈，然后发觉一个很搞笑的问题，所有的中文资料只有一两份样子剩下的全部都是不眨眼不脑的复制粘贴。。。。。。。。。。90%以上的文章复制粘贴还都是同一篇文章。它们都在开头搭建开发环境时千篇一律的选择使用 RVM 来装JRuby，编译完之后还有再去找到所在路径然后用命令行加相应的参数来启动 Burp Suite。

由于我只是简单的写个练练手没必要扯上版本管理，于是我默默地去 JRuby 官网下载了 JRuby 的 jar 包，然后打开 Burp Suite 在 Extender 的 option 里手动载入。

![burp-jruby](/assets/images/burp-jruby.png)

写啥插件好呢？完全没想法。。。于是就网上搜了搜，看到一篇是通过 Jython 来 shodan API 来做一些简易查询的例子感觉这个挺简单的就照着样子用 JRuby 抄了个 -。-

从官网的示例代码来看必须要实现 `BurpExtender` 类，根据你所使用的场景功能还要 include 相对应 module:

```ruby
require 'java'
java_import 'burp.IBurpExtender'
java_import 'burp.IHttpListener'
java_import 'burp.IProxyListener'
java_import 'burp.IScannerListener'
java_import 'burp.IExtensionStateListener'

class BurpExtender
  include IBurpExtender, IHttpListener, IProxyListener, IScannerListener, IExtensionStateListener
    
  #
  # implement IBurpExtender
  #
  
  def	registerExtenderCallbacks(callbacks)
    # keep a reference to our callbacks object
    @callbacks = callbacks
    
    # set our extension name
    callbacks.setExtensionName "Event listeners"
    
    # obtain our output stream
    @stdout = java.io.PrintWriter.new callbacks.getStdout, true
    
    # register ourselves as an HTTP listener
    callbacks.registerHttpListener self
    
    # register ourselves as a Proxy listener
    callbacks.registerProxyListener self
    
    # register ourselves as a Scanner listener
    callbacks.registerScannerListener self
    
    # register ourselves as an extension state listener
    callbacks.registerExtensionStateListener self
  end
  
  #
  # implement IHttpListener
  #

  def processHttpMessage(toolFlag, messageIsRequest, messageInfo)
    @stdout.println(
            (messageIsRequest ? "HTTP request to " : "HTTP response from ") +
            messageInfo.getHttpService.toString +
            " [" + @callbacks.getToolName(toolFlag) + "]")
  end

  #
  # implement IProxyListener
  #

  def processProxyMessage(messageIsRequest, message)
    @stdout.println(
            (messageIsRequest ? "Proxy request to " : "Proxy response from ") +
            message.getMessageInfo.getHttpService.toString)
  end

  #
  # implement IScannerListener
  #

  def newScanIssue(issue)
    @stdout.println "New scan issue: #{issue.getIssueName}"
  end

  #
  # implement IExtensionStateListener
  #

  def extensionUnloaded()
    @stdout.println "Extension was unloaded"
  end
end
```

registerExtenderCallbacks 相当于 main 函数，扩展的名字啥的都在这里做定义，API 上的说明是:

> This method is invoked when the extension is loaded. It registers an instance of the IBurpExtenderCallbacks interface, providing methods that may be invoked by the extension to perform various actions.

利用 registerExtenderCallbacks 返回的对象的 `registerContextMenuFactory` 方法来实现创建上下文菜单以及期望的入口:

> This method is used to register a factory for custom context menu items. When the user invokes a context menu anywhere within Burp, the factory will be passed details of the invocation event, and asked to provide any custom context menu items that should be shown.

所以 `registerExtenderCallbacks` 的方法内容如下:

```ruby
def registerExtenderCallbacks(callbacks)
  @callbacks = callbacks
  helpers = callbacks.getHelpers()
  callbacks.setExtensionName("Shodan Scan")
  callbacks.registerContextMenuFactory(self)
end
```

接下来需要将扩展的名称添加到 Burp 的菜单中:

```ruby
def createMenuItems(invocation)
  menu_list = []
  menu = JMenuItem.new "Scan with Shodan", nil
  menu.addActionListener do |e|
    startThreaded(invocation)
  end
  menu_list << menu
  menu_list
end
```

最后实现相关功能部分就行了，[完整源码](https://github.com/alex-marmot/learning-burp-suite-extender)


## 资料来源

- [https://github.com/hackzsd/BurpExtenderPractise](https://github.com/hackzsd/BurpExtenderPractise)
- [http://www.itnose.net/detail/6701646.html](http://www.itnose.net/detail/6701646.html)