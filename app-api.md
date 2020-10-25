# 上策智能马桶APP接口
本文档记录上策项目的APP客户端与服务端的通信接口设计。

## 总体
1. http 通信，全部都是 POST 请求
2. http 请求数据都放入 http body 中，采用 json 序列化
3. 登陆成功后，服务端会给客户端返回一个 TOKEN，可以用来验证用户的身份，客户端后续发起的请求需要在 http header `X-Token` 中带上这个 TOKEN
4. 服务端返回的 http status code 全部都是 `200`，主要是为了简单地与中间可能的负载均衡返回的状态码区分开
5. 服务端返回的业务状态和各个接口不同的业务数据放在 http body 中，采用 json 序列化；其中 `code` 字段是业务状态码，`msg` 字段是业务状态描述，`data` 字段是接口返回的业务数据；只有在成功的状态下，`data` 里的数据才有意义
6. 很多接口共用了一些业务状态，放到 `通用业务状态` 一节中，不再在每个接口中列出
7. 为了方便记录接口，本文档采用自定义的接口文档记录方式，说明在 `接口记录格式说明` 一节

## 通用业务状态

| code | msg             | 说明           |
| ---- | --------------- | -------------- |
| 0    | OK              | 成功           |
| 1001 | unauthenticated | 需要登陆       |
| 1011 | invalid-method  | 请求方法不合法 |
| 1012 | invalid-input   | 请求参数不合法 |
| 1101 | server-error    | 服务端错误     |

## 接口记录格式说明
分成两个区：`schemas` 和 `APIs`。

/* ... */ 是为了解释加的说明

### schemas
记录多个接口公用的数据结构，单个示例
```
  Book -> {                 /* schema名，后面用到时用 #Book 引用 */
    id: int                 /* 字段及其对应的类型 */
    // 书名                 /* 对下面字段的说明 */
    title: string
    // 作者
    author: string
    referenced_by: []#Book  /* []表示列表，这里表示 referenced_by 的类型是 Book列表 */
    more_type: [string]{    /* string -> object 哈希表结构 */
        field_1: int
        field_2: string
    }
  }
```

### APIs
记录接口详情，单个示例
```
  POST /app/api/users/v1/wxa_login  /* http 方法+路径 */
    desc: 微信小程序登陆             /* 描述 */
    input: {                        /* request payload 结构，如果请求无需 payload ，没有这个 `input` 节 */
      code: string
    }
    biz_errors: {                   /* 业务状态码及其说明，如果该接口的所有状态在上面的 通用业务状态 中，没有 `biz_errors` 节 */
      2001: 微信系统繁忙，请重试
      2002: code无效
      2003: 微信频率限制
    }
    output: {                       /* response payload 结构，如果无返回 payload ，没有这个 `output` 节 */
      // session token
      token: string
    }
```

## 接口详情

```
schemas

  paginator -> {
    // 从 0 开始
    page: int
    size: int
  }


APIs

  POST /app/api/users/v1/wxa_login
    desc: 微信小程序登陆
    input: {
      code: string
    }
    biz_errors: {
      2001: 微信系统繁忙，请重试
      2002: code无效
      2003: 微信频率限制
    }
    output: {
      // session token
      token: string
    }

  POST /app/api/users/v1/me
    desc: 获取自己的用户信息
    input: {
      // 以下字段是小程序 wx.getUserInfo 接口返回的子集， https://developers.weixin.qq.com/miniprogram/dev/api/open-api/user-info/wx.getUserInfo.html
      userInfo: {
        nickName: string
        gender: int
      }
      rawData: string
      signature: string
      encryptedData: string
      iv: string
    }

  POST /app/api/users/v1/me
    desc: 获取自己的用户信息
    output: {
      id: string
      nickname: string
      // 手机号码，空字符串代表还没绑定手机号
      phone: string
      // 剩余使用次数，包含赠送的
      remaining_usage_count: int
      // 性别，1-男，2-女，-1-未知
      sex: int
      // 注册时间，秒级时间戳
      created_time: int
    }

  POST /app/api/users/v1/send_phone_binding_sms
    desc: 发送手机号码绑定验证码短信
    input: {
      phone: string
    }
    biz_errors: {
      2004: 请求发短信过于频繁
    }

  POST /app/api/users/v1/bind_phone
    desc: 绑定手机号码
    input: {
      phone: string
      binding_code: string
    }
    biz_errors: {
      2005: 手机号码和验证码不匹配
      2006: 手机号码已被绑定
    }

  POST /app/api/users/v1/send_phone_login_sms
    desc: 发送手机号码登陆验证码短信
    input: {
      phone: string
    }
    biz_errors: {
      2007: 请求发短信过于频繁
    }

  POST /app/api/users/v1/phone_login
    desc: 手机号码登陆
    input: {
      phone: string
      login_code: string
    }
    biz_errors: {
      2008: 手机号码和验证码不匹配
    }
    output: {
      // session token
      token: string
    }

  POST /app/api/users/v1/feedback
    desc: 意见反馈
    input: {
      message: string
    }

  POST /app/api/orders/v1/init_wxa_deposit
    desc: 发起微信小程序支付充值
    input: {
      // 充值金额，单位为分
      amount: int
    }
    output: {
      // 微信小程序调起支付需要的参数，详见https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=7_7&index=5
      pay_args: {
        appId: string
        timeStamp: string
        nonceStr: string
        package: string
        signType: string
        paySign: string
      }
      // 我们生成的支付ID，用来查询是否支付成功
      pay_id: string
    }

  POST /app/api/orders/v1/get_pay_status
    desc: 获取支付状态
    input: {
      // 调用支付接口返回的支付ID
      pay_id: string
    }
    biz_errors: {
      2101: 支付ID未找到
    }
    output: {
      // 状态：1-支付中，2-支付成功，3-支付失败
      pay_status: int
    }

  POST /app/api/orders/v1/list
    desc: 获取用户自己的订单列表
    input: #paginator
    output: {
      list: []{
        id: string
        // 订单类型：1-充值，2-购买服务，3-提现
        type: int
        // 金额变更，单位为分，充值时这个值是正数，购买服务、提现时是负数
        amount: int
        // 订单状态：1-进行中，2-完成
        status: int
        // 订单产生时时间，秒级时间戳
        time: int
        // 地点名称，比如某某购物中心，仅在订单类型是 购买服务 时有该字段
        location_name: string
        // 地址，比如xxx市xxx路xxx号，仅在订单类型是 购买服务 时有该字段
        address: string
      }
    }

  POST /app/api/toilets/v1/list_toilets_nearby
    desc: 附近的马桶列表
    input: {
      // 纬度
      latitude: float
      // 经度
      longitude: float
      // 性别：1-男，2-女，-1-男厕女厕都列出来
      sex: int
      paginator: #paginator
    }
    output: {
      list: []{
        washroom: {
          // 地点名称，比如某某购物中心
          location_name: string
          // 地址，比如xxx市xxx路xxx号
          address: string
          // 层数
          floor: int
          // 性别：1-男，2-女
          sex: int
          // 跟用户的距离，单位为米
          distance: int
        }
        toilets: []{
          hardware_id: string
          // 状态：1-空闲，2-占用，3-维修中
          toilet_status: int
          // 状态状态持续时长，单位为秒
          status_lasting_seconds: int
        }
      }
    }

  POST /app/api/toilets/v1/use
    desc: 使用马桶
    input: {
      // 二维码里的ID
      bar_code_id: string
    }
    biz_errors: {
      2201: 找不到马桶
      2202: 马桶不是空闲状态
      2203: 余额不足
    }
    output: {
      order_id: string
    }

```