
**weixin_rebot** 用Python实现的一个web微信机器人。

## 依赖
```
1. linux: 
   1). ubuntu 12.04
   2). sudo apt-get install imagemagick
2 python:
   1). version > 2.6
   2). pip install requests
   3). pip install pypng
   4). pip install Pillow
   5). pip install pyopenssl ndg-httpsclient pyasn1
```

## web微信登录过程
```
1. 获取uuid
   1). 说明: 微信Web版本不使用用户名和密码登录，而是采用二维码登录，所以服务器需要首先分配一个唯一的会话ID，用来标识当前的一次登录
   2). api: https://login.wx.qq.com/jslogin?appid=wx782c26e4c19acffb&redirect_uri=https%3A%2F%2Fwx.qq.com%2Fcgi-bin%2Fmmwebwx-bin%2Fwebwxnewloginpage&fun=new&lang=zh_CN&_=%s
   3). get 请求
   3). 参数说明:
        a. appid 固定值wx782c26e4c19acffb表示微信网页版
        b. redirect_uri 固定值
        c. fun 固定值 new
        d. lang 固定值 表示中文
        e. _ 表示13位时间戳

2. 获取登录二维码
   1). 说明: get请求拿到数据，再保存为图片并展示
   2). api: https://login.weixin.qq.com/qrcode/%s
   3). get 请求
   4). 参数说明:
       a. 唯一参数是uuid

3. 扫描二维码等待用户确认
   1). 说明: 当用户拿到二维码数据之后，用户需要扫描二维码并点击确认登录
   2). api: https://login.weixin.qq.com/cgi-bin/mmwebwx-bin/login?uuid=%s&tip=1&_=%s
   3). get 请求
   4). 参数说明
       a. uuid 表示当前uuid
       b. tip
          1->表示的是未扫描，等待用户扫描。返回window.code=201表示扫描成功，返回window.code=408表示扫描超时
          0->表示等待用户确认登录。返回window.code=200;window.redirect_uri="https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxnewloginpage?ticket=AWskQEu2O8VUs7Wf2TVOH8UW@qrticket_0&uuid=IZu2xb_hIQ==&lang=zh_CN&scan=1467393709"; 表示登录成功
       c. _表示当前时间戳，13位
   5). 点击登录成功之后，返回redirect_uri表示用户已经在手机端完成了授权过程，需要继续访问当前链接获取wxuin和wxsid

4. 访问登录地址，获得uin和sid
   1). 说明: wxuin和wxsid在后续的通信中都需要用到
   2). api: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxnewloginpage?ticket=AWskQEu2O8VUs7Wf2TVOH8UW@qrticket_0&uuid=%s==&lang=zh_CN&scan=%s
   3). get 请求
   4). 参数说明
       a. uuid 表示当前uuid
       b. scan 表示当前的扫描时间戳 10位
   5). 返回结果
       <error><ret>0</ret><message>OK</message><skey>@crypt_b13bcf4_b49e9a98c34a5e263381f032e6ae18cb</skey><wxsid>gJmKzszzW2KnFIXf</wxsid><wxuin>828185640</wxuin><pass_ticket>vBTQ11xy5sLBraR8BT1K7xGQldAmLRSZBzylJ%2FiEVbV77SgMcNyOBj0seNho8kUM</pass_ticket><isgrayscale>1</isgrayscale></error>
       a. skey
       b. wxsid
       c. wxuin --> 用户唯一的id
       d. pass_ticket

5. 初始化登录页信息
   1). 说明: 初始化登录后页面的信息，获取常用联系人以及微信公众号
   2). api: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxinit?r=%s&lang=zh_CN&pass_ticket=%s
   3). post 请求
   4). post data
       {
          'BaseResponse': {
              'Uin': wxuin,
              'Sid': wxsid,
              'Skey': skey,
              'DeviceID': //随机串 'e'+str(random.random())[2:17]
          }
       }
   5). 返回json串包含当前user的相关信息，BaseResponse内Ret位0表示成功
   
6. 开启微信状态通知
   1). 说明: 有新消息通知的时候会显示
   2). api:  https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxstatusnotify?lang=zh_CN&pass_ticket=%s
   3). post 请求
   4). post data
       {
          'BaseResponse': {
              'Uin': wxuin,
              'Sid': wxsid,
              'Skey': skey,
              'DeviceID': //随机串 'e'+str(random.random())[2:17]
          }
          'ClientMsgId': 13位时间戳,
          'Code': 3  //固定值
          'FromUserName':  userNmae, //从初始化登录信息那边取到
          'ToUserName': userNmae, //从初始化登录信息那边取到
       }
   5). 返回json串，BaseResponse内Ret位0表示成功
   
7. 同步刷新，检查服务端消息
   1). 说明: 为了实时取到消息，需要客户端不断发送请求给服务端查询
   2). api: https://webpush.wx.qq.com/cgi-bin/mmwebwx-bin/synccheck?r=%s&skey=%s&sid=%s&uin=%s&deviceid=%s&synckey=%s&_=%s
   3). get 请求
   4). 参数说明
       a. r --> 13位时间戳
       b. skey 同上,需要url quote
       c. sid 同上
       d. devicedid 同上
       e. synckey由 初始化登录页面信息返回串中Sync的list列表组成, 需要url quote
       f. _ 13位时间戳
   5). 返回值
       window.synccheck={retcode:"xxx",selector:"xxx"}
       retcode:
       a. 0 正常
       b. 1100 失败/登出微信
       c. 1101 从其它设备登录微信网页版
       selector:
       a. 0 正常
       b. 2 新的消息
       c. 7 手机操作了微信
       
8. 获取消息
   1). 说明: 根据同步刷新的接口拿到当前的状态后，如果是新的消息，需要获取新的消息
   2). api: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxsync?sid=%s&skey=%s&pass_ticket=%s
   3). 返回json串
       a. BaseResponse，Ret位0表示返回成功
       b. AddMsgCount 表示新消息个数
       c. AddMsgList 表示新消息列表
          a. MsgType -> 51表示系统原始消息

9. 发送消息
   
```

## 参考
```
1. https://github.com/xiangzhai/qwx/blob/master/doc/protocol.md
2. https://github.com/Urinx/WeixinBot
3. https://github.com/liuwons/wxBot
```
