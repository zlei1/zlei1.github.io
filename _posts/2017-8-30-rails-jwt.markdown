---
layout: post
title: "JWT构建Rails API授权登录"
date: 2017-08-30
categories: rails
---

### JWT

在JWT的规范定义中，它由头部，载荷和签名，三部分字符串组成其中前两部分是用JSON对象进过Base64编码而来的。

**头部** 是由typ和alg两部分组成，typ 表示自己是一个JWT，alg表示签名使用了什么算法。

```ruby
{
  "typ": "JWT",
  "alg": "HS256"
}
```

经过base64编码后的结果就是：
```ruby
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

**载荷** 是JWT中真正承载用户信息的部分，它也是一个json对象，由自定义部分和规范定义部分组成

在[JWT规范定义](https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32) 中描述的几个可选的信息
```ruby
{
    "iss": "JWT-Rails-Server", // 签发者
    "aud": "www.baidu.com", // 接收者
    "iat": 1472263256, // JWT 签发的时间
    "exp": 1472522525, // 过期时间
    "sub": "jwt@baidu.com" // JWT对应的用户
      "user_id": 1211 // 自定义
}
```
我们还可以在上面的JSON中添加我们自定义的部分。

最后载荷也是需要通过Base64进行编码的：
```ruby
eyJpc3MiOiJKV1QtUmFpbHMtU2VydmVyIiwiYXVkIjoid3d3LmJhaWR1LmNv\nbSIsImlhdCI6MTQ3MjI2MzI1NiwiZXhwIjoxNDcyNTIyNTI1LCJzdWIiOjEx\nMjF9\n
```

**签名** 就是将 头部和载荷使用 "." 连接成的字符串 再使用我们自己提供的一个密钥 进行HS256加密后的字符串。

如果是用 "jwt-rails" 作为密钥的话，签名：
```ruby
cd5a6c7a135e811477918c5c0f864582bced820ff6b5ed6766974c3ef8ca9773
```

JWT的 安全重点就是在签名的密钥上，如果仅仅有服务器端知道密钥的话，其他人如果获得了 JWT字符串并对它进行了篡改，那么它发送到服务端后就无法通过密钥加密的签名验证，这样就有效的阻止这类安全问题。但是要注意的是，载荷部分所携带的信息是Base64编码"非加密"，所以我们不要把有关用户的敏感信息存放在其中，一般在API接口开发中仅需要存放，能够标识用户的ID或UUID即可。

### JWT in Rails API

[JWT-Ruby](https://github.com/jwt/ruby-jwt) gem 已经帮我们实现JWT规范的库，现在只有使用它提供的API就可以使用 JWT 进行开发了。

我们接下来就，开发一个具有rails 5 API的后端示例应用。
```ruby
rails new jwt_rails --api
```

再添加gem 到 Gemfile
```ruby
gem 'jwt'
gem 'bcrypt'
```

我们先创建一个users controller，users_controller 会返回有关用户的信息，但是求助这个
```ruby
rails g controller users
```

然后在创建 User 模型
```ruby
rails g model User username:string email:string password_digest:string
```

填充User模型的代码
```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
end
```

console中创建一个用户
```ruby
2.3.0 :003 > User.create(username: 'json', email: 'json@gmail.com', password: '12345', password_confirmation: '12345')
 => #<User id: 1, username: "json", email: "json@gmail.com", created_at: "2016-08-27 03:19:44", updated_at: "2016-08-27 03:19:44", password_digest: "$2a$10$3KrwpUYEgYfBJTBJJMX.5uU9d14hs91rf5Fnt8cUEvZ...">
```

接下来就是把JWT集成到项目中，先创建叫Token的包装类，其中使用了 Rails的secret key 作为JWT的加密密钥。
```ruby
# app/models/token.rb
class Token
  def self.encode(payload)
    JWT.encode(payload, Rails.application.secrets.secret_key_base)
  end

  def self.decode(token)
    HashWithIndifferentAccess.new(JWT.decode(token, Rails.application.secrets.secret_key_base)[0])
  rescue
    nil
  end
