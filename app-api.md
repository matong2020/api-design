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

  POST /app/api/users/v1/upload_wxa_user_info
    desc: 上传微信用户信息
    input: {
      // 小程序 wx.getUserInfo 接口返回的 UserInfo 的部分字段，不传或传 null 代表不更新， https://developers.weixin.qq.com/miniprogram/dev/api/open-api/user-info/wx.getUserInfo.html
      user_info: {
        nickname: string
        gender: int
      }
      // 小程序 getPhoneNumber 接口返回的部分字段，不传或传 null 代表不更新， https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/getPhoneNumber.html
      phone_raw: {
        encrypted_data: string
        iv: string
      }
    }

  POST /app/api/users/v1/me
    desc: 获取自己的用户信息
    output: {
      id: string
      nickname: string
      // 手机号码，空字符串代表还没绑定手机号
      phone: string
      // 性别，1-男，2-女，0-未知
      sex: int
      // 头像 url
      avatar_url: string
      // 剩余使用次数，包含赠送的
      remaining_usage_count: int
      // 注册时间，秒级时间戳
      created_time: int
    }

  POST /app/api/users/v1/feedback
    desc: 意见反馈
    input: {
      message: string
    }

  POST /app/api/orders/v1/list_deposit_specs
    desc: 获取充值配置列表
    output: {
      list: []{
        id: string
        // 金额，单位为分
        amount: int
        // 付费的次数
        pay_for_usage_count: int
        // 赠送的次数
        free_usage_count: int
      }
    }

  POST /app/api/orders/v1/init_wxa_deposit
    desc: 发起微信小程序支付充值
    input: {
      // 充值配置ID
      spec_id: string
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
      open_id: string
      // 充值订单ID
      deposit_id: string
    }

  POST /app/api/orders/v1/get_deposit_status
    desc: 获取充值订单状态
    input: {
      // 发起充值接口返回的充值订单ID
      deposit_id: string
    }
    biz_errors: {
      2101: 充值订单ID未找到
    }
    output: {
      // 状态：1-完成，2-支付中，3-失败
      pay_status: int
    }

  POST /app/api/orders/v1/list_deposits
    desc: 获取用户自己的充值记录
    input: {
      // cursor 用来标记流式列表接口拉取进度，每次请求需要带上上次请求返回的 cursor，第一次请求传空字符串
      cursor: string
      // 有些时候返回的条目数量可能不会严格等于这个值
      size: int
    }
    output: {
      // 下次拉取的开始标记，空字符串代表结束
      cursor: string
      list: []{
        id: string
        // 金额，单位为分
        amount: int
        // 状态：1-完成，2-支付中，3-失败
        status: int
        // 订单创建时间，秒级时间戳
        created_time: int
      }
    }

  POST /app/api/toilets/v1/list_toilets_nearby
    desc: 附近的马桶列表
    input: {
      // 纬度
      latitude: float
      // 经度
      longitude: float
      // cursor 用来标记流式列表接口拉取进度，每次请求需要带上上次请求返回的 cursor，第一次请求传空字符串
      cursor: string
      // 有些时候返回的条目数量可能不会严格等于这个值
      size: int
    }
    output: {
      // 下次拉取的开始标记，空字符串代表结束
      cursor: string
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

  POST /app/api/toilets/v1/get_toilet_status
    desc: 获取马桶状态
    input: {
      // 二维码里的ID
      bar_code_id: string
    }
    biz_errors: {
      2201: 找不到马桶
    }
    output: {
      // 状态：1-空闲，2-占用，3-维修中
      toilet_status: int
    }

  POST /app/api/toilets/v1/use
    desc: 使用马桶
    input: {
      // 二维码里的ID
      bar_code_id: string
    }
    biz_errors: {
      2202: 找不到马桶
      2203: 马桶不是空闲状态
      2204: 余额不足
    }
    output: {
      order_id: string
    }

  POST /app/api/orders/v1/list_use_records
    desc: 获取用户自己的使用记录
    input: {
      // cursor 用来标记流式列表接口拉取进度，每次请求需要带上上次请求返回的 cursor，第一次请求传空字符串
      cursor: string
      // 有些时候返回的条目数量可能不会严格等于这个值
      size: int
    }
    output: {
      // 下次拉取的开始标记，空字符串代表结束
      cursor: string
      list: []{
        id: string
        // 马桶硬件ID
        hardware_id: string
        // 订单状态：1-进行中，2-完成
        status: int
        // 地点名称，比如某某购物中心
        location_name: string
        // 地址，比如xxx市xxx路xxx号
        address: string
        // 使用时间，秒级时间戳
        record_time: int
      }
    }

```