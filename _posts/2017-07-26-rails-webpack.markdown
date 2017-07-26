---
layout: post
title: "Rails with Webpack"
date: 2017-07-26
categories: rails
---

#### [github](https://github.com/rails/webpacker)

JavaScript发展太快，浏览器端虽然对ES6语法的支持还不是很完善，但是现在前端的各种框架和代码都在用ES6写了，有的甚至在用ES7了，因此就需要把ES6或者ES7的语法转换成传统的ES5，然后再交给浏览器来运行。

这个专题我们就来说说怎样在Rails中使用Webpack来管理JS和CSS。
```ruby
虽然Rails在 5.1 之后已经默认添加了Webpack的支持，但是需要我们指定开启，而且需要一定的定制，我们这个专题就细细来给大家说下怎样安装，配置和使用Webpack，以及怎样组织你的JS代码。
```

## Webpack介绍
Webpack是现在非常流行的基于nodejs的静态资源(assets)管理框架，比如JS和CSS的压缩合并。

Webpack中可以配置各种插件来完成资源的中间编译处理，比如ES6/7转成ES5([babel](https://babeljs.io/))，把sass转换成css，图片的无损压缩([image-webpack-loader](https://github.com/tcoopman/image-webpack-loader))等等。

当然我们使用Webpack来管理JS的另一个好处是不需要再手工下载第三方的JS库了 (比如jQuery，React等等)，直接在nodejs中安装就行了。

<!-- ![Image](../post_images/webpack.png) -->

## Rails的Asset Pipeline
在Rails中有自己的静态资源管理组件Asset Pipeline，负责静态资源的压缩合并等，和Webpack作用是类似的，但是默认情况下Asset Pipeline是不支持把ES6转化成ES5的，虽然有对应的gem可以做到，但是还不是很完善，而且这些gem也都是基于nodejs的babel来实现的，所以如果你想要在项目中使用ES6，那么还是建议采用Webpack。
```ruby
如果在Rails的项目中采用Webpack的话，从Webpack的角度来说，我们就不再需要Rails的Asset Pipeline了，所有静态资源的管理都可以交给Webpack来处理；但是Asset Pipeline毕竟是Rails中的重要组件，相信有一定Rails使用经验的同学也难以割舍，而且现在很多已有项目都是采用Asset Pipeline方式来组织的，因此我们可以保留Asset Pipeline，根据实际情况，结合两个一块使用。
```

## 安装
Rails在 5.1 之后已经默认添加了Webpack的支持，但是需要指定开启，而且需要一定的定制。

Webpack在Rails中的安装有以下要求：
```ruby
Ruby 2.2+
Rails 4.2+
Node.js 6.4.0+
Yarn 0.20.1+
```

Yarn是另一个nodejs的package管理工具，和npm的作用类似，你可以使用npm来安装yarn。
如果你是在老的Rails项目中使用Webpack并且Rails版本低于上面要求的话就需要先升级下Rails和Ruby。

Ruby和Rails的安装这里就不再重复了，nodejs的安装请直接使用brew或者yum或者apt-get进行安装，不过还是建议直接从[nodejs](https://nodejs.org/en/)官方网站直接下载安装，这样才够新:)，然后安装yarn:
```ruby
npm install -g yarn
```

## 已有项目

首先打开你Rails项目的Gemfile，添加下面的gem:
```ruby
gem 'webpacker', '~> 2.0'
```

然后安装gem:
```ruby
bundle
```

然后你就可以在你项目根目录下查看所有的Webpack命令了：
```ruby
rails webpacker
```

然后安装gem:
```ruby
# 如果网络失败请科学上网
rails webpacker:install
```

Rails的webpacker gem已经添加了各个主流JS框架的支持: React, Angular, Vue和Elm，这里以React为例，如果你想要在你的项目中使用React的话，可以再运行下面的命令：
```ruby
rails webpacker:install:react
```

现在项目中已经自动添加了很多配置文件，这里的大部分文件都是不用动的，个别文件会稍后来说明。

下面需要添加ES6中对类似 module.exports 语法的支持，修改你项目根目录中的.babelrc文件，这个是刚刚自动创建的，在presets配置项中添加es2015:
```ruby
...
"react",
"es2015"
...
```

然后安装nodejs的packagebabel-preset-es2015:
```ruby
yarn add babel-preset-es2015
```
至此我们的配置就都结束了。

## 新项目

首先保证你本地的Rails版本 >= 5.1，然后创建项目：
```ruby
# 如果网络失败请科学上网
rails new project-name --webpack=react
```
当然如果你使用其他的JS框架的话上面的react可以替换为vue | angular | elm。


## 使用
我们写的ES6的代码，需要webpack能够实时的编译成ES5的语法，Webpack提供了独立的开发服务器，让我们可以在开发环境下实时的看到编译后的代码，并且提供热加载的机制(通过WebSocket)，首先启动Webpack的开发服务器：
```ruby
bin/webpack-dev-server
```
上面的命令会启动0.0.0.0:8080端口，Webpack就是通过这个server来提供实时编译后的资源，本地的开发环境的资源都会从0.0.0.0:8080加载，当然这里就需要使用Rails已经提供的helper方法来完成。

所有JS和CSS文件此时就可以放在app/javascript这个目录中，要注意这里是app目录下的javascript目录，而不是app/assets下的。

在app/javascript目录中有一个packs目录，所有前端直接引用的文件都需要放在这个目录中，也就是说Webpack会自动编译app/javascript/packs目录下的文件，而子模块可以直接放在app/javascript下，这个原理和Rails的Asset Pipeline类似。

```ruby
如果你想要修改默认的8080号端口，请参考config/webpacker.yml文件中的设置。
```

Rails已经帮我们在app/javascript/packs目录中建立一个测试文件hello_react.jsx:
```ruby
import React from 'react'
import ReactDOM from 'react-dom'
import PropTypes from 'prop-types'

const Hello = props => (
  <div>Hello {props.name}!</div>
)

Hello.defaultProps = {
  name: 'David'
}

Hello.propTypes = {
  name: PropTypes.string
}

document.addEventListener('DOMContentLoaded', () => {
  ReactDOM.render(
    <Hello name="React" />,
    document.body.appendChild(document.createElement('div')),
  )
})
```

我们直接使用这个文件来测试，下面大家需要建立一个测试页面，建立对应的controller和路由，这个就不再赘述了，在你的测试页面中添加以下内容：
```html
<%= javascript_pack_tag 'hello_react' %>
```

注意上面使用的是javascript_pack_tag，不是javascript_include_tag，这个就是专门用来引用webpack资源的helper方法，在本地开发环境下，它会输出绝对地址:
```ruby
http://0.0.0.0:8080/packs/hello_react.js
```

请求到webpack的开发服务器，javascript_pack_tag方法会查找app/javascript/packs目录下的文件，对应的是stylesheet_pack_tag，用来引入packs目录下的css文件，css的使用我们在下一章节中会讲述。

最后启动你的rails server:
```ruby
$ rails s
```
打开你的浏览器访问localhost:3000，访问你的页面，应该就能看到hello David的内容了。


## 代码组织 - 模块化
模块化相信大家都明白，简单说就是把独立的功能放到独立的文件中，同时做到代码重用，可以把前端的不同页面看做不同的大模块，页面的元素看做小模块(可以重用)。

不同的框架就会引入不同的设计思路，模块化也是采用Webpack后更方便的组织代码的一种方式。

全局变量

比如像jQuery，以及已经在项目中引入了React，基本上在所有页面都需要用到的，因此可以把它们放入全局引用中。

打开app/javascript/packs/application.js文件，修改为以下内容：
```ruby
import React from 'react'
import ReactDOM from 'react-dom'

window.React = React;
window.ReactDOM = ReactDOM;
```
我们把React和ReactDOM这两个模块放入了window对象中，也就是全局对象中，然后打开目模板文件app/views/layouts/application.html.erb(或者你自定义的模板文件)，修改为以下内容：

```html
<!DOCTYPE html>
<html>
  <head>
    <title>RailsWebpackReactTemplate</title>
    <%= csrf_meta_tags %>

    <!-- 1 -->
    <%= yield :css %>
  </head>

  <body>
    <%= yield %>
    <!-- 2 -->
    <%= javascript_pack_tag 'application' %>
    <!-- 3 -->
    <%= yield :js %>
  </body>
</html>
```
解释一下上面的代码：

#1: 自定义代码块css，方便之后动态的插入具体页面的样式文件
#2: 采用javascript_pack_tag方法引入上面的app/javascript/packs/application.js文件
#3: 自定义代码块js，方便插入页面的JS模块

注意上面把JS的加载放到了页面的最底部，css放到了上部，这个也是前端页面优化的一个技巧，大家要注意。

然后需要大家建立一个对应的测试页面，controller和路由，这里不再赘述，假如我们建立的controller为WelcomeController，对应的view为app/views/welcome/index.html.erb，首先在app/javascript目录下建立一个新目录welcome，然后在其中建立文件index.js:

```ruby
class WelcomePage extends React.Component {
  render() {
    return (
      <h1>hello <span>{this.props.name}</span></h1>
    )
  }
}

module.exports = WelcomePage
```

然后在app/javascript/packs目录中建立文件welcome.js:
```ruby
import WelcomePage from '../welcome/index'

document.addEventListener('DOMContentLoaded', () => {
  ReactDOM.render(
    <WelcomePage name="eggman.tv" />,
    document.body.appendChild(document.createElement('div')),
  )
})
```

然后在对应的view app/views/welcome/index.html.erb中：
```html
<%= content_for :js do %>
  <%= javascript_pack_tag "welcome" %>
<% end %>
```

重启你的webpack-dev-server进程。
```ruby
每次新建，删除，重命名app/javascript目录下文件的时候都需要重启webpack-dev-server进程
```

打开浏览器访问测试页面，你应该就能看到Hello eggman.tv的内容了。

大家可以参考这种思路来设计你的其他页面。

## 怎样从后端传递变量

上面的app/javascript/packs/welcome.js文件在被前端页面加载时就会运行了，如果需要传递后端的变量到这个JS中应该怎么办呢？

简单来说可以在页面加载完毕后手工初始化JS组件，把后端的变量以JSON的方式传递到JS组件中。

但是默认情况下所有的模块，比如上面的WelcomePage模块在前端的页面中都是不可访问的，也就是说被Webpack编译到了局部变量中，下面就需要修改Webpack的配置来导出模块为全部变量。

打开config/webpacker/shared.js，修改output的配置项：
```ruby
entry: packPaths.reduce(
  // ...
  }, {}
),

output: {
  library: 'App',
  filename: '[name].js',
  path: output.path,
  publicPath: output.publicPath
},
// ...
}
```

添加了library: 'App'的配置，这样通过javascript_pack_tag方法引入的模块都会被放入App这个全局变量中，当然对应引入的JS文件也需要export出对应的变量。

修改app/javascript/packs/welcome.js为：
```ruby
import WelcomePage from '../welcome/index'
import assign from 'object-assign'

const WelcomeApp = {
  init(options) {
    assign(this, {}, options);

    ReactDOM.render(
      <WelcomePage {...this} />,
      document.body.appendChild(document.createElement('div'))
    )
  }
}

module.exports = WelcomeApp
```

上面定义了一个WelcomeApp的对象，内部是一个init的方法，把接收变量options最终又传递进了WelcomeApp组件中。

最后再修改一下视图app/views/welcome/index.html.erb:
```html
<%= content_for :js do %>
  <%= javascript_pack_tag "welcome" %>
  <script>
    App.init(<%= {name: 'eggman.tv'}.to_json.html_safe %>)
  </script>
<% end %>
```

直接调用了App.init方法，把后端变量传递了进去，代码还是比较直观的，我们就不解释了。

这时大家可以重启一下webpack-dev-server，打开页面应该可以看到同样的结果。
```html
这种办法会比较灵活，但是负面效果就是如果你的页面有多个JS文件引入，就会有App变量互相覆盖的影响，解决办法就是在config/webpacker/shared.js中配置的App参数，实际上是可以用[name]来替换为当前的js文件名称，当然需要处理成比较友好的形式，尤其在有子目录的情况下，比如app/javascript/packs/hello/world.js，可以格式化为HelloWorld，这里就不再细述了，大家可以研究一下。但是从实际情况来说，最简单的做法就是可以在每个js文件引用后直接调用App.init的方法就可以了
```

在Webpacker gem的官方文档中有提供怎样[传递变量到JS](https://github.com/rails/webpacker#pass-data-from-view)，简单来说就是把后端变量放入DOM中，然后再读取DOM属性，同样可以作为参考。

## CSS的使用
CSS在Webpack中的使用有两种方式。

### 方法1
直接建立css文件，比如建立文件app/javascript/packs/test.scss:
```html
h1 {
  color: blue;

  span {
    color: green;
  }
}
```
然后在app/views/welcome/index.html.erb最上面添加：
```html
<%= content_for :css do %>
  <%= stylesheet_pack_tag 'test' %>
<% end %>

<%= content_for :js do %>
  # ...
```

### 方法2

采用模块化的方式导入JS中。

首先建立文件app/javascript/welcome/index.scss:
```html
h1 {
  color: blue;

  span {
    color: red;
  }
}
```
然后在app/javascript/packs/welcome.js最上面添加：
```html
import '../welcome/index.scss'
```
最后把app/views/welcome/index.html.erb中的:
```html
<%= stylesheet_pack_tag 'test' %>
```
修改为:
```html
<%= stylesheet_pack_tag 'welcome' %>
```
这里的关键是在哪个JS文件中import的css，最终会被编译成和JS文件同名的.css文件，因此这里直接采用stylesheet_pack_tag方法引入welcome的CSS文件。
```ruby
当然这里不要忘了，你依旧可以采用传统的Asset Pipeline的方式来组织和使用CSS
```

## 引入第三方模块
比如想要使用jQuery，现在就不需要手工下载jquery的源文件，或者采用gem的方式来安装了，可以直接通过yarn安装jquery:
```ruby
yarn add jquery
```
因为jQuery是作为全局使用的，因此你依旧可以修改app/javascript/packs/application.js文件，修改为以下内容：
```ruby
import jQuery from 'jquery';
import React from 'react';
import ReactDOM from 'react-dom';

window.$ = window.jQuery = jQuery;
window.React = React;
window.ReactDOM = ReactDOM;
```
或者你也可以在你需要使用的模块中单独import，不过你要避免在同一个页面的不同JS模块中import同一个模块，否则会造成重复加载。

## 部署
webpacker gem中已经有个现成的rake任务rails webpacker:compile来处理app/javascript/packs下面资源的编译，而且已经和已有的rails assets:precompile任务挂在了一起，因此对于部署我们只需要保证：

服务器中已经安装nodejs, npm和yarn
部署时运行了rails assets:precompile任务

## 其他
如果你本地有多个项目需要修改webpack-dev-server监听端口号可以修改配置文件: config/webpacker.yml，包括默认的app/javascript目录路径等等

如果你不想每次开发都启动webpack-dev-server和rails两个server，你可以使用foreman

不同后缀资源的文件由不同的webpack插件来处理，可以参看config/webpack/loaders目录中各种配置文件
