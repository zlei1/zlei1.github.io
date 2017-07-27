---
layout: post
title: "Web Push Notifications"
date: 2017-07-27
categories: rails
---

# Web Push Notifications: 介绍，环境搭建和server keys的使用

Push Notification（通知推送）在手机端大家都已经非常熟悉了，而Web端基于网页的通知推送现在也慢慢被很多网站应用，尤其是技术类和技术类的服务网站，这个还是因为HTML5和JavaScript的发展太过迅猛，尤其是JavaScript，这个系列文档中我们来研究和实现一下Web端的通知推送。

## 准备
  - Web Push Notifications前端实现
  - 后端推送实现

### 目标
- HTML, JavaScript
- Ruby on Rails


### 学习要求
Web端的通知推送核心还是在于前端的JavaScript，这个文档中后端是用Ruby on Rails来实现的，你可以根据自己的情况(Node.js / Python / PHP)，用什么后端无所谓，后端是作为扩展讲述的。
```html
Web Push Notifications要求localhost环境是可以使用HTTP的，非localhost必须使用HTTPS，在浏览器中测试时这里一定要注意
```

### 浏览器支持
Web端推送得益于HTML的标准，接口还是比较统一的，不同的浏览器的实现基本是一样的，这也是相对于APP端推送的优势；但是我们知道不同浏览器支持的程度是有区别的，目前来说Chrome和Firefox支持的最完善的，IE和Edge还不支持，Safari是使用的苹果自己的[机制](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/NotificationProgrammingGuideForWebsites/PushNotifications/PushNotifications.html#//apple_ref/doc/uid/TP40013225-CH3-SW1)来实现的，因此在这个文档中的代码是完全支持Chrome和Firefox的，IE/Edge/Safari不支持。

### Web Push Notifications使用到的技术

涉及到的知识点有两个，当然都是浏览器端的:

**Service Worker**：简单说就是在浏览器后台运行的一段JavaScript代码，即使用户没有打开你的网站。当从服务器端push消息时，Service Worker会接收到。

**Notification**: 当你在Service Worker中收到推送消息时就可以调用Notification相关的API来展示通知并处理用户的点击事件，比如打开你的网站。

### 数据交互流程
**用户接收推送**

![Image](https://github.com/zlei1/zlei1.github.io/blob/master/post_images/user_accept_push.png?raw=true)

**服务器端发送推送**

![Image](https://github.com/zlei1/zlei1.github.io/blob/master/post_images/service_send_push.png?raw=true)

### 科学上网
因为不同浏览器采用的推送中心的服务器是不一样的，Chrome是google的服务器，在向用户发送推送消息时需要连接google的API，所以这里大家应该都明白，在测试推送时科学上网是必须的，Firefox的推送API暂时没有问题。

## 创建项目
上面说了，我们这里采用的是Ruby on Rails来开发的，你可以根据自己的爱好选择不同的后端语言和框架，原理是一样的。

这里采用的是Rails 5.1的版本，请保证你的环境满足需求:
```ruby
# rails -v
Rails 5.1.1
```
创建项目
```ruby
rails new --skip-test --skip-puma --skip-coffee --skip-turbolinks --skip-bundle web_push_sample
```
ruby gems的安装源国内被墙了，因此上面跳过了自动运行bundle，创建完项目之后，进入到项目目录中，编辑文件Gemfile，修改第一行的gem源地址：
```ruby
source 'https://rubygems.org'
```
改为:
```ruby
 source 'https://gems.ruby-china.org'
```
然后安装gem:
```ruby
bundle
```
为了简化测试，数据库这里采用了默认的SQLite，如果你想用其他数据库，在上面的 rails new 命令中添加 -d mysql参数就行了。

启动:
```ruby
rails s
```
上面的命令会打开你的localhost的3000号端口，然后打开浏览器访问localhost:3000，如果能打开网页就证明项目已经可以了。

## 生成server keys

这里的server keys简单来说就是验证开发者服务器和推送中心服务器的用户名和密码，和我们平时常见的其他API服务的keys没有什么区别。

**Web Push Protocol**

Web Push Notifications的实现也是遵循的专用的[Web Push Protocol](https://tools.ietf.org/html/draft-ietf-webpush-protocol)协议，规定了推送的数据传递方式，这里最关键的就是server keys了，在上面的服务器端推送流程中我们知道，实际上是一个三方的通信过程，浏览器, 推送中心消息服务器和开发者服务器，推送中心服务器就是不同浏览器自己的中心服务器，负责把由开发者服务器发出的推送消息推送到用户的浏览器中，这个过程我们不用关心。

**webpush gem**

生成server keys的方式由很多中，根据你现在不同的开发环境可以选择对应的第三方工具，我们这里用的Ruby，因此可以使用和[webpush](https://github.com/zaru/webpush)这个gem，编辑Gemfile这个文件，添加下面的内容:

```ruby
gem 'webpush'
```
然后再次运行bundle命令。

生成**server keys**

进入rails console:
```ruby
$ rails c
Loading development environment (Rails 5.1.1)
2.4.0 :001 > a = Webpush.generate_key
=> #<Webpush::VapidKey:3ff734519c24 :public_key=BPlmwZjgJ5g0SYNmVOtC5iYQSZxxQJPzinYC1KBnVCjL26w_B8bZDTSCmkuumAjlzK91merXNs_6zDlZKNJ8Na8= :private_key=Pb_a0VHwfCMDibzf69_FeNPuCCkoL5Q_efJQFw0o7t0=>
2.4.0 :003 > a.public_key
=> "BPlmwZjgJ5g0SYNmVOtC5iYQSZxxQJPzinYC1KBnVCjL26w_B8bZDTSCmkuumAjlzK91merXNs_6zDlZKNJ8Na8="
2.4.0 :004 > a.private_key
=> "Pb_a0VHwfCMDibzf69_FeNPuCCkoL5Q_efJQFw0o7t0="
```

手工复制下上面打印出的public_key和private_key，保存在你的config/application.rb文件中:
```ruby
WEB_PUSH_PUBLIC_KEY = 'BOlN-gCdCk7NxYEQAV6xELMj0F1RrVBB0dx3PwXTdwkmnzjvZr2BS03W6p7Oq7b5I3e0i8ybNPEdYA4pBjJwqE8='
WEB_PUSH_PRIVATE_KEY = 'TQO4dfgILSsFuNZ3oH55O2Wk5dYPQzAIPdbz-h8NlLs='

module WebPushSample
  class Application < Rails::Application
    # ...
```

不要忘了把你的rails进程重启一下。

如果你在使用nodejs，可以使用web-push这个package。
## 使用server keys

我们需要在前端用户接收推送时使用public key，因此这里需要来做一些工作，让我们能够在JavaScript读取在config/application.rb中配置的public_key。

把app/assets/javascripts中的application.js重命名为application.js.erb，这样就能够在里面写erb的代码了，然后在application.js.erb中放入以下内容：

```ruby
window.AppConfig = {
  PUBLIC_SERVER_KEY: '<%= WEB_PUSH_PUBLIC_KEY %>',
}
```

上面的代码创建了一个全局的AppConfig的变量，把Rails中所配置的变量设置到AppConfig中，这样就能在前端使用public key了。

然后再建立前端对应的测试页面：

首先修改路由，打开config/routes.rb文件，修改为以下内容：
```ruby
Rails.application.routes.draw do
  root 'welcome#index'
end
```

然后创建首页的Controller，在app/controllers目录中建立文件welcome_controller.rb:
```ruby
class WelcomeController < ApplicationController
end
```

最后在app/views目录中建立目录welcome，并在下面建立文件index.html.erb文件:
```ruby
<h1>Web Push Notification</h1>
```

最后就可以测试下能否在前端页面中访问到AppConfig了，在浏览器中打开localhost:3000，然后在开发者工具(比如你使用的是Chrome)中打开Console的tab，运行下面的JavaScript代码：
```ruby
> AppConfig
Object {PUBLIC_SERVER_KEY: "BOlN-gCdCk7NxYEQAV6xELMj0F1RrVBB0dx3PwXTdwkmnzjvZr2BS03W6p7Oq7b5I3e0i8ybNPEdYA4pBjJwqE8="}
```
这时你应该能看到上面的输出结果，这就表示配置工作已经做完了，接下来就可以正式开发Push Notifications功能了。

## 推送订阅实现
首先在app/assets/javascripts中建立一个JS文件push.js，大部分逻辑都将在这个文件中实现。

打开push.js文件添加以下代码：
```ruby
window.addEventListener('load', function() {
  if ('PushManager' in window) {
    askPermission()
    .then(function() {
      if ('serviceWorker' in navigator) {
        // 这个文件必须和当前网站在同一个域名下
        registerServiceWorker('/service-worker.js');
        navigator.serviceWorker.ready.then(function(serviceWorkerRegistration) {
          subscribePush(serviceWorkerRegistration);
        });
      } else {
        console.log("不支持Service Worker");
      }
    })
    .catch(function(err) {
      console.log('Notification被拒绝');
    })
  } else {
    console.log("不支持Push");
  }
})
```
我们监听了页面的load事件，然后：

- 首先判断当前浏览器是否支持PushManager也就是推送功能
- askPermission: 如果支持，调用 askPermission 方法来弹出通知授权的窗口，让用户来选择时候要接收推送通知
- 再次判断浏览器是否支持 serviceWorker
- registerServiceWorker: 调用 registerServiceWorker 方法注册Service Worker
- navigator.serviceWorker.ready.then: 当Service Worker注册成功后
- subscribePush: 调用 subscribePush 方法来获取当前的浏览器的推送subscription信息，并发送到开发者服务器端保存起来，以便之后发送推送消息时使用
- 以上就是整个通知推送订阅的流程。

在上面使用了三个关键方法 **askPermission, registerServiceWorker, subscribePush**，接下来我们分别讲述下三个方法的实现。

### 通知权限获取
当用户打开浏览器时探测用户的浏览器是否支持Push的功能，然后再请求推送通知的权限。打开push.js，添加下面的方法：
```ruby
function askPermission() {
  return new Promise(function(resolve, reject) {
    // permissonResult: granted | default | denied
    var permissonResult = Notification.requestPermission(function(result) {
      resolve(result);
    });

    if (permissonResult) {
      permissonResult.then(resolve, reject);
    }
  })
  .then(function(permissonResult) {
    if (permissonResult !== 'granted') {
      throw new Error('notification denied!');
    }
  })
}
```
上面的askPermission方法会返回一个Promise对象，因为在推送相关的大部分API中都是采用的Promise的方式调用的，所以这里也采用同样的方式来定义， 方法本身主要就是调用了 Notification.requestPermission API来请求当前通知推送的权限，这时用户会看到一个小型的浏览器弹出窗口，来确认或者拒绝通知推送，和手机端的体验基本是一样的。

授权结果保存在permissonResult中，取值有三种情况: granted | default | denied，我们只关心granted的情况，其他都为没有被授予权限。

如果用户确认的话 askPermission 会返回一个Promise对象，如果拒绝的话会抛出异常，需要在接下来的代码中捕获这个异常。待会会使用这个方法。

### 注册Service Worker
在第一章节中提到通知推送需要配合Service Worker来使用，期望的结果是即使用户没有打开我们的网站也能收到消息的推送，Service Worker就是运行在浏览器后台的一段JavaScript代码，当浏览器收到推送的消息时会触发规定的事件，我们就可以在Service Worker中监听这些事件，然后做出处理，这个和在网页中的事件操作很相似。

打开push.js文件，添加下面的方法：
```ruby
function registerServiceWorker(workerFile) {
  return navigator.serviceWorker.register(workerFile)
  .then(function(registration) {
    console.log('service worker registered!')
    return registration;
  })
  .catch(function(err) {
    console.error('unable to register service worker', err);
  });
}
```
registerServiceWorker方法接收一个参数，这个参数为一个JS文件的地址，也就是在Service Worker中要运行的代码，比如这样来调用:
```ruby
registerServiceWorker('/service-worker.js');
```

这个方法本身比较简单，直接调用了navigator.serviceWorker.register浏览器内置方法来注册一个Service Worker，如果注册失败会在Console中打印出错误信息。

接下来就需要实现Service Worker的逻辑了。

### Service Worker实现
建立一个文件，命名为service-worker.js，把这个文件放入public目录当中。
```ruby
这里一定要把 service-worker.js 放入你项目的根路径当中，也就是Rails的public目录当中，而不能放入上面的app/assets/javascripts目录中，因为这和Service Worker的设计和不同浏览器的实现有关，所以为了在不同浏览器中行为一致，一定要放到你项目的根路径当中，比如 http://localhost:3000/service-worker.js，不能放到子目录中。
```

首先在service-worker.js中添加下面的方法：
```ruby
self.addEventListener('push', function(event) {
  // console.log('push event: ', event);
  var data = event.data.json();

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: data.icon,
      tag: data.tag,
      data: {url: data.url}
    })
  );
});
```

首先监听了push事件，就是当收到推送消息时，event变量中能够获得推送消息的内容本身，也就是说在发送推送时，可以指定一个JSON类型的消息体，然后在event中能够反向取得消息的内容。

然后调用waitUntil方法，在内部调用了showNotification方法来弹出系统的通知框，showNotification 方法返回的同样是一个Promise对象，在showNotification方法中可以指定通知框的具体内容，这里直接根据event中的数据来显示通知内容，第一个参数为标题，第二个参数为一个对象：

- body: 通知框内容
- icon: 在通知框中显示的图标，这个一般可以指定为一个URL地址
- tag: 通知分类信息，这个可以根据自己的实际情况随便设置就行
- data: 这个为自定义的属性，可以在这个属性中放置需要的自定义信息，比如这里可能需要传递当前信息的URL地址

因为Service Worker为由浏览器控制的后台任务，这里采用 waitUntil 方法是用来把 showNotification 方法放入浏览器的任务队列当中，如果这样:
```ruby
self.addEventListener('push', function(event) {
  // console.log('push event: ', event);
  var data = event.data.json();

  self.registration.showNotification(data.title, {
    body: data.body,
    icon: data.icon,
    tag: data.tag,
    data: {url: data.url}
  })
});
```
是不行的，一定要注意。

当通知显示的时候，用户是可以和通知框进行交互的，比如点击通知内容，或者关闭通知，当点击通知内容时就可以打开对应的网页了，因此需要来处理点击的事件。

再次打开service-worker.js，添加下面的代码：
```ruby
self.addEventListener('notificationclick', function(event) {
  event.notification.close();

  event.waitUntil(clients.matchAll({
    type: 'window'
  }).then(function(clientList) {
    if (event.notification.data && event.notification.data.url) {
      return clients.openWindow(event.notification.data.url);
    } else {
      return clients.openWindow("https://eggman.tv");
    }
  }));
});
```
上面的方法监听了notificationclick事件，也就是通知点击事件，同样可以在回调方法的event参数中取得当前通知的消息内容，这里主要是取出了自定义数据data中的url，然后再控制浏览器打开url。比如推送了一篇最近技术文章的消息，同时在消息中传递了文章的URL，然后这里就可以取得这个URL，当用户点击通知时就能直接打开对应的页面了。

这里根据data是否存在做了不同的处理，因为在Firefox中，data属性有丢失的情况。

### 获取推送的subscription信息

我们已经获取到了推送的权限，注册了Service Worker来处理推送的消息，接下面最关键的问题是怎样给用户发送推送呢，这就是最后一个方法subscribePush了。

重新打开push.js文件，添加下面的代码：
```ruby
function urlBase64ToUint8Array(base64String) {
  var padding = '='.repeat((4 - base64String.length % 4) % 4);
  var base64 = (base64String + padding).replace(/\-/g, '+').replace(/_/g, '/');

  var rawData = window.atob(base64);
  var outputArray = new Uint8Array(rawData.length);

  for (var i = 0; i < rawData.length; ++i) {
      outputArray[i] = rawData.charCodeAt(i);
  }
  return outputArray;
}

function subscribePush(registration) {
  var subscribeOptions = {
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(AppConfig.PUBLIC_SERVER_KEY)
  };
  return registration.pushManager.subscribe(subscribeOptions)
  .then(function(pushSubscription) {
    console.log('push subscription: ', JSON.stringify(pushSubscription));

    return pushSubscription;
  });
}
```

我们添加了两个方法，第一个 urlBase64ToUint8Array 是用来把一个base64的字符串转化为整数数组的，因为根据Web Push协议的要求需要把public key来转化整数数组才能发送到推送中心服务器，这个方法就不多说了。

关键是subscribePush方法，在这个方法中调用了registration.pushManager.subscribe方法，其中registration是在load中Service Worker注册成功后返回的对象，registration.pushManager.subscribe就是用来注册并返回当前浏览器的推送的subscription信息，这个方法需要传递一个参数，也就是subscribeOptions。

subscribeOptions:

- userVisibleOnly: 通知用户是否课件，目前Chrome和Firefox只支持可见的通知，也就是这个参数必须为true
- applicationServerKey: 这个就是在第一章中配置的public key了，这里注意要使用urlBase64ToUint8Array方法转化为整数数组。

调用registration.pushManager.subscribe方法简单来说就是向推送的中心服务器注册并取得当前浏览器的推送识别参数，就是和当前浏览器绑定的全局唯一的一组Hash，我们称为subscription信息，也就是上面的 pushSubscription 变量。pushSubscription的结构为这样的：
```ruby
{
  "endpoint": "https://fcm.googleapis.com/fcm/send/cBBFXqYE6Ew:APA91bFKMLXp9lDwbjiZxjLIRdl1CjKUeDwXljGqdaEmdVdywdntFsPZE1KMeeDr13YIsaAzj2up8fwHTWCjsongLxdxb_vsCdVk",
  "keys":{
    "p256dh": "BHAPGwdf4j81Z_nZtgaHhoXpLFq-EvfM1aZGyhHrdbj2loRsXwm0_7Box0jCR8oXtsIRjh4F3zvw9uzdyddh-9A=",
    "auth": "y7PfWaVteugvp8C37NzA5Q=="
  }
}
```
这个数据待会在测试中就能看到。

目前为止我们已经完成了所有前端工作，简单来说就是两个JS文件：app/assets/javascripts/push.js 和 public/service-worker.js。

## 后端调整

然后就是怎样在页面中引入 push.js 文件了，打开 app/assets/javascripts/application.js.erb，删除其中的：
```ruby
 //= require_tree .
```

这行代码，最终的 app/assets/javascripts/application.js.erb 为这样：
```ruby
//= require rails-ujs

window.AppConfig = {
 PUBLIC_SERVER_KEY: '<%= WEB_PUSH_PUBLIC_KEY %>',
}
```

打开 config/initializers/assets.rb 文件，修改precompile行的配置为这样：
```ruby
Rails.application.config.assets.precompile += %w(
  push.js
)
```

打开 app/views/layouts/application.html.erb 文件，修改为：
```html
<!DOCTYPE html>
<html>
  <head>
    <title>WebPushSample</title>
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag    'application', media: 'all' %>
    <%= javascript_include_tag 'application' %>
  </head>

  <body>
    <%= yield %>
    <%= yield :js %>
  </body>
</html>
```

最后再打开 app/views/welcome/index.html.erb，修改为:
```html
<h1>Web Push Notification</h1>
<%= content_for :js do %>
    <%= javascript_include_tag "push" %>
<% end %>
```

## 测试和发送推送
重启你的 rails s 进程，打开浏览器的开发者工具中的console，访问localhost:3000，这时你应该就能看浏览器弹出通知推送的窗口:

![Image](https://github.com/zlei1/zlei1.github.io/blob/master/post_images/notification.png?raw=true)

点击同意之后，在console中就能看到打印出的subscription信息了，结构和上面提到的 pushSubscription 一样:
```ruby
{
  "endpoint": "https://fcm.googleapis.com/fcm/send/cBBFXqYE6Ew:APA91bFKMLXp9lDwbjiZxjLIRdl1CjKUeDwXljGqdaEmdVdywdntFsPZE1KMeeDr13YIsaAzj2up8fwHTWCjsongLxdxb_vsCdVk",
  "keys":{
    "p256dh": "BHAPGwdf4j81Z_nZtgaHhoXpLFq-EvfM1aZGyhHrdbj2loRsXwm0_7Box0jCR8oXtsIRjh4F3zvw9uzdyddh-9A=",
    "auth": "y7PfWaVteugvp8C37NzA5Q=="
  }
}
```

如果你是在Firefox中测试的话，会看到endpoint的URL地址会不一样，这也就是我们提到的不同浏览器的中心服务器不一样，大家只用把这个信息拷贝一下，发送推送就是使用的这个信息，接下来就可以测试了：

进入 rails c 中：
```ruby
> message = {
  title: "eggman.tv 通知",
  body: "web push notifications",
  icon: "https://eggman.tv/images/web-push-icon.png",
  url: "https://eggman.tv"
}
> sub_data = {
  endpoint："https://fcm.googleapis.com/fcm/send/c170mQtn8y4:APA91bH4pT6po7s0IJ8yFbYcw4Xl07qv8Kxl3muhUxwnleL9ShBSUwrrRbGXylK9StOhvWFBJy1oUo0f_0Ky0SklHgf388K2yrBnV4stdnuvD-DksJwQbzFBRL_LiP1ba86eX7li9-E",
  keys： {
    p256dh： "BBeXsuEI7lv1zg89bVDRLmWE257Rb6a6Y3iv0rNrbSILLtAmDt17CwB-zywVztjPJupnPxd0E9cJhI7ldjpTSQ=",
    auth： "ijEK4vIGJ3vDCeeJbopDtg=="
  }
}
> Webpush.payload_send(
  endpoint: sub_data[:endpoint],
  message: message.to_json,
  p256dh: sub_data[:keys][:p256dh],
  auth: sub_data[:keys][:auth],
  vapid: {
    subject: 'service@eggman.tv',
    public_key: WEB_PUSH_PUBLIC_KEY,
    private_key: WEB_PUSH_PRIVATE_KEY
  }
)

```
上面的 message 为发送的测试信息，sub_data为刚刚在浏览器console中复制的subscription信息，最后调用webpush gem的payload_send方法来发送推送，参数都是比较直观的，其中vapid.subject需要输入一个Email地址，这也是Web Push协议要求的，大家可以随便填写一个email就可以了。

运行完上面的代码你应该就已经成功收到浏览器的推送消息了。

## Debug
JavaScript的调试可以在Chrome的开发者工具中进行，打断点，这个就不再多做介绍了，这里主要是Service Worker的调试给大家介绍一下。

在Chrome的开发者工具中可以点击Application的tab，然后选择左侧菜单中的Application - Service Worker来查看当前在该域名下注册的所有的Service Worker，以及进行一些管理：

![Image](https://github.com/zlei1/zlei1.github.io/blob/master/post_images/debug.png?raw=true)

在开发过程中可以点击右侧的Unregister按钮来删除Service Worker，然后刷新页面重新注册。

## 保存推送的subscription信息
```ruby
{
   "endpoint": "https://fcm.googleapis.com/fcm/send/cBBFXqYE6Ew:APA91bFKMLXp9lDwbjiZxjLIRdl1CjKUeDwXljGqdaEmdVdywdntFsPZE1KMeeDr13YIsaAzj2up8fwHTWCjsongLxdxb_vsCdVk",
   "keys":{
     "p256dh": "BHAPGwdf4j81Z_nZtgaHhoXpLFq-EvfM1aZGyhHrdbj2loRsXwm0_7Box0jCR8oXtsIRjh4F3zvw9uzdyddh-9A=",
     "auth": "y7PfWaVteugvp8C37NzA5Q=="
   }
 }
```
发送推送就需要这个信息，所以现在就来保存这个信息到数据库中。

### WebPushEndpoint模型
采用保存subscription信息的模型为WebPushEndpoint，在项目的根目录中运行下面的命令来创建模型：
```ruby
rails g model web_push_endpoint
```
然后编辑app/models/web_push_endpoint.rb文件为：
```ruby
class WebPushEndpoint < ApplicationRecord

  belongs_to :user

  def self.find_or_create! options
    unless record = find_by(endpoint: options[:endpoint])
      record = create! user: options[:user],
        endpoint: options[:endpoint],
        subscription: options.except(:user).to_json
    end

    record
  end

end
```
在模型中定义了find_or_create!方法，这个方法判断了endpoint是否存在，如果不存在会根据传递进来的subscription信息来创建新的记录。这里有一点要注意的是我们把subscription信息转化为了JSON字符串保存在了subscription字段中。我们会在controller中调用这个方法。

对应的数据库移植为：
```ruby
class CreateWebPushEndpoints < ActiveRecord::Migration[5.1]
  def change
    create_table :web_push_endpoints do |t|
      t.integer :user_id
      t.string :endpoint
      t.text :subscription
      t.timestamps null: false
    end

    add_index :web_push_endpoints, [:user_id]
    add_index :web_push_endpoints, [:endpoint], unique: true
  end
end
```
字段： user_id：用来管理系统中的用户模型 endpoint: subscription信息中的 endpoint URL，因为subscription信息和浏览器是绑定的，endpoint URL是全局唯一的，我们把这个作为模型的唯一主键。用户每次在打开浏览器时都会发送一遍subscription信息，此时只有在endpoint URL不存在的情况下才会创建新记录，同一个浏览器每次发送的subscription信息是一样的，除非当用户取消授权的情况下才会发生变化。 subscription: 这个字段用来存储整个subscription信息，当然这里endpoint URL是作为冗余信息再存储一遍。

### 发送subscription信息到后端

subscription 是在调用 subscribePush这个方法时获取的，修改 push.js 文件中的的这个方法为：
```ruby
function subscribePush(registration) {
  var subscribeOptions = {
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(AppConfig.PUBLIC_SERVER_KEY)
  };
  return registration.pushManager.subscribe(subscribeOptions)
  .then(function(pushSubscription) {
    fetch('/web_push_endpoints',
      {method: 'POST', body: JSON.stringify(pushSubscription)})
    .then(function(response) {
      if (response.ok == true) {
        response.json().then(function(data){
          if (data.status == 'ok') {
            console.log('push subscription: ', JSON.stringify(pushSubscription));
          }
        })
      }
    });

    return pushSubscription;
  });
}
```
这里采用HTML5的 fetch 方法把获取到的subscription信息发送到后端的/web_push_endpoints URL，实现逻辑是比较简单的，就不再详细解释了。

然后添加对应的controller逻辑，首先修改config/routes.rb文件，添加路由：
```ruby
resources :web_push_endpoints, only: [:create]
```

然后在app/controllers目录下面创建对应的web_push_endpoints_controller.rb:
```ruby
class WebPushEndpointsController < ApplicationController

  skip_before_action :verify_authenticity_token, only: [:create]

  def create
    data = request.body.read
    subscription = JSON.parse(data)
    subscription.deep_symbolize_keys!
    subscription.merge!(user: current_user) if logged_in?

    WebPushEndpoint.find_or_create! subscription
    render json: {status: 'ok'}
  end

end
```
上面调用了模型WebPushEndpoint中的find_or_create!方法来创建subscription信息。

至此我们已经完成了后端subscription信息的创建功能，大家可以重新刷新一下前端页面，应该可以看到后台的web_push_endpoints表中新增了一条数据记录，然后发送推送信息时就可以从WebPushEndpoint模型中读取了。

### 发送推送
批量发送推送的话就可以这样：
```ruby
WebPushEndpoint.find_each do |web_push_endpoint|
  sub_data = JSON.parse web_push_endpoint.subscription

  message = {
    title: options[:title],
    body: options[:body],
    icon: options[:icon],
    tag: options[:tag],
    url: options[:url],
  }
  Webpush.payload_send(
    endpoint: sub_data['endpoint'],
    message: message.to_json,
    p256dh: sub_data['keys']['p256dh'],
    auth: sub_data['keys']['auth'],
    vapid: {
      subject: 'service@eggman.tv',
      public_key: WEB_PUSH_PUBLIC_KEY,
      private_key: WEB_PUSH_PRIVATE_KEY
    }
  )
end
```
在线上的环境中可以结合一些异步任务组件比如Sidekiq来实现批量的异步发送，这里就不再深入说明了.
