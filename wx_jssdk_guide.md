# 使用jssdk方法

接口文档

http://mp.weixin.qq.com/wiki/7/aaa137b55fb2e0456bf8dd9148dd613f.html

## 1、 先去拿accessToken

接口文档

 [http://mp.weixin.qq.com/wiki/15/54ce45d8d30b6bf6758f68d2e95bc627.html](http://mp.weixin.qq.com/wiki/15/54ce45d8d30b6bf6758f68d2e95bc627.html)

> access_token是公众号的全局唯一票据
>
> access_token的存储至少要保留512个字符空间。
>
> access_token的有效期目前为2个小时，需**定时**刷新，重复获取将导致上次获取的access_token失效。



```
1、为了保密appsecrect，第三方需要一个access_token获取和刷新的中控服务器。而其他业务逻辑服务器所使用的access_token均来自于该中控服务器，不应该各自去刷新，否则会造成access_token覆盖而影响业务；
2、目前access_token的有效期通过返回的expire_in来传达，目前是7200秒之内的值。中控服务器需要根据这个有效时间提前去刷新新access_token。在刷新过程中，中控服务器对外输出的依然是老access_token，此时公众平台后台会保证在刷新短时间内，新老access_token都可用，这保证了第三方业务的平滑过渡；
3、access_token的有效时间可能会在未来有调整，所以中控服务器不仅需要内部定时主动刷新，还需要提供被动刷新access_token的接口，这样便于业务服务器在API调用获知access_token已超时的情况下，可以触发access_token的刷新流程。
```

- 使用AppID和AppSecret调用本接口来获取access_token
- AppID和AppSecret可在微信公众平台官网-开发者中心页中获得

```
http请求方式: GET
https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=APPID&secret=APPSECRET
```

**参数说明**

| 参数         | 是否必须 | 说明                                |
| ---------- | ---- | --------------------------------- |
| grant_type | 是    | 获取access_token填写client_credential |
| appid      | 是    | 第三方用户唯一凭证                         |
| secret     | 是    | 第三方用户唯一凭证密钥，即appsecret            |

**返回说明**

正常情况下，微信会返回下述JSON数据包给公众号：

```
{"access_token":"ACCESS_TOKEN","expires_in":7200}

```

| 参数           | 说明          |
| ------------ | ----------- |
| access_token | 获取到的凭证      |
| expires_in   | 凭证有效时间，单位：秒 |

错误时微信会返回错误码等信息，JSON数据包示例如下（该示例为AppID无效错误）:

```
{"errcode":40013,"errmsg":"invalid appid"}
```



##  2、获取jsapi_ticket方法

根据获取的accessTocken（有效期7200秒，开发者必须在自己的服务全局缓存jsapi_ticket）

```
http请求方式：GET
https://api.weixin.qq.com/cgi-bin/ticket/getticket?access_token=ACCESS_TOKEN&type=jsapi
```

成功返回如下JSON：

```
{
"errcode":0,
"errmsg":"ok",
"ticket":"bxLdikRXVbTPdHSM05e5u5sUoXNKd8-41ZO3MhKoyN5OfkWITDGgnr2fwJ0m9E8NYzWKVZvdVtaUgWvsdshFKA",
"expires_in":7200
}
```

获得jsapi_ticket之后，就可以生成JS-SDK权限验证的签名了。



## 3、生成JS-SDK签名

> 签名生成规则如下：
>
> 参与签名的字段包括noncestr（16位随机字符串）,
>
> 有效的jsapi_ticket, timestamp（时间戳）,
>
> url（当前网页的URL，不包含#及其后面部分） 。
>
> 对所有待签名参数按照字段名的ASCII 码从小到大排序（字典序）后，使用URL键值对的格式（即key1=value1&key2=value2…）拼接成字符串string1。这里需要注意的是所有参数名均为小写字符。对string1作**sha1**加密，字段名和字段值都采用原始值，**不进行**URL 转义。

即signature=sha1(string1)。 示例：

- noncestr=Wm3WZYTPz0wzccnW
- jsapi_ticket=sM4AOVdWfPE4DxkXGEs8VMCPGGVi4C3VM0P37wVUCFvkVAy_90u5h9nbSlYy3-Sl-HhTdfl2fzFy1AOcHKP7qg
- timestamp=1414587457
- url=[http://mp.weixin.qq.com?params=value](http://mp.weixin.qq.com/?params=value)

步骤1. 对所有待签名参数按照字段名的ASCII 码从小到大排序（字典序）后，使用URL键值对的格式（即key1=value1&key2=value2…）拼接成字符串string1：

```
jsapi_ticket=sM4AOVdWfPE4DxkXGEs8VMCPGGVi4C3VM0P37wVUCFvkVAy_90u5h9nbSlYy3-Sl-HhTdfl2fzFy1AOcHKP7qg&noncestr=Wm3WZYTPz0wzccnW&timestamp=1414587457&url=http://mp.weixin.qq.com?params=value

```

步骤2. 对string1进行sha1签名，得到signature：

```
0f9de62fce790f9a083d5c99e95740ceb90c27ed

```

注意事项

1. 签名用的noncestr和timestamp必须与wx.config中的nonceStr和timestamp相同。
2. 签名用的url必须是调用JS接口页面的完整URL。
3. 出于安全考虑，**开发者必须在服务器端实现签名的逻辑**



***以上全部在服务器生成***

-----

***以下全部由前端处理***



## 1、引入JS文件

> 在需要调用JS接口的页面引入如下JS文件，（支持https）：[http://res.wx.qq.com/open/js/jweixin-1.0.0.js](http://res.wx.qq.com/open/js/jweixin-1.0.0.js)
>
> 请注意，如果你的页面启用了https，务必引入 [https://res.wx.qq.com/open/js/jweixin-1.0.0.js](https://res.wx.qq.com/open/js/jweixin-1.0.0.js) ，否则将无法在iOS9.0以上系统中成功使用JSSDK
>
> 如需使用摇一摇周边功能，请引入 jweixin-1.1.0.js

备注：支持使用 AMD/CMD 标准模块加载方法加载



## 2、通过config接口注入权限验证配置

```
wx.config({
    debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
    appId: '', // 必填，公众号的唯一标识
    timestamp: , // 必填，生成签名的时间戳
    nonceStr: '', // 必填，生成签名的随机串
    signature: '',// 必填，签名，见附录1
    jsApiList: [] // 必填，需要使用的JS接口列表，所有JS接口列表见附录2
});
```

以上信息除了jsApiList，其他都通过后台获取。



## 3、通过ready处理成功验证

```
wx.ready(function(){

    // config信息验证后会执行ready方法，所有接口调用都必须在config接口获得结果之后，config是一个客户端的异步操作，所以如果需要在页面加载时就调用相关接口，则须把相关接口放在ready函数中调用来确保正确执行。对于用户触发时才调用的接口，则可以直接调用，不需要放在ready函数中。
});
```



## 4、分享功能

### **获取“分享到朋友圈”按钮点击状态及自定义分享内容接口**

```
wx.onMenuShareTimeline({
    title: '', // 分享标题
    link: '', // 分享链接
    imgUrl: '', // 分享图标
    success: function () {
        // 用户确认分享后执行的回调函数
    },
    cancel: function () {
        // 用户取消分享后执行的回调函数
    }
});

```

### **获取“分享给朋友”按钮点击状态及自定义分享内容接口**

```
wx.onMenuShareAppMessage({
    title: '', // 分享标题
    desc: '', // 分享描述
    link: '', // 分享链接
    imgUrl: '', // 分享图标
    type: '', // 分享类型,music、video或link，不填默认为link
    dataUrl: '', // 如果type是music或video，则要提供数据链接，默认为空
    success: function () {
        // 用户确认分享后执行的回调函数
    },
    cancel: function () {
        // 用户取消分享后执行的回调函数
    }
});

```

### **获取“分享到QQ”按钮点击状态及自定义分享内容接口**

```
wx.onMenuShareQQ({
    title: '', // 分享标题
    desc: '', // 分享描述
    link: '', // 分享链接
    imgUrl: '', // 分享图标
    success: function () {
       // 用户确认分享后执行的回调函数
    },
    cancel: function () {
       // 用户取消分享后执行的回调函数
    }
});

```

### **获取“分享到腾讯微博”按钮点击状态及自定义分享内容接口**

```
wx.onMenuShareWeibo({
    title: '', // 分享标题
    desc: '', // 分享描述
    link: '', // 分享链接
    imgUrl: '', // 分享图标
    success: function () {
       // 用户确认分享后执行的回调函数
    },
    cancel: function () {
        // 用户取消分享后执行的回调函数
    }
});

```

### **获取“分享到QQ空间”按钮点击状态及自定义分享内容接口**

```
wx.onMenuShareQZone({
    title: '', // 分享标题
    desc: '', // 分享描述
    link: '', // 分享链接
    imgUrl: '', // 分享图标
    success: function () {
       // 用户确认分享后执行的回调函数
    },
    cancel: function () {
        // 用户取消分享后执行的回调函数
    }
});
```



文档信息截止2016年12月07日15:00:38

-----
