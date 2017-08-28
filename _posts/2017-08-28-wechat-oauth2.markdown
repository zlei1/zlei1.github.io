---
layout: post
title: "wechat-oauth2"
date: 2017-08-28
categories: rails
---

## Rails 5 + devise + omniauth-wechat-oauth2

### Gemfile
```ruby
gem “omniauth-wechat-oauth2”
```

### Route
```ruby
devise_for :users, controllers: {
    registrations: "user/registrations",
    omniauth_callbacks: 'omniauth_callback'
  }
```

### Config (config/initializers/devise.rb)
```ruby
# Use this hook to configure devise mailer, warden hooks and so forth.
# Many of these configuration options can be set straight in your model.
Devise.setup do |config|

  config.omniauth :wechat,
  ENV['WECHAT_MP_APPID'],
  ENV['WECHAT_MP_SECRET'],
  :authorize_params => {:scope => "snsapi_userinfo"}

# ....

```

### 微信配置
![Image](https://github.com/zlei1/zlei1.github.io/blob/master/post_images/wechat_oauth1.png?raw=true)

![Image](https://github.com/zlei1/zlei1.github.io/blob/master/post_images/wechat_oauth2.png?raw=true)

### Controller
```ruby
class OmniauthCallbackController < Devise::OmniauthCallbacksController

  def wechat
    @user = User.from_omniauth(request.env["omniauth.auth"])
    if @user.persisted?
      sign_in_and_redirect @user, :event => :authentication
    else
      session["devise.wechat_data"] = request.env["omniauth.auth"]
      redirect_to new_user_registration_url
    end
  end

end
```
sign_in_and_redirect 这个方法是 devise 提供的。

### User Model
```ruby
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :omniauthable,
         :omniauth_providers => [:wechat]

  def self.from_omniauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
        user.provider = auth.provider
        user.uid = auth.uid
        user.email = auth.info.email || "#{auth.uid}@herego.com"
        user.name = auth.info.nickname
        user.portrait_url = auth.info.headimgurl
        user.password = Devise.friendly_token[0,20]
        user.save
      end
  end

end
```

- 重要的部分1：:omniauthable.
- 重要的部分2：omniauth_providers => [:wechat]
注意这个 providers 其实不写也成，但是如果你写了什么其他 facebook twitter 之类的，暂时又不想显示 facebook twitter，这里又不写这么个过滤，那么<%= render “devise/shared/links” %>
就会把 facebook twitter 也输出出来。
- 重要的部分3：def self.from_omniauth(auth)

### 访问
/users/auth/wechat
