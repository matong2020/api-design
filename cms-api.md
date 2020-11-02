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

  cmsToilet -> {
    // 设备ID
    toilet_id: string
    // 省
    province: string
    // 市
    city: string
    // 区
    district: string
    // 地址，比如 xxx路xxx号
    address: string
    // 名字，比如 xxx购物中心
    location_name: string
    // 楼层
    floor: int
    // 性别；1-男，2-女
    sex: int
    // 编号
    no: int
    // 状态；1-正常，2-套膜不足，3-电量不足，4-疑似故障，5-明确故障，6-套膜不足关闭，7-电量不足关闭
    status: int
    // 物业人员名字
    maintainer_name: string
    // 物业人员手机号
    maintainer_phone: string
    // 物业公司ID
    manager_company_id: string
    // 备注
    remark: string
    // 添加时间，秒级时间戳
    created_time: int
  }

  cmsConsumerPayPerUsage -> {
    amount: int
  }

  paginator -> {
    page: int
    size: int
  }

  cmsThreshold -> {
    // 套膜不足标准
    cover_insufficient: int
    // 电量不足标准，传15代表15%
    battery_insufficient: int
    // 电量关闭标准，传10代表10%
    battery_close: int
    // 疑似故障标准，传50代表50%
    possible_glitch: int
    // 一次使用消耗金额，单位为分
    cost_per_usage: int
  }

  cmsDepositSpec -> {
    id: string
    // 金额，单位为分
    amount: int
    // 付费的次数
    pay_for_usage_count: int
    // 赠送的次数
    free_usage_count: int
    // 免费次数过期时间，单位为天
    free_usage_expired_days: int
  }


APIs

  POST /cms/api/users/v1/login
    desc: 登陆
    input: {
      phone: string
      password: string
    }
    biz_errors: {
      3001: 登陆错误
    }
    output: {
      token: string
    }

  POST /cms/api/users/v1/list_toilets
    desc: 马桶列表
    input: {
      // 分页
      paginator: #paginator
      // 开始时间，包含
      start_time: int
      // 结束时间，不包含
      end_time: int
      // 省
      province: string
      // 市
      city: string
      // 区
      district: string
      // 地址，比如 xxx路xxx号，模糊匹配
      address: string
      // 名字，比如 xxx购物中心，模糊匹配
      location_name: string
      // 楼层
      floor: int
      // 性别；1-男，2-女
      sex: int
      // 编号
      no: int
      // 马桶ID，模糊匹配
      toilet_id: string
      // 物业人员名字，模糊匹配
      maintainer_name: string
      // 物业公司，模糊匹配
      manager_company: string
      // 状态；1-正常，2-套膜不足，3-电量不足，4-疑似故障，5-明确故障，6-套膜不足关闭，7-电量不足关闭
      status: int
    }
    output: {
      list: []#cmsToilet
    }

  POST /cms/api/users/v1/update_toilet
    desc: 更新马桶
    input: {
      // toilet的所有字段都要传有意义的值，不要因为不更新就不传
      toilet: #cmsToilet
    }

  POST /cms/api/users/v1/add_toilet
    desc: 添加马桶
    input: {
      // toilet的所有字段都要传有意义的值
      toilet: #cmsToilet
    }
    biz_errors: {
      3002: 该马桶已存在
      3003: 物业人员不存在
      3004: 物业公司不存在
    }

  POST /cms/api/users/v1/get_threshold
    desc: 获取阈值
    output: #cmsThreshold

  POST /cms/api/users/v1/set_threshold
    desc: 更新阈值
    input: {
      // threshold的所有字段都要传有意义的值，不要因为不更新就不传
      threshold: #cmsThreshold
    }

  POST /cms/api/users/v1/list_toilet_usages
    desc: 马桶使用列表
    input: {
      paginator: #paginator
      toilet_id: string
    }
    output: {
      list: []{
        // 马桶ID
        toilet_id: string
        // 使用时间，秒级时间戳
        created_time: int
        // 一天里的第几次使用
        no_of_day: int
        // 使用时长，单位为秒
        duration: int
      }
    }

  POST /cms/api/users/v1/get_consumer_pay_per_usage
    desc: 获取单次消费费用
    output: #cmsConsumerPayPerUsage

  POST /cms/api/users/v1/set_consumer_pay_per_usage
    desc: 设置单次消费费用
    input: #cmsConsumerPayPerUsage

  POST /cms/api/users/v1/list_deposit_specs
    desc: 充值配置列表
    output: {
      list: []#cmsDepositSpec
    }

  POST /cms/api/users/v1/delete_deposit_spec
    desc: 删除充值配置
    input: {
      deposit_spec_id: string
    }

  POST /cms/api/users/v1/add_deposit_spec
    desc: 添加充值配置
    input: #cmsDepositSpec

  POST /cms/api/users/v1/list_companies
    desc: 公司列表
    output: {
      list: []{
        id: string
        name: string
      }
    }

  POST /cms/api/users/v1/list_region_provinces
    desc: 省份列表
    output: {
      list: []string
    }

  POST /cms/api/users/v1/list_region_cities
    desc: 城市列表
    input: {
      province: string
    }
    output: {
      list: []string
    }

  POST /cms/api/users/v1/list_region_districts
    desc: 区列表
    input: {
      province: string
      city: string
    }
    output: {
      list: []string
    }

  POST /cms/api/users/v1/import_toilets
    desc: 批量导入马桶
    input: {
      // 这个接口不是通常的API请求，这个字段只是为了方便文档记载。这个接口需要上传xlsx文件
      __doc_placeholder__: string
    }
    output: {
      // 这个接口返回的 code 肯定都是0。如果有错误，具体错误描述放在这个字段里，列表里的每个元素是一个错误的描述
      error_list: []string
    }

  POST /cms/api/users/v1/download_import_toilets_template
    desc: 下载批量导入马桶模板xlsx文件
    output: {
      // 这个接口不是通常的API请求，这个字段只是为了方便文档记载。这个接口下载xlsx文件
      __doc_placeholder__: string
    }

```