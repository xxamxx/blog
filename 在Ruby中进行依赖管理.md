# 在Ruby中进行依赖管理

日常编码我们经常会需要重用别人的功能代码，那么我们该怎么做呢？又会带来什么问题呢？通过点"历史"我们来了解下这件事发展的过程。

## 了解"历史"

早期*Ruby*提供了`require`方法用于加载外部代码，让各位猿人们可以引入其他人的代码模块，方便了项目灵活组织代码。`require`有两个全局变量， `$LOADED_FEATURES`记录已加载文件名避免重复加载，`$LOAD_PATH`用于提供未指明路径的文件名可被加载的目录，例如`require "rails"`不需要给出绝对路径，会自动从`$LOAD_PATH`中检索并加载。



后来有人觉得可以有个固定目录，所有从网上下载的库文件都放到该目录中，将目录地址加入到`$LOAD_PATH`中就能很方便的使用其他人的库了，这就是后面`setup.rb`做的事情。从网上下载库后，在目录中执行`ruby setup.rb all`就将该库加入到了目录。不过慢慢大家发现如果作者更新了功能，或是fix bug，需要去找到作者的主要查看新版本，然后重新下载、解压、`ruby setup.rb`，看下你现在项目依赖了多少库就知道这件事情有多繁琐了。那么我们如何优雅的跟进呢？于是大家开始考虑需要对库进行版本控制了。



接着就整出了我们的*RubyGems*。

> RubyGems允许你通过简单的方式可下载、安装、使用Ruby包。这些包被称为`gem`，其包含了Ruby应用或是库。
>
> Gems可扩展或修改的Ruby应用。通常，将可重用的功能分发共享给其他开发者的应用或库。有些Gems则会提供命令行以帮助自动化任务和提高效率。

通过它提供简单的`gem`命令行工具，如同简介所说，我们可以非常简单的方式可下载、安装、使用*Ruby包*（通常称作做*gem*，为更好区分下文简称"包"）。

```bash
$ gem install rails
$ gem list rails
$ gem uninstall rails
```

而且，*RubyGems*还支持多版本控制！它会自动的帮你把最新版本加入到`$LOAD_PATH`。你也可以在项目中调用`gem`方法，并带有一个版本号，该版本的路径就会被加入`$LOAD_PATH`，后面当你调用 require 的时候，那个指定的版本就是你得到的版本。

```ruby
gem "rack", "1.0"
require "rack"
```



随着我们项目和依赖越来越多，特别是需要多台电脑协作一个项目的时候，会发现依赖之间的版本冲突或不兼容问题愈发需要耗精力去处理，例如一个包说依赖于某版本的某库，那么这个库的路径就被放在 `$LOAD_PATH`中了，但这时来了另外一个包，它同样依赖于某库，然而依赖的是另外一个版本，这时这个版本就不能被放进来了，因为另外一个版本已经存在了，这个时候就会抛出`activation error`。然而版本冲突的解析不应该发生在运行时，而应该发生在安装时。

```bash
$ rails s
Gem::LoadError: Can't activate rack (~> 1.0.0., runtime) for ["actionpack-2.3.5"], already activated rack-1.1.0 for ["thin-1.2.7"]
```

而*Bundler*就是解决问题的方案，执行`bundle install`命令它会读取项目根目录下用户声明依赖的`Gemfile`文件,通过"Dependency Graph Resolution"过程然后得出一个可行的依赖版本的解，并写入`Gemfile.lock`中。而`bundle exec`会确保在这个命令下运行的*Ruby*代码 require 到的库都是`Gemfile.lock`中指定的那些特定的版本，而且会事先把`$LOAD_PATH`中多余的东西清理掉让你不会误引用到其他的版本。





上面的事情让我们了解到我们应该利用*Bundler*进行依赖版本管理，那接下来才是我们的主角，下文我们会介绍如何使用`Gemfile`和`Gemfile.lock`管理依赖及版本。

## Gemfile

`Gemfile`是一套**描述*Ruby*程序的*gem依赖*的文件**，通过`Gemfile`可以管理项目所依赖的包并指明引入依赖的条件或环境，实际上它是一份可被*Bundler*执行的*Ruby*代码。

