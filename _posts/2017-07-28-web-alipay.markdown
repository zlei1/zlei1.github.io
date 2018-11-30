---
layout: post
title: "alipay"
date: 2017-07-28
categories: rails
---
## [蚂蚁金服开发文档](https://openhome.alipay.com/developmentDocument.htm)

### 创建payment模型
```ruby
rails g  model payment

create_table :payments do |do|
  t.integer :uesr_id
  t.string :payment_no,transaction_no
  t.string :status, default: 'initial' #未付款状态为initial
  t.decimal :total_money, precision:10, scale: 2
  t.datetime :payment_at
  t.text :raw_response
  t.timesstamps
end

#user_id 用户id
#payment_no 支付单号
#transaction_no 支付宝返回的流水号
#status 支付状态
#total_money 支付总金额
#payment_at 支付成功后要刷新的时间戳
#raw_response :支付宝返回结果的数据集（返回的json数据）
```

### 订单表添加字段
```ruby
rails g migration add_order_payment_id_column

def change
  add_column :orders, :payment_id, :integer
  add_column :orders, :status, :string, defalut: 'initial'
end
```


### user.rb
```ruby
  has_many :payments
```

### order.rb
```ruby
  belongs_to :payment

  #定义常量
  module OrderStatus
    Initial = 'initial'
    Paid = 'paid'
  end

  #判断订单是否支付
  def is_paid
    self.status == OrderStatus::Paid
  end
```

### payment.rb
```ruby
  belongs_to :user
  has_many :orders

  module PaymentStatus
    Initial = 'initial'
    Success = 'success'
    Failed = 'failed'
  end

  #create之前生成payment_no 支付单号
  before_create :gen_payment_no

  def gen_payment_no
    self.payment_no = generate_utoken(32)
  end

  def self.generate_utoken len = 8
    a = lambda { rand(36).to_s(36) }
    token = ""
    len.times { |t| token << a.call.to_s }
    token
  end
```

### routes.rb
```ruby
  resources :payments, only: [:index] do
    collection do
      get :generate_pay
      get :pay_return
      get :pay_notify
      get :success
      get :failed
    end
  end
```

### payments_comtroller.rb
```ruby
  def index

  end

  def generate_pay
    #接收需要支付的订单
    #order_nos需要支付的所有订单号
    orders = current_user,orders.where(order_no params[:order_nos].split(','))
    #生成付款记录
    payment = Payment.create_from_orders!(current, orders) #create_from_orders!方法
    #跳转
    redirect_to payments_path(payment_no: payment.payment_no) #添加参数
  end
```

### payment.rb
```ruby
  #添加create_from_orders!方法
  def self.create_from_orders! user, *orders
    orders.flatten! #处理成一维数组
    payment = nil
    transaction do
      #创建付款记录
      payment = user.payments,create!(
        total_money: orders.sum(&:total_money)
      )
      orders,each do |order|
        if order.is_paid?
          #如果订单已经支付过了，抛出异常
          raise "rorder #{order.order_no} has already paid"
        end
        order.payment = payment
        order.save!
      end
    end

    payment
  end
```

### config/application.rb
```ruby
  ENV['ALIPAY_PID'] = 'YOUR-ALIPAY-PARTNER-ID'
  ENV['ALIPAY_MD5_SECRET'] = 'YOUR-ALIPAY-MD5-SECRET'
  ENV['ALIPAY_URL'] = 'https://openapi.alipay.com/gateway.do'
  ENV['ALIPAY_RETURN_URL'] = 'http://localhost:3000/payments/pay_return'
  ENV['ALIPAY_NOTIFY_URL'] = 'http://localhost:3000/payments/pay_notify'
```


