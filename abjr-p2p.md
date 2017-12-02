# 第三方固收平台接口设计-v1

该文档是用于第三方固收产品平台对接业务系统（客户端系统）的接口(版本号1.0)的开发指引。

各个客户端系统使用第三方固收平台的接口时，需由第三方固收平台为各个客户端系统分配appId，appId代表客户端系统编号。
客户端系统生成RSA 1024bit的密钥对，并将公钥进行BASE64编码后提交给第三方固收平台。客户端系统使用私钥进行请求签名和响应数据中敏感数据的解密。

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [第三方固收平台接口设计-v1](#第三方固收平台接口设计-v1)
	1. [通用约定](#1)
	2. [异常编码表](#2)
	3. [签名生成规则](#3)
		- [3.1 待签名的字符串组成](#3.1)
		- [3.2 签名和加密](#3.2)
	4. [敏感信息的加密处理](#4)
	5. [产品列表](#5)
		- [5.1 产品列表接口](#5.1)
	6. [用户模块](#6)
		- [6.1 是否开户（玖富）](#6.1)
		- [6.2 开户绑卡接口](#6.2)
		- [6.3 确认购买接口](#6.3)
		- [6.4 发送支付验证码接口](#6.4)
		- [6.5 确认支付接口](#6.5)
		- [6.6 银行卡列表接口](#6.6)
		- [6.7 债权信息查询接口](#6.7)
		- [6.8 我的资产接口](#6.8)
		- [6.9 订单查询接口](#6.9)
		- [6.10 查询合同接口](#6.10)



<!-- /TOC -->
## <h1 id="1">通用约定</h1>

- 接口格式说明

  本文档描述的全部接口均采用如下格式：

  ```
    GET  /jf/xxx
    POST /jf/xxx

    - URL参数 //URL参数

    - 请求消息 //请求消息体

    - 响应消息 //响应消息体

    - 异常编码 //异常响应码

  ```

- 命名约定

  针对玖富的接口以/api/jf/**作为前缀，针对app的接口以/jf/**为前缀

- 传输协议

  1. 全部接口（特殊说明除外）均采用HTTPS作为传输协议。

  1. 数据类型约定

        - 日期类型：整数(距离1970-01-01 00:00:00)

  1. 请求消息约定

        - 请求头说明

          > Content-Type:application/json

          请求消息体为JSON格式时，需要包含该请求头

          > Accept:application/json

          响应消息体为JSON格式时，需要包含该请求头

        - URL参数说明

          每个调用第三方固收平台的URL都必须包含如下四个参数：

      - appId: 第三方固收平台为使用该平台的业务系统分配的id值
      - data: 字符串,对于不同的请求该值应唯一

      示例：
      ```
      GET /jf/productList?data=alskdjfalksdjflkasjdfkasdjlfqoweifjoaifzxcva&appId=abjr
      ```

  1. 响应消息约定

        - 响应消息体通用格式

      ```
			{
				errorCode:"0000", //响应码(0000:成功,其他：错误码)
				msg:"",     //和响应码对应的文本信息
				data:业务信息   //响应结果对应的业务信息，可以是JSON对象、数组或基本类型值等
			}
      ```

    	**注意**：文档中具体接口中响应消息格式均代表data属性的消息结构。

## <h1 id="2">异常编码表</h1>


```
{
  code:"9999",
  message:"服务端异常"
}
```

编码      | 说明
-------- | ------ |
1001		| 绑卡失败
1002    		| 用户未绑卡
1003		| 玖富端实名认证失败
1004		| 玖富端开户失败
2001		| 玖富生成订单失败
3001		| 开通个人金账户失败
4001		| 低于最小起投量
4002		| 剩余额度不足，只能全部购买
4003		| 产品额度查询失败
5001		| 购买支付失败
5002            | 微支付账户查询失败
5003            | 富友状态和银行卡四要素验证失败
5004            | 微支付开户请求失败
5005            | 微支付开户失败
6001		| 写数据库失败

## <h1 id="3">签名生成规则</h1>

例如：

```
appId = abjr

客户端系统私钥：

MIICdgIBADANBgkqhkiG9w0BAQEFAASCAmAwggJcAgEAAoGBALj/iE8ik7uBQOwhPOpZiJ+PmFCPuIMk+vbyJo/voeVZAKLYHgG22YLJTXnbn9d4IV7uSLv/oT6zhWUGmLd+20a0+IoTI6SR/upQ7yDvaeu3UroZjKVnJEBKD7r+H1IlQbs07eMz3tQo2jxZA3IlPU8kKPXJNw7qfjSqBbwQzIlvAgMBAAECgYEAq90Y8QuaW1OU0MmAIebTughY5F7gd1VfoRMNKCLjMIIiySYlmkoYgBwrUc3rDO2ZcuvDvoOZdPqqLlSWg8HiSpVuNEUZ0ltAE6dDn3GEGVCQGi+mbMfa/67oM41RoAgJ7CaieW27l84bMgJJ5jJuRb9zEKGCbsKHrnOAfhcJKWECQQDb6qBKyo9I0ECtAejDYQXFLb4nlBVcOhUn+y9QwP1g74v9MHg6OQ2gOLVr88BD6So7vv5MLklK1l3DG+YpwQWNAkEA11oySqspWKU5NX86vZICtA9gjoS2ZZpdad0ZrCtdQN+Ucrq+T9K51A0MsnKr/gC9GDWxgLAaDOJH6VKtfR916wJALUHQsPOUnyh0VuZQr3yVAmoSevSnnK47UloH97dvrXY+ueEyrNC29CUXeNrV02P1lAwPK0BPRv5sl01zhV46tQJAdFbw5m/TVWVlI6aJSFJyDW5lPnkpxHgBUSi2LtH6fgqLOvPxzlPMOmeWXW0fx4gEn+iZ7Si12hIAwWb9/KObYwJAUZRRluISKf0gK1QfLG6v12fX2PloK/+XhRL15IYHDl3Q4MxzxR7S1Ge4czjaSAsXh5xwOt0uun19WV2PMKB+Pw==

客户端公钥：

MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC4/4hPIpO7gUDsITzqWYifj5hQj7iDJPr28iaP76HlWQCi2B4BttmCyU1525/XeCFe7ki7/6E+s4VlBpi3fttGtPiKEyOkkf7qUO8g72nrt1K6GYylZyRASg+6/h9SJUG7NO3jM97UKNo8WQNyJT1PJCj1yTcO6n40qgW8EMyJbwIDAQAB

GET /jf/productList?pageNo=1&pageSize=10&appId=abjr
```

### <h1 id="3.1">待签名的字符串组成</h1>

  > 待签名的字符串=data参数中的json字符串


  示例的待签名的字符串为：

  ```
  pageNo=1&pageSize=10
  ```

### <h1 id="3.2">签名和加密</h1>

  > 对待签名的字符串计算MD5进行签名。

  示例签名值：

  ```
		7556890632c21fa4910b87976a764645
  ```
  > 对签名后的字符串和待签名的字符串转成json后进行RSA加密，作为data的参数值，实例代码如下：
        JSONObject jsonObject = new JSONObject();

        jsonObject.put("channelId","abjr");
        jsonObject.put("userId", "123456");
        jsonObject.put("idType", "0");
        jsonObject.put("idCard", "123456789012345678");
        jsonObject.put("idName","测试用户");
        jsonObject.put("mobile", "13800138000");

        Map<String, String> params = new TreeMap<>();
        params.put("channelId","abjr");
        params.put("userId", "123456");
        params.put("idType", "0");
        params.put("idCard", "123456789012345678");
        params.put("idName","测试用户");
        params.put("mobile", "13800138000");

        StringBuilder sb = new StringBuilder();

        Iterator<String> iter = params.keySet().iterator();

        while (iter.hasNext()) {

           String key = iter.next();

           String value = params.get(key);

           sb.append(key + "=" + value + "&");
        }

        String result = sb.toString();


        if (result.endsWith("&")) {
           result = result.substring(0, result.length() - 1);
        }

        System.out.println("needSing = " + result);

        //加签
        String sign = MD5Util.md5Encode(result);

        jsonObject.put("sign", sign);

        //加密
        String data = RsaHelper.encipher(jsonObject.toJSONString(), pubkey);

  >请求的完整URL

  ```
   http://hostname:port/jf/productList?appId=abjr&data=E5f+hvwa/Ef+gwdI7HO9JGeLAOo+Qaetz4D4eq2770qz+jW83blUuPN5x7HDNRJgjWYlrU7kKigyCQV9mdOxt8sd6fdMy9LllTNhPtgb1ETsIPKFoXrb79puxZ2+fopWquaA1a50AZnYH/HlEH9NTt8iawfCXRUK5vCq1RA2+sM=
  ```

## <h1 id="4">敏感信息的加密处理</h1>

第三方固收平台使用AES算法对敏感信息加密处理。

加密算法：AES/ECB/PKCS5Padding


## <h1 id="5">产品模块</h1>

### <h1 id="5.1">产品列表</h1>

#### <h1 id="5.1.1">产品列表</h1>
    APP端使用webView工程中的通用产品列表进行查询

```
GET /lcb_webView/lcbproductinfo/getPrdListNew_interface?pclass2=29
```

- URL参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
pclass2   | string | required | 产品类型，由产品菜单接口返回

- 响应消息

```
 "data": {
    "firstPage": true,
    "firstResult": 0,
    "hasNextPage": false,
    "hasPreviousPage": false,
    "lastPage": true,
    "lastPageNumber": 1,
    "linkPageNumbers": [
      1
    ],
    "nextPageNumber": 2,
    "pageSize": 10,
    "previousPageNumber": 0,
    "result": [
      {
        "continueStatus": "B03",
        "createTime": 1512004622000,
        "creater": "system",
        "endTime": 1539761743000,
        "incomePaymentWay": "1",
        "isContinueProduct": 0,
        "issuePeriod": "1",
        "lc_account_type": "B8010",
        "lc_trustee_type": "B134001",
        "matchFundType": 3,
        "maxInvestamt": 100.33,
        "minInvestamt": 0.01,
        "prdCat": 10,
        "prdId": 170614172906431,
        "prdName": "安邦金融",
        "productId": "171124153629688",
        "profit": 8,
        "rest": 100.33,
        "riskLevel": 2,
        "sale": 0,
        "startTime": 1509953734000,
        "status": 1,
        "timeLong": 45,
        "timeUnit": "D",
        "total": 100.33,
        "updateTime": "2017-11-30 09:29:24.0",
        "updater": "system"
      }
    ],
    "thisPageFirstElementNumber": 1,
    "thisPageLastElementNumber": 1,
    "thisPageNumber": 1,
    "totalCount": 1
  }

```

## <h1 id="6">用户模块</h1>

### <h1 id="6.1">是否开户（玖富）</h1>
```
POST /jf/register
```

- 请求参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
idCard   | string | required | 身份证号
mobile   | string | required | 手机号
idName   | string | required | 姓名

- 响应消息

参数名    | 说明
-------- | ------
userId   | 第三方固收平台用户ID
newUserFlag   | 新用户标识
riskLevel |用户风险等级，1-5由低到高
bankCardNoLastFour | 银行卡后四位
dealLimit | 单笔限额
dayLimit  | 单日限额
bankName  | 银行名称


- 异常编码：1002、1003、1004

## <h1 id="6.2">开户绑卡接口</h1>

开户接口

```
POST /jf/bindCard
```

- 请求参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
userId   | string | required | 第三方固收平台用户ID
bankCardNo   | string | required | 银行卡号
bankCode   | string | required | 银行编号
bankType | string | required | 银行类型


- 响应消息

参数名    | 说明
-------- | ------
id   | 绑卡记录id

- 异常编码：1001

## <h1 id="6.3">确认购买接口</h1>
产品购买页面调用

```
POST /jf/confirmBuy
```

- URL参数


参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
userId   | string | required | 第三方固收平台用户ID
amount   | Double | required | 购买金额，单位：元
productId | Long | required | 产品ID
- 响应消息


- 异常编码 4001

## <h1 id="6.4">发送支付验证码短信接口</h1>
用于发送验证短信

```
POST /jf/sendSms
```
- URL参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
orderId   | Long | required | 订单号
mobile   | String | required | 手机号

- 响应消息
0000 代表成功，其余代表失败

- 异常编码 9999


## <h1 id="6.5">确认支付接口</h1>
支付确认页面调用

```
POST /jf/confirmPay
```

- URL参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
orderId   | Long | required | 订单号
smsCode   | String | required | 验证码

- 响应消息
0000 success代表成功，其余代表失败

- 异常编码 9999

## <h1 id="6.6">银行卡列表接口</h1>
绑卡时查询银行卡列表

```
POST /jf/banklist
```

- URL参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
bankcode   | String | non-required | 银行编码

传银行编码可查询单个银行信息

- 响应消息
0000 success代表成功，其余代表失败

返回银行卡信息列表中的字段信息：

参数名    | 说明
-------- | ------
merPayType   | 查询类型
bankName   | 银行名称
bankCode | 银行编码
dayLimit | 日限额
monthLimit | 月限额
singleLimit  | 单笔限额


- 异常编码 9999

## <h1 id="6.7">债权信息查询接口</h1>
查询用户债权信息

```
POST /jf/claimInfo
```

- URL参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
orderNo   | String | non-required | 订单号

- 响应消息



参数名    | 说明
-------- | ------



- 异常编码 9999

## <h1 id="6.8">我的资产接口</h1>
查询我的资产

```
POST /jf/accountOverview
```

- URL参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------
userId   | Long | required | 账户ID

- 响应消息
0000 success代表成功，其余代表失败

返回银行卡信息列表中的字段信息：

参数名    | 说明
-------- | ------
capital   | 投资本金
earnings   | 预期收益
haveEarninged | 已赚取收益

- 异常编码 9999

## <h1 id="6.9">订单查询接口</h1>
查询用户订单

```
POST /jf/investmentInfo
```

- URL参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------



- 响应消息




参数名    | 说明
-------- | ------



- 异常编码 9999

## <h1 id="6.10">查询合同接口</h1>
查询订单对应的合同

```
POST /jf/contractInfo
```

- URL参数

参数名      | 类型     | 规则       | 说明
-------- | ------ | -------- | -------------



- 响应消息




参数名    | 说明
-------- | ------



- 异常编码 9999