```ruby
source "https://rubygems.org"

gem "rails", "4.1.0.rc2"
gem "nokogiri", "~> 1.6.1"

group :production do
	gem "rack-cache"
end
```

文件提供了非常灵活的*DSL*方法用来**描述依赖**，详细可见[Gemfile文档][man-gemfile]，以下是常用方法：

- `source` 告诉*Bundler*你指定的源

- `ruby` 指定ruby版本

- `gem` 声明依赖的包名称

- `group` 提供一个分组概念，通过*Bundler*的`--with`,`--without`参数可灵活的安装指定包

像这个例子，第一行指定从`https://rubygems.org`这个源下载*gem*。下面说明我们依赖`rails`、`nokogiri`、`rack-cache`。

其中`rails`、`nokogiri`有特别指定了版本，而`nokogiri`只有在`bundle install --with production`时才会被安装。



```ruby
source "https://rubygems.org"

gem "rails", "4.1.0.rc2"
gem "nokogiri", "~> 1.6.1"
gem 'activesupport', '5.2.2', :require => ['active_support']
gem 'harmonious_dictionary', git: 'https://github.com/yeezon/harmonious_dictionary.git'

group :production do
	gem "rack-cache"
end
```

`gem`方法支持强大的参数，它也是我们用的最多。

- `git` 直接引用依赖的*git*仓库地址，*Bundler*会直接从仓库下载并安装包
- `require` 支持两种参数；`true`/`false`代表是否
- `path` 引用本地依赖地址
- `plaforms` 描述该依赖只在某个平台才引入,默认就是`ruby`

- 其他可见[Gemfile文档][man-gemfile]

上面示例我们看到`gem`可以指定版本，而实际场景这也是我们最需要关注的。通常我们会看到以下几种情况。

- `rails 4.1.0.rc2` 即只依赖`4.1.0.rc2`这版本的`rails`，我们也可以指定多个版本如`rails 4.1.0,4.5.0`
- `rack >= 0.4` 说明只安装版本大于等于`0.4`的`rack`
- `nokogiri ~> 1.6.1` 其中`~> 1.6.1`意思是`>= 1.6.1`且`< 1.7.0`



## Gemfile.lock

讲完`Gemfile`就要提提`Gemfile.lock`，**`Gemfile.lock`把你的应用变成一个你的代码跟第三方代码最后一次你确定能正常工作的包。**

`bundle install`会根据你以及*gem*的依赖计算出可行的版本并记录到`Gemfile.lock`中，下次`bundle install`时就会跳过解决依赖版本这个环节，直接根据`Gemfile.lock`下载并安装*gem*。

```ruby
source "https://rubygems.org"

gem "rails", "4.1.0.rc2"
gem "rack-cache", require: "rack/cache"
gem "nokogiri", "~> 1.6.1"
```

当我们需要更新*gem*版本时可以直接修改`Gemfile`，但如果该版本的*gem*所依赖的包在本地并没有安装过，那启动就会失败，所以通常会推荐通过`bundle update [my_gem]`将它及它的依赖更新到最新的可用的版本。

避免更新操作导致依赖和依赖之间不兼容这个问题，*Bundler*更新时**采取保守策略**。当更新一个*gem*时，如果有其*gem*有与它相同的依赖，*Bundler*就不会更新那个相同的依赖。举个例子更新`rails`也会更新`rack 1.5.3`，但由于`rack-cache`依赖`rack 1.5.2`，*Bundler*不会更新`rack`。这样保证了更新`rails`不会不小心搞坏`rack-cache`。由于`rails 4.1.0`保留了`rack 1.5.2`的兼容，*Bundler*就不会管它，`rack-cache`就会继续工作。

而`bundle update`则会从头开始解决依赖并重写`Gemfile.lock`，不过这个操作风险很大，你要准备好`git reset --hard`和测试用例，一般不推荐。从头解决依赖会有意想不到的结果，特别是一部分你依赖的第三方库在你上一次更新的时候发布了新的版本。





------

[man-gemfile]: https://bundler.io/man/gemfile.5.html#GLOBAL-SOURCES	"Gemfile 文档"