end
```

再修改User模型，让其支持通过id作为承载信息，然后生成的token的方法。
```ruby
class User < ApplicationRecord
  has_secure_password

  def token
    {
      token: Token.encode(user_id: self.id)
    }
  end

  def to_json
    self.slice(:username, :email)
  end
end
```

在 app/controllers/application_controller.rb 中添加验证token是否有效的方法。
```ruby
class ApplicationController < ActionController::API

  attr_accessor :current_user

  protected

  def authenticate!
    render_failed and return unless token?
    @current_user = User.find_by(id: auth_token[:user_id])
  rescue JWT::VerificationError, JWT::DecodeError
    render_failed
  end

  private

  def render_failed(messages = ['验证失败'])
    render json: { errors: messages}, status: :unauthorized
  end

  def http_token
    @http_token ||= if request.headers['Authorization'].present?
                      request.headers['Authorization'].split(' ').last
                    end
  end

  def auth_token
    @auth_token ||= Token.decode(http_token)
  end

  def token?
    http_token && auth_token && auth_token[:user_id].to_i
  end

end
```

在app/controllers/authentication_controller.rb 中处理用户登录然后返回授权token。
```ruby
class AuthenticationController < ApplicationController

  def create
    if user = User.find_by(username: params[:username]).try(:authenticate, params[:password])
      render json: user.token
    else
      render json: {errors: ['用户名或密码错误']}, status: :unauthorized
    end
  end

end
```

然后通过授权的token 访问用户信息 app/controllers/users_controller.rb 其中使用了我们在application_controller定义的验证方法，作为前置过滤器。
```ruby
class UsersController < ApplicationController
  before_action :authenticate!

  def index
    render json: current_user.to_json
  end

end
```

最后添加路由：
```ruby
Rails.application.routes.draw do
  resources :users, only: :index
  resources :authentication, only: :create
end
```

启动服务
```ruby
rails s
```

下面我们使用curl来请求验证一下我们刚刚写的API。

登录验证：
```ruby
curl -X POST -d username="json" -d password="12345" http://localhost:3000/authentication
```

返回结果：
```ruby
{"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOjF9.j8-GAiEQ2LIzC8GdbqZ6H5aUA32Mux07uaY9RfOQrx8"}
```

如果不用Token直接访问用户信息的话。
```ruby
curl http://localhost:3000/users
```

会直接返回验证失败：
```ruby
{"errors":["验证失败"]}
```

使用Token请求用户信息
```ruby
curl --header "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOjF9.j8-GAiEQ2LIzC8GdbqZ6H5aUA32Mux07uaY9RfOQrx8" http://localhost:3000/users
```

返回结果：
```ruby
{
  "username":"json",
  "email":"json@gmail.com"
}
```
通过验证。

**注销**

JWT 对应注销已签发的token有三种方式：

payload中的exp过期时间
客户端丢弃本地缓存的token
服务端维护一个token废弃池
exp

使用JWT规范定义中payload可以携带的过期时间键值对，我们可以对上面的程序做一些修改。

首先在app/models/token.rb 中修改encode方法：
```ruby
def self.encode(payload)
  payload.merge!(exp: (Time.now.to_i + 3600)) # 添加过期时间为一小时
  JWT.encode(payload, Rails.application.secrets.secret_key_base)
end
```

然后再修改验证过滤器，让它支持捕获token过期异常
```ruby
rescue JWT::ExpiredSignature
  render_failed ['授权已过期']
end
```

最后如果请求发送的token过期结果就是：
```ruby
{"errors":["授权已过期"]}
```

废弃池

在严格要求废弃指定的token的场景下，推荐使用Redis维护这样一个废弃池，在每次需要验证的请求中，过滤掉已经废弃的token。

客户端丢弃

这是成本最低的方式，把任务分散到各个客户端，可以很好的与现在的移动端开发配合，每次用户注销只要删除本地存放的token即可。

**结论**

JWT作为一种轻量级的令牌验证方案，是很轻便的，使用它，服务端就可以无需维护令牌的状态，同时也解决了多系统的同步登录问题。
