本文以使用 Node.js 开发一个简单常见的客服场景 Demo 为例，介绍微信订阅号集成腾讯云即时通信 IM 的基本流程。
>示例仅供参考，正式上线前需要进一步完善，例如服务器负载均衡、接口并发控制、信息持久化存储等。此类优化操作不在本文介绍范围内，请开发者根据实际情况自行实现。

## 场景流程及效果图
本文 Demo 场景的基本流程如下：
1. 客户通过某服装电商订阅号询问“童装啥时候上新？”。
2. 客户的咨询消息经过腾讯云 IM 系统传输至此服装电商的坐席客服。
3. 客服人员回复“5月份会上新，敬请关注！”，消息经过腾讯云 IM 系统和微信传输推送给客户。



## 注意事项
- 消息传输链路较长，可能会影响消息收发耗时。
- 个人注册的订阅号，不能使用微信公众平台的【客服消息】接口向订阅者主动推送消息。

## 前提条件

- 准备一台可以运行 Node.js 的公网开发服务器或云服务器。
- [注册](https://mp.weixin.qq.com/cgi-bin/registermidpage?action=index&lang=zh_CN&token=) 微信订阅号或服务号。
- 详细阅读 [微信公众平台开发文档](https://developers.weixin.qq.com/doc/offiaccount/Basic_Information/Access_Overview.html)。
- 已 [创建即时通信 IM 应用](https://intl.cloud.tencent.com/document/product/1047/34577)。
- 需提前 [逐个导入](https://intl.cloud.tencent.com/document/product/1047/34953) 或 [批量导入](https://intl.cloud.tencent.com/document/product/1047/34954) 即时通信 IM 用户帐号，例如 user0 和 user1。

## 参考文档

- [API 文档](https://intl.cloud.tencent.com/document/product/1047/34620)
- [第三方回调](https://intl.cloud.tencent.com/document/product/1047/34354)
- [微信公众平台开发指南](https://developers.weixin.qq.com/doc/offiaccount/Getting_Started/Overview.html)
- [Express 框架教程](https://expressjs.com/zh-cn/)

## 操作步骤

### 步骤1：创建开发项目并安装依赖

```javascript
npm init -y

// express 框架
npm i express@latest --save 

// 加密模块
npm i crypto@latest --save

// 解析 xml 的工具
npm i xml2js@latest --save

// 发起 http 请求
npm i axios@latest --save

// 计算 userSig
npm i tls-sig-api-v2@latest --save
```

### 步骤2：填入 IM 应用信息并计算 UserSig

<pre>
// ------------ IM ------------
const IMAxios = axios.create({
  timeout: 10000,
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8',
  },
});

// 已导入 IM 帐号系统的用户 ID 映射表，非持久化存储，作 Demo 快速检索用，生产环境请用别的技术方案
const importedAccountMap = new Map();
// IM 应用及 App 管理员信息，请登录 <a href="https://console.cloud.tencent.com/im">即时通信 IM 控制台</a> 获取
const SDKAppID = 0; // 填入 IM 应用的 SDKAppID
const secrectKey = ''; // 填入 IM 应用的密钥
const AppAdmin = 'user0'; // 设置 user0 为 App 管理员帐号
const kfAccount1 = 'user1'; // 设置 user1 为一个坐席客服帐号
// 计算 UserSig，调用 REST API 时需要用到，详细操作请参考 <a href="https://github.com/tencentyun/tls-sig-api-v2-node">Github</a>
const api = new TLSSigAPIv2.Api(SDKAppID, secrectKey);
const userSig = api.genSig(AppAdmin, 86400*180);
console.log('userSig:', userSig);
</pre>

### 步骤3：配置 URL 和 Token
>此指引文档是直接参考微信公众平台开发指南所写，若有变动，请以 [接入指南](https://mp.weixin.qq.com/wiki) 为准。

1. 登录订阅号管理后台。
2. 选择【基本配置】，勾选协议成为开发者。
3. 单击【修改配置】，填写相关信息：
 - URL：服务器地址，用作接收微信消息和事件的接口 URL，必填参数。
 - Token：可任意填写，用作生成签名，该 Token 会和接口 URL 中包含的 Token 进行比对，从而验证安全性，必填参数。
 - EncodingAESKey：手动填写或随机生成，用作消息体加解密密钥，选填参数。

### 步骤4：启动 Web 服务监听端口，并正确响应微信发送的 Token 验证

<pre>
const express = require('express'); // express 框架 
const crypto =  require('crypto'); // 加密模块
const util = require('util');
const xml2js = require('xml2js'); // 解析 xml
const axios = require('axios'); // 发起 http 请求
const TLSSigAPIv2 = require('tls-sig-api-v2'); // 计算 userSig

// ------------ Web 服务 ------------
var app = express();
// Token 需在【订阅号管理后台】>【基本配置】设置

// 处理所有进入80端口的 get 请求
app.get('/', function(req, res) {
  // ------------ 接入微信公众平台 ------------
  // 详细请参考 <a href="https://developers.weixin.qq.com/doc/offiaccount/Basic_Information/Access_Overvie">微信官方文档</a> 
  // 获取微信服务器 Get 请求的参数 signature、timestamp、nonce、echostr
  var signature = req.query.signature; // 微信加密签名
  var timestamp = req.query.timestamp; // 时间戳
  var nonce = req.query.nonce; // 随机数
  var echostr = req.query.echostr; // 随机字符串

  // 将 token、timestamp、nonce 三个参数进行字典序排序
  var array = [myToken, timestamp, nonce];
  array.sort();

  // 将三个参数字符串拼接成一个字符串进行 sha1 加密
  var tempStr = array.join('');
  const hashCode = crypto.createHash('sha1'); // 创建加密类型 
  var resultCode = hashCode.update(tempStr,'utf8').digest('hex'); // 对传入的字符串进行加密

  // 开发者获得加密后的字符串可与 signature 对比，标识该请求来源于微信
  if (resultCode === signature) {
    res.send(echostr);
  } else {
    res.send('404 not found');
  }
});

// 监听80端口
app.listen(80);
</pre>

### 步骤5：实现开发者服务器侧业务逻辑

- 收到微信推送的关注事件时，调用 [导入单个帐号](https://intl.cloud.tencent.com/document/product/1047/34953) 或 [导入多个帐号](https://intl.cloud.tencent.com/document/product/1047/34954) API 向帐号系统导入帐号。
- 收到微信推送的关注事件时，被动回复消息。
- 收到微信推送的取消关注事件时，调用 [删除帐号](https://intl.cloud.tencent.com/document/product/1047/34955) API 将该帐号从帐号系统删除。
- 收到微信推送的普通消息时，调用 [单发单聊消息](https://intl.cloud.tencent.com/document/product/1047/34919) API 向客服帐号发单聊消息。

```javascript
const genRandom = function() {
  return  Math.floor(Math.random() * 10000000);
}

// 生成 wx 文本回复的 xml
const genWxTextReplyXML = function(to, from, content) {
  let xmlContent = '<xml><ToUserName><![CDATA[' + to + ']]></ToUserName>'
  xmlContent += '<FromUserName><![CDATA[' + from + ']]></FromUserName>'
  xmlContent += '<CreateTime>' + new Date().getTime() + '</CreateTime>'
  xmlContent += '<MsgType><![CDATA[text]]></MsgType>'
  xmlContent += '<Content><![CDATA[' + content + ']]></Content></xml>';

  return xmlContent;
}

/**
 * 向 IM 帐号系统导入用户
 * @param {String} userID 要导入的用户 ID
 */
const importAccount = function(userID) {
  console.log('importAccount:', userID);
  return new Promise(function(resolve, reject) {
    var url = util.format('https://console.tim.qq.com/v4/im_open_login_svc/account_import?sdkappid=%s&identifier=%s&usersig=%s&random=%s&contenttype=json',
      SDKAppID, AppAdmin, userSig, genRandom());
    console.log('importAccount url:', url);
    IMAxios({
      url: url,
      data: {
        "Identifier": userID
      },
      method: 'POST'
    }).then((res) => {
      if (res.data.ErrorCode === 0) {
        console.log('importAccount ok.', res.data);
        resolve();
      } else {
        reject(res.data);
      }
    }).catch((error) => {
      console.log('importAccount failed.', error);
      reject(error);
    })
  });
}

/**
 * 从 IM 帐号系统删除用户
 * @param {String} userID 要删除的用户 ID
 */
const deleteAccount = function(userID) {
  console.log('deleteAccount', userID);
  return new Promise(function(resolve, reject) {
    var url = util.format('https://console.tim.qq.com/v4/im_open_login_svc/account_delete?sdkappid=%s&identifier=%s&usersig=%s&random=%s&contenttype=json',
      SDKAppID, AppAdmin, userSig, genRandom());
    console.log('deleteAccount url:', url);
    IMAxios({
      url: url,
      data: {
        "DeleteItem": [
          {
            "UserID": userID,
          },
        ]
      },
      method: 'POST'
    }).then((res) => {
      if (res.data.ErrorCode === 0) {
        console.log('deleteAccount ok.', res.data);
        resolve();
      } else {
        reject(res.data);
      }
    }).catch((error) => {
      console.log('deleteAccount failed.', error);
      reject(error);
    })
  });
}

/**
 * 单发单聊消息
 */
const sendC2CTextMessage = function(userID, content) {
  console.log('sendC2CTextMessage:', userID, content);
  return new Promise(function(resolve, reject) {
    var url = util.format('https://console.tim.qq.com/v4/openim/sendmsg?sdkappid=%s&identifier=%s&usersig=%s&random=%s&contenttype=json',
      SDKAppID, AppAdmin, userSig, genRandom());
    console.log('sendC2CTextMessage url:', url);
    IMAxios({
      url: url,
      data: {
        "SyncOtherMachine": 2, // 消息不同步至发送方。若希望将消息同步至 From_Account，则 SyncOtherMachine 填写1。
        "To_Account": userID,
        "MsgLifeTime":60, // 消息保存60秒
        "MsgRandom": 1287657,
        "MsgTimeStamp": Math.floor(Date.now() / 1000), // 单位为秒，且必须是整数
        "MsgBody": [
          {
            "MsgType": "TIMTextElem",
            "MsgContent": {
              "Text": content
            }
          }
        ]
      },
      method: 'POST'
    }).then((res) => {
      if (res.data.ErrorCode === 0) {
        console.log('sendC2CTextMessage ok.', res.data);
        resolve();
      } else {
        reject(res.data);
      }
    }).catch((error) => {
      console.log('sendC2CTextMessage failed.', error);
      reject(error);
    });
  });
}

// 处理微信的 post 请求
app.post('/', function(req, res) {
  var buffer = [];
  // 监听 data 事件，用于接收数据
  req.on('data', function(data) {
    buffer.push(data);
  });
  // 监听 end 事件，用于处理接收完成的数据
  req.on('end', function() {
    const tmpStr = Buffer.concat(buffer).toString('utf-8');
    xml2js.parseString(tmpStr, { explicitArray: false }, function(err, result) {
      if (err) {
        console.log(err);
        res.send("success");
      } else {
        if (!result) {
          res.send("success");
          return;
        }
        console.log('wx post data:', result.xml);
        var wxXMLData = result.xml;
        var toUser = wxXMLData.ToUserName; // 接收方微信
        var fromUser = wxXMLData.FromUserName;// 发送仿微信
        if (wxXMLData.Event) {  // 处理事件类型
          switch (wxXMLData.Event) {
            case "subscribe": // 关注订阅号
              res.send(genWxTextReplyXML(fromUser, toUser, '欢迎关注，XX竭诚为您服务！'));
              importAccount(fromUser).then(() => {
                // 记录已导入用户的 ID
                importedAccountMap.set(fromUser, 1);
              });
              break;
            case "unsubscribe": // 取消关注
              deleteAccount(fromUser).then(() => {
                importedAccountMap.delete(fromUser);
              });
              res.send("success");
              break;
          }
        } else { // 处理消息类型
          switch (wxXMLData.MsgType) {
            case "text":
              // 处理文本消息
              sendC2CTextMessage(kfAccount1, '来自微信订阅号的咨询：' + wxXMLData.Content).then(() => {
                console.log('发送C2C消息成功');
              }).catch((error) => {
                console.log('发送C2C消息失败');
              });
              break;
            case "image":
              // 处理图片消息
              break;
            case "voice":
              // 处理语音消息
              break;
            case "video":
              // 处理视频消息
              break;
            case "shortvideo":
              // 处理小视频消息
              break;
            case "location":
              // 处理发送地理位置
              break;
            case "link":
              // 处理点击链接消息
              break;
            default:
              break;  
          }
          res.send(genWxTextReplyXML(fromUser, toUser, '正在为您转接人工客服，请稍等'));
        }
      }
    })
  });
});
```

### 步骤6：注册并处理 IM 第三方回调

<pre>
// 处理 IM 第三方回调的 post 请求
app.post('/imcallback', function(req, res) {
  var buffer = [];
  // 监听 data 事件 用于接收数据
  req.on('data', function(data) {
    buffer.push(data);
  });
  // 监听 end 事件 用于处理接收完成的数据
  req.on('end', function() {
    const tmpStr = Buffer.concat(buffer).toString('utf-8');
    console.log('imcallback', tmpStr);
    const imData = JSON.parse(tmpStr);
    // kfAccount1 发的消息推送给客户
    if (imData.From_Account === kfAccount1) {
      // 组包消息，并通过微信的【客服消息】接口，向指定的用户推送消息
      // 注意！个人注册的订阅号不支持使用此接口，详情请参见 <a href="https://developers.weixin.qq.com/doc/offiaccount/Message_Management/Service_Center_messages.html">客服消息</a> 
    }

    res.send({
      "ActionStatus": "OK",
      "ErrorInfo": "",
      "ErrorCode": 0 // 0表示允许发言，1表示拒绝发言
    });
  });
});
</pre>