### payments_comtroller.rb
```ruby
class PaymentsController < ApplicationController

  #protect_from_forgery 防止攻击
  protect_from_forgery except: [:alipay_notify]

  #验证用户必须登入
  before_action :auth_user, except: [:pay_notify]
  before_action :auth_request, only: [:pay_return, :pay_notify]
  before_action :find_and_validate_payment_no, only: [:pay_return, :pay_notify]

  def index
    @payment = current_user.payments.find_by(payment_no: params[:payment_no])
    @payment_url = build_payment_url
    @pay_options = build_request_options(@payment) #表单中提交到支付宝的参数
  end

  #order页面，表单提交执行 redirect_to generate_pay_payments_path(order_nos: orders.map(&:order_no).join(',')) 订单结算页面跳转
  def generate_pay
    orders = current_user.orders.where(order_no: params[:order_nos].split(','))
    payment = Payment.create_from_orders!(current_user, orders)
    redirect_to payments_path(payment_no: payment.payment_no)  #重定向到index页面中
  end

  #同步通知
  def pay_return
    do_payment
  end

  #异步通知
  def pay_notify
    do_payment
  end

  def success
  end

  def failed
  end

  private
  #判断当前支付是否是成功的
  def is_payment_success?
    #trade_status 支付宝参过来的参数，判断是是否为TRADE_SUCCESS TRADE_FINISHED其中的一个
    %w[TRADE_SUCCESS TRADE_FINISHED].include?(params[:trade_status])
  end

  def do_payment
    unless @payment.is_success? # 避免同步通知和异步通知多次调用
      if is_payment_success?
        @payment.do_success_payment! params
        redirect_to success_payments_path
      else
        @payment.do_failed_payment! params
        redirect_to failed_payments_path
      end
    else
     redirect_to success_payments_path
    end
  end

  def auth_request
   #build_is_request_from_alipay 验证当前的请求是否来自于支付宝
    unless build_is_request_from_alipay?(params)
      Rails.logger.info "PAYMENT DEBUG NON ALIPAY REQUEST: #{params.to_hash}"
      redirect_to failed_payments_path
      return
    end
    # build_is_request_sign_valid
    unless build_is_request_sign_valid?(params)
      Rails.logger.info "PAYMENT DEBUG ALIPAY SIGN INVALID: #{params.to_hash}"
      redirect_to failed_payments_path
    end
  end

  def find_and_validate_payment_no
   #out_trade_no支付宝返回的参数，支付单号
    @payment = Payment.find_by_payment_no params[:out_trade_no]
    unless @payment
      if is_payment_success?
        # TODO
        render text: "未找到支付单号，但是支付已经成功"
        return
      else
        render text: "未找到您的订单号，同时您的支付没有成功，请返回重新支付"
        return
      end
    end
  end

  def build_request_options payment
    # opts:
    #   service: create_direct_pay_by_user | mobile.securitypay.pay
    #   sign_type: MD5 | RSA
    pay_options = {
      "service" => 'create_direct_pay_by_user', #固定的字符串，指定当前的支付类型
      "partner" => ENV['ALIPAY_PID'], #ALIPAY_PID
      "seller_id" => ENV['ALIPAY_PID'],
      "payment_type" => "1",
      "notify_url" => ENV['ALIPAY_NOTIFY_URL'],
      "return_url" => ENV['ALIPAY_RETURN_URL'],
      "anti_phishing_key" => "",
      "exter_invoke_ip" => "",
      "out_trade_no" => payment.payment_no,
      "subject" => "商品购买",
      "total_fee" => payment.total_money,
      "body" => "商品购买",
      "_input_charset" => "utf-8",
      "sign_type" => 'MD5', #编码方式
      "sign" => ""
    }

    pay_options.merge!("sign" => build_generate_sign(pay_options)) #build_generate_sign 根据sign_type生成hash
    pay_options
  end

  def build_payment_url
    "#{ENV['ALIPAY_URL']}?_input_charset=utf-8"
  end

  def build_is_request_from_alipay? result_options
  #如果notify_id为空的话 表示该请求不是来自支付宝
    return false if result_options[:notify_id].blank?
    #RestClient gem 'rest-client' 调用http接口的第三方库
    body = RestClient.get ENV['ALIPAY_URL'] + "?" + {
      service: "notify_verify",
      partner: ENV['ALIPAY_PID'],
      notify_id: result_options[:notify_id]
    }.to_query

    body == "true"
  end

  def build_is_request_sign_valid? result_options
    options = result_options.to_hash
    options.extract!("controller", "action", "format") #变量排除的操作，排除掉rails添加的参数

    if options["sign_type"] == "MD5"
      options["sign"] == build_generate_sign(options)
    elsif options["sign_type"] == "RSA"
      build_rsa_verify?(build_sign_data(options.dup), options['sign'])
    end
  end



  def build_generate_sign options
    sign_data = build_sign_data(options.dup)

    if options["sign_type"] == "MD5"
      Digest::MD5.hexdigest(sign_data + ENV['ALIPAY_MD5_SECRET'])
    elsif options["sign_type"] == "RSA"
      build_rsa_sign(sign_data)
    end
  end

  # RSA 签名
  def build_rsa_sign(data)
    #拿到本地生成的私钥
    private_key_path = Rails.root.to_s + "/config/.alipay_self_private"
    pri = OpenSSL::PKey::RSA.new(File.read(private_key_path))
    signature = Base64.encode64(pri.sign('sha1', data))
    signature
  end

  # RSA 验证
  def build_rsa_verify?(data, sign)
    public_key_path = Rails.root.to_s + "/config/.alipay_public"
    pub = OpenSSL::PKey::RSA.new(File.read(public_key_path))
    digester = OpenSSL::Digest::SHA1.new
    sign = Base64.decode64(sign)
    pub.verify(digester, sign, data)
  end

  def build_sign_data data_hash
    data_hash.delete_if { |k, v| k == "sign_type" || k == "sign" || v.blank? }
    data_hash.to_a.map { |x| x.join('=') }.sort.join('&')
  end
end
```

### payment/index.html.erb
```html
<h1>支付</h1>
<div class="row">
  <div class="container">
    <div class="page-header">
      <h4><i class="fa fa-credit-card"></i> 支付宝付款</h4>
    </div>
    <p>订单号: <%= @payment.orders.map { |order| "<strong>#{order.order_no}</strong>" }.join(', ').html_safe %></p> #在页面上输出要支付的订单号
    <ul class="list-group">
      <li class="list-group-item">
        <%= image_tag "alipay.png", width: 100 %>
      </li>
    </ul>
    <br />
    <div class="pull-right">
      <form action="<%= @payment_url %>" method="post">
        <% @pay_options.each do |k, v| %>
          <input type="hidden" name="<%= k %>" value="<%= v %>" />
        <% end -%>
        <strong>¥<%= @payment.total_money %></strong>
        <input type="submit" value="支付" class="btn btn-success btn-lg" />
      </form>
    </div>
  </div>
</div>
```

### payment.rb
```ruby
 #添加
 # is_success判断当前的付款记录是否是成功的
 def is_success?
   self.status == PaymentStatus::Success
 end

 def do_success_payment! options
   self.transaction do
     self.transaction_no = options[:trade_no]
     self.status = Payment::PaymentStatus::Success
     self.raw_response = options.to_json
     self.payment_at = Time.now
     self.save!

     # 更新订单状态
     self.orders.each do |order|
       if order.is_paid?
         raise "order #{order.order_no} has alreay been paid"
       end
       order.status = Order::OrderStatus::Paid
       order.payment_at = Time.now
       order.save!
     end
   end
 end

 def do_failed_payment! options
   self.transaction_no = options[:trade_no]
   self.status = Payment::PaymentStatus::Failed
   self.raw_response = options.to_json
   self.payment_at = Time.now
   self.save!
 end
```
