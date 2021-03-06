wechat openflatform 3rd-party service
====================

微信开放平台第三方开放平台版SDK－主动调用接口


## 功能说明

微信开放平台第三方开放平台接口主动调用，以及提供应用授权接口。

## 安装方法

```sh
$ npm install wechat-open-service
```

## 使用方法

- 回调接口请移步: [微信公共平台企业号版(第三方企业套件)SDK－回调接口](https://github.com/node-webot/wechat-corp-service-callback)
- 对, 你没看错, 这是可以用的, 修改一下InfoType的判断就行

```js
var _route = function(message, req, res, next) {

      if (message.InfoType == 'component_verify_ticket') { //微信服务器发过来的票，每10分钟发一次
        ParamConfigService.set('component_verify_ticket',message.ComponentVerifyTicket);
        sails.log.error("ticket:"+message.ComponentVerifyTicket);
        res.reply('success');
      }else if (message.InfoType == 'unauthorized') { //取消授权的通知
        //更新到数据库
        sails.log.error("取消授权的appid:"+message.AuthorizerAppid);
        res.reply('success');
      } else {
        res.reply('success');
      };
    }

    wechat_cs(_config, _route)(req, res, next);
```


### 前提

- 首先，你要有一个开放平台号(open.weixin.qq.com)。
- 然后，你要通过开发者认证以及申请第三方开放平台。
- 接下来才可以创建第三方平台应用，并且设置第三方平台应用。
- 基于本SDK开发具体的第三方平台应用。

### 用法

其中的token，encodingAESKey，suite_id可以在套件的信息配置界面获取。

```js
var WechatOpenAPI = require('wechat-open-service');

var get_token = function(cb) {
  var self = this;
  SuiteConfig
    .findOne({
      suite_id: self.suiteId,
    })
    .exec(function(err, result) {
      if (result && (new Date().getTime()) < (new Date(result.suite_access_token_expire)).getTime()) {
        // 有效期内，直接返回
        cb(null, {
          suite_access_token: result.suite_access_token,
          expires_in: result.suite_access_token_expire
        });
      } else {
        cb(null, null);
      }
    });
}

var save_token = function(token, cb) {
  var self = this;
  async.waterfall([
    function(cb) {
      SuiteConfig
        .findOne({
          suite_id: self.suiteId,
        }).exec(cb);
    },
    function(sc, cb) {
      if (sc) {
        //已经存在了，就更新一下token，否则创建一条新的
        sc.suite_access_token = token.suite_access_token;
        sc.suite_access_token_expire = new Date((new Date()).getTime() + 7190000);
        sc.save(cb);
      } else {
        cb(null, null);
      }
    }
  ], cb);
}

var openApi = new WechatOpenAPI(component_appid,component_appsecret,ticket,get_token,save_token);
// 授权完成后跳转的URL，一般返回套件开发商自己的页面，并且获取用户授权的信息。
redirect_uri = 'http://xxx.xxx.xxx/auth_callback_url',
auth_url = '';

// 获取临时授权码，生成授权页面（带一个授权的按钮）
openApi.getPreAuthCode(appid, function(err, result) {
  auth_url = openApi.generateAuthUrl(result.pre_auth_code, encodeURIComponent(redirect_uri));
});

// 授权后，跳转回来的URL，可以获取auth_code，然后换取authorizer_refresh_token。得到永久授权之后就能知道是哪个微信公众号appid了(authorizer_appid)。
var auth_code = req.query.auth_code;
openApi.getAuthorizerRefreshToken(auth_code, cb);

```

### 代替授权公众号调用接口用法, 基于前面的方法获得authorizer_refresh_token后, 可以这样调用

```js

openApi.getLatestToken(function(err, token){
	var component_access_token = token.component_access_token;
	var apiAgent = new WechatOpenAPI.Agent(component_appid,component_access_token,authorizer_appid,authorizer_refresh_token,get_token,save_token);
		//举例,获取客户公众号的素材数量
	apiAgent.getMaterialCount(function(err, result){...});	
});

```

### 通过代理服务器访问

#### 场景

对于大规模的集群部署模式，为了安全和速度，会有一些负载均衡的节点放在内网的服务器上（即负载均衡的节点与主结点通过内网连接，并且内网服务器上没有外网的IP）。这时，就需要配置代理服务器来使内网的机器可以有限度的访问外网的资源。例如：微信套件中的各种主动调用接口。

如何架设代理服务器在这里不做赘述，一般推荐使用squid 3，免费、快速、配置简单。

#### 技术原理

由于需要访问的微信API服务器是https协议，所以普通的http代理模式不能使用。
而一般都是http协议的代理服务器。
我们要实现的就是通过http代理通道来走https的请求。

基本的步骤是2步：

- 连接到代理服务器，发送CONNECT命令，打开一个TCP连接。
- 使用上一步打开的TCP连接，发送https的请求。

#### 实现步骤

一、下载[node-tunnel](https://github.com/koichik/node-tunnel) 注意：npm上的版本较老，不支持node v0.10以上的版本。

二、使用 httpsOverHttp 这个agent。

三、将agent配置给urllib，通过urllib的beforeRequest这个方法。

```js
var tunnel = require('tunnel');
var APICorp = require('wechat-corp-service');

var agent = tunnel.httpsOverHttp({
  proxy: {
    host: 'proxy_host_ip',
    port: 3128
  }
});

var apicorp = new APICorp(sc.suite_id, sc.suite_secert, sc.suite_ticket, get_token, save_token);

apicorp.setOpts({
    beforeRequest:function(options){
        options.agent = agent;
    }
});

```

## 相关文档
- [微信企业号－第三方应用授权](http://qydev.weixin.qq.com/wiki/index.php?title=%E7%AC%AC%E4%B8%89%E6%96%B9%E5%BA%94%E7%94%A8%E6%8E%88%E6%9D%83)


## License
The MIT license.

## 交流群
QQ群：157964097，使用疑问，开发，贡献代码请加群。

## 感谢
感谢以下贡献者：
```
 project  : wechat-corp-service
 repo age : 10 months
 active   : 7 days
 commits  : 16
 files    : 13
 authors  :
     6  Jackson Tian  37.5%
     6  Nick Ma       37.5%
     3  hezedu        18.8%
     1  马剑          6.2%

```

## 捐赠
如果您觉得Wechat企业号版本对您有帮助，欢迎请作者一杯咖啡

![捐赠wechat](https://cloud.githubusercontent.com/assets/327019/2941591/2b9e5e58-d9a7-11e3-9e80-c25aba0a48a1.png)
