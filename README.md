# 关于安全性的编程建议

程序员是建筑工人, 黑客是爆破工人.

我一直认可 20-80 定律. 我认为, 只要一个程序员在安全性投入不多的时间, 就可以让自己的应用增加80%的安全性.

经过一些日子的接触, 我发现有各种情况都可以拿到某个服务器的root权限. 所以, 作为建筑工人,我们务必慎之又慎.

## SQL语句务必使用 prepared statement

只要使用了SQL,就务必记得使用prepared statement.

很多框架(例如 Rails, Hibernate) 都提供了内置的解决方案. 例如:

```
# ruby, 获得 age = 23的 User

User.where("age = ?", 23)
```

错误写法:

```
User.where("age = #{params[:age]}")
```

在错误写法中, 只要黑客在浏览器中输入 "http...?age=23 or 1=1"  (转义形式), 就可以让数据库把相关的数据都查出来



## 在生产环境中, 不要展示 错误的stack trace

正确的做法:

任何出错页面, 都给出提示:

```
服务器开小差了! 请稍后再试~
```

错误的例子:

```
Missing template comments/update, application/update with {:locale=>[:en], :formats=>[:html], :variants=>[], :handlers=>[:erb, :builder, :raw, :ruby]}. Searched in: * "/workspace/wooyun_rails/app/views" * "/home/siwei/.rbenv/versions/2.5.0/lib/ruby/gems/2.5.0/gems/devise-4.4.3/app/views" * "/home/siwei/.rbenv/versions/2.5.0/lib/ruby/gems/2.5.0/gems/kaminari-core-1.1.1/app/views"
Extracted source (around line #46):
    def find(*args)
      find_all(*args).first || raise(MissingTemplate.new(self, *args))
    end

    def find_file(path, prefixes = [], *args)

Rails.root: /workspace/wooyun_rails

Application Trace | Framework Trace | Full Trace
actionview (4.2.10) lib/action_view/path_set.rb:46:in `find'
actionview (4.2.10) lib/action_view/lookup_context.rb:121:in `find'
actionview (4.2.10) lib/action_view/renderer/abstract_renderer.rb:18:in `find_template'
actionview (4.2.10) lib/action_view/renderer/template_renderer.rb:40:in `determine_template'
actionview (4.2.10) lib/action_view/renderer/template_renderer.rb:8:in `render'
actionview (4.2.10) lib/action_view/renderer/renderer.rb:46:in `render_template'
actionview (4.2.10) lib/action_view/renderer/renderer.rb:27:in `render'
actionview (4.2.10) lib/action_view/rendering.rb:100:in `_render_template'
actionpack (4.2.10) lib/action_controller/metal/streaming.rb:217:in `_render_template'
actionview (4.2.10) lib/action_view/rendering.rb:83:in `render_to_body'
```

在上面的例子中, 黑客可以通过 数据库注入 + 报错页面,得知你的系统是否有漏洞.

只要保证在生产环境中不展示错误的详细信息, 就算你的程序中有漏洞, 黑客也基本不知道漏洞在哪里.

## 关于密码

### 绝对不要使用弱密码

下面是中国人喜欢用的:

```
123456
666666
88888888
iloveyou
love123
```

### 绝对不要在重要的账户中使用旧密码

你在一些国内知名网站的密码早都泄露了. 在2010年之前, 貌似国内网站都喜欢保存明文密码.

所以, 务必使用一个新密码.

### 重要系统中, 使用google验证器

例如, 在后台管理员端, 除了 "用户名", "密码" 之外,还应该有一个 google验证器, 它的作用跟短信验证码是一样的.

## API 接口,只给前端需要的数据.

多余的数据绝对不要有.

例如, 前端仅需要展示用户的名称列表时(例如某个排行榜), 这样的json是合理的:

```
[
  "张三",
  "李四",
  "韩梅梅",
  "隔壁老王"
]

```

下面是一个错误的例子:
```
[
  {
    name: "张三",
    id_card: "220110198908081234",
    email: "zhangsan@email.com",
    mobile: "13800138000",
    id: 32842
  }
]

```
在上面的例子中,除了 name 这个列是有用的,其他的都是没用的. 可能后端程序员为了偷懒, 直接 `model.to_json` 就把数据展示出来了,
实际上这给黑客提供了很多的猜测空间. 黑客完全可以通过 id: 32842 知道, 原来数据库的用户已经到了3万多. 也可以通过email/mobile
得知该用户的邮箱和手机,进一步人肉他. 身份证那个就更不用说了.

## 不要让黑客通过id来猜

例如：

黑客看到的一个页面：

```
http://yoursite.com/page?id=1000
```

那么黑客最喜欢的事儿就是访问：

```
http://yoursite.com/page?id=1001
http://yoursite.com/page?id=1002
http://yoursite.com/page?id=1003
```

如果上面有一个是包含了敏感内容， 那么黑客就可以随便一个轮询，获得这个数据库的数据

## 服务器端的注意事项

### 绝对不要 为了偷懒把正式库的数据放到测试库上

这样做的好处是测试环境很带感.

坏处则是:

任何员工(从资深员工到实习生)都可以获得生产环境数据, 造成泄露

正确做法:

1. 测试环境,应该使用测试数据.
2. 如果一定要加入生产数据,务必把email, mobile隐藏部分字符串
3. 如果要进行压力测试的话,可以用程序随机生成百万千万记录.

### 数据库等敏感服务器, 要设置ip白名单.

绝对不要允许任何人都可以访问数据库

别有用心的黑客可以通过密码撞库来获得访问权限

原则上 MySQL, Redis, 数字币节点服务器,只允许局域网内的ip来访问

### SSH端口不要使用默认的22

你不知道黑客看到可用的端口的时候有多兴奋

### SSH用户不要使用默认的root

你不知道黑客看到可用的root的时候有多兴奋

## 小心任何上传的地方

例如: 上传头像, 上传身份证, 上传图片等.

黑客可以通过低版本的imagemagick 来获得root权限.

黑客可以用过上传一些 .php 等脚本文件来获得root权限

所以, 这里要小心, 注意阿里云等云服务器厂商给出的安全提示. 他们的人专门是做这个的,非常专业.

## 绝对不要把公司的代码上传到github上。

公司的代码很可能包含了：

服务器的ip, port, username, password等信息。

好的办法：

1. repo 不够了赶紧让上面增加新的
2. 任何需要下载后修改的配置文件，都要使用.example。 例如：

坏的例子：

```
config/database.yml
```

好的例子：

```
config/database.yml.example
```
