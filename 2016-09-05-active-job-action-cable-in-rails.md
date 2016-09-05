

---
title: Rails 应用生产环境利用  Active Job 和 Action Cable 实现消息推送
date: 2016-09-05 23:53
categories: [技术]
tags: [active job, action cable, web socket, passenger, queue, sidekiq]
---

## 设计

基本的设计思路是，在相应的场合创建 message，通过 Active Job 广播到 Action Cable。

### Message

为了在 rails 中实现消息通知系统，简单调研了一些开源项目的实现方式。大致逻辑是，建立一个如下所示的 Message 表：

| id   | title | content | type | recevier | actor | target | path |
| ---- | ----- | ------- | ---- | -------- | ----- | ------ | ---- |
|      | 标题    | 内容      | 消息类型 | 接收者      | 操作者   | 操作的对象  | 跳转地址 |

在需要推送消息的场合，比如创建评论，@他人等，在 Comment model 中添加一个`after_create`动作来创建 message。web socket 的推送只显示 title，消息列表可以显示 title 和 content。点击消息时按照 path 来跳转。

### Action Cable

为了实现 web socket 推送，我们还需要开启 action cable[^1]。关于 action cable，请参见[DHH 的教学视频](http://railscasts-china.com/episodes/action-cable-rails-5)。

开启`cable.js`和`routes`。

```js
// app/assets/javascripts/cable.js
//= require action_cable
//= require_self
//= require_tree ./channels
(function() {
  this.App || (this.App = {});
  App.cable = ActionCable.createConsumer();
}).call(this);
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  mount ActionCable.server => "/cable"
end
```

创建连接。这里连接的时候会写入 current user，供接下来按照 user 订阅用。

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    protected
    def find_verified_user
      # 我使用了 devise，可以这样获取 current user
      if current_user = env['warden'].user
        current_user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

创建 channel：`rails g channel messages`。

```ruby
# app/channels/messages_channel.rb
class MessagesChannel < ApplicationCable::Channel
  def subscribed
    # 从 connection 可以拿到 current user
    stream_from "messages_#{current_user.id}"
  end
  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end
end
```

```coffeescript
# app/assets/javascripts/channels/messages.coffee
jQuery(document).on 'turbolinks:load', ->
  App.messages = App.cable.subscriptions.create "MessagesChannel",
    connected: ->
      console.log 'connected'
    disconnected: ->
      console.log 'disconnected'  
    received: (data) ->
      console.log 'received'
      # 其他操作
```

这样，我们便可以在创建 message 的时候推送了，比如，在 message model 中添加`after_create_commit`动作，触发广播。

```ruby
ActionCable.server.broadcast "messages_#{message.receiver.id}", data
```

然而这里更推荐使用 active job。

### Active Job

首先创建 job，`rails g job message_notification`。

```ruby
class MessageNotificationJob < ApplicationJob
  queue_as :default

  def perform(message)
    ActionCable.server.broadcast "messages_#{message.receiver.id}", data # 这里是 message 组装的 data 
  end
end
```

然后把上面的`after_create_commit`改为：`MessageNotificationJob.perform_later(self)`。

如此重启服务器即可正常工作。

## 生产环境配置

在生产环境下，Active Job 使用 sidekiq，Action Cable 使用 redis。(Ruby on Rails 的生产环境配置参见 [Ruby on Rails 开发和生产环境搭建](/2016/03/ruby-on-rails-on-aliyun/))

### Action Cable

安装 redis[^2]。

```shell
# 源码安装
# wget/tar/configure/make/make install

# yum 安装
rpm -Uvh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
yum --enablerepo=remi,remi-test install redis
# 配置
vi /etc/sysctl.conf # vm.overcommit_memory=1
sysctl vm.overcommit_memory=1
sysctl -w fs.file-max=100000
# 添加 service
chkconfig --add redis
chkconfig --level 345 redis on
service redis start/stop/restart
```

设置 redis 的地址。

```yaml
# config/cable.yml
development:
  adapter: async
 
test:
  adapter: async
 
production:
  adapter: redis
  url: redis://localhost:12345
```

环境设置。

```ruby
# config/environments/production
config.action_cable.allowed_request_origins = ["https://example.com", "http://example.com"]
```

nginx 服务器配置。

```nginx
server {
  listen 80;
  server_name example.com;

  passenger_enabled on;
  passenger_ruby your_ruby_path;
  rails_env production;
  root your_project_path/current/public;

  location /cable {
    passenger_app_group_name your_app_name_action_cable;
    passenger_force_max_concurrent_requests_per_process 0;
  }

  error_page 500 502 503 504 /50x.html;
  location = /50x.html {
    root html;
  }
}
```



### Active Job

Job 适配器使用 [sidekiq](http://sidekiq.org/)，在`Gemfile`中添加并`bundle install`。

```ruby
# config/environments/production.rb
config.active_job.queue_adapter = :sidekiq
```

初始化。

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: 'redis://localhost:12345' }
end

Sidekiq.configure_client do |config|
  config.redis = { url: 'redis://localhost:12345' }
end
```

确保 redis 已经正常启动，然后在项目目录下执行`sidekiq`，此时便可以正常工作了。

如果你的 job 的 queue 不是  default，那么你还需要创建`config/sidekiq.yml`文件。执行的时候需要`sidekiq -C ./config/sidekiq.yml`。

```yaml
---
:concurrency: 1
production:
  :concurrency: 25
:queues:
  - [task1, 5]
  - [task2, 1]
```

使用 mina 部署的时候，需要添加`mina_sidekiq`这个 gem[^3]。

```ruby
require 'mina_sidekiq/tasks'
# 在需要的地方 invoke
invoke :'sidekiq:quiet'
invoke :'sidekiq:restart'
```

[^1]: 参见样例 [Building a chess server in Rails 5 with Action Cable-powered WebSockets](http://jargon.io/joeyschoblaska/rails-5-chess-with-action-cable-websockets) 和官方文档 [Action Cable Overview — Ruby on Rails Guides](http://edgeguides.rubyonrails.org/action_cable_overview.html)
[^2]: 安装[gist](https://gist.github.com/nghuuphuoc/7801123)，[配置](https://gorails.com/episodes/deploy-actioncable-and-rails-5)
[^3]: [gem](https://github.com/Mic92/mina-sidekiq)，[示例](https://ruby-china.org/topics/26661)