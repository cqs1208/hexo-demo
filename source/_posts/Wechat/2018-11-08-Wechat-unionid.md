---
layout: post
title: wechat unionid介绍
tags:
- Wechat
categories: Wechat
description: Wechat
---

同一用户，对同一个微信开放平台下的不同应用，unionid是相同的

<!-- more --> 

### UnionID机制说明

​	如果开发者拥有多个移动应用、网站应用和公众帐号，可通过获取用户基本信息中的unionid来区分用户的唯一性，因为只要是同一个微信开放平台帐号下的移动应用、网站应用和公众帐号，用户的unionid是唯一的。同一用户，对同一个微信开放平台下的不同应用，unionid是相同的

![unionid机制说明](/images/Wechat/Wechat_unionid.png)

### 1 公众号

#### 1.1 scope---snsapi_base和snaspi_userinfo 区别

概念：**snsapi_base与snsapi_userinfo属于微信网页授权获取用户信息的两种作用域。** 

**区别：**有无弹框 

- 以snsapi_base为scope发起的网页授权 ，是静默授权 ，用户无感知 
- 以snsapi_userinfo为scope发起的网页授权，是用来获取用户的基本信息的。但这种授权需要用户手动同意，并且由于用户同意过，所以无须关注，就可在授权后获取该用户的基本信息

 #### 1.2 用户access_token 获取

**接口地址**

  ```java
https://api.weixin.qq.com/sns/oauth2/access_token?appid=APPID&secret=SECRET&code=CODE&grant_type=authorization_code
  ```

**返回结果根据授权类型区分**

- 静默授权(snsapi_base)

```java
{
  "access_token": "access_token",
  "expires_in": 7200,
  "refresh_token": "refresh_token",
  "openid": "oU3Orv5e8s-XWOLZI64Kq03fTrAM",
  "scope": "snsapi_base"
}
```

- 非静默授权(snsapi_userinfo)

```java
{
  "access_token": "access_token",
  "expires_in": 7200,
  "refresh_token": "refresh_token",
  "openid": "oU3Orv5e8s-XWOLZI64Kq03fTrAM",
  "scope": "snsapi_userinfo", 
  "unionid": "omfguw3rWOvXtBq5qA5OurgHhJCw"
}
```

**说明**

1. 如果公众平台没有绑定该公众号，那么不会返回unionid属性
2. 返回的json串中，如果用户是静默授权，也不会返回unionid属性
3. 查看前端所有h5项目和后端api项目，发现所有授权方式都使用的是非静默授权（snsapi_userinfo）

#### 1.3 4、通过网页授权access_token和**openid获取用户基本信息（支持**UnionID机制） 

```java
https://api.weixin.qq.com/sns/userinfo?access_token=access_token&openid=openid=zh_CN
```

**返回结果**

```java
{
  "openid": "oU3Orv5e8s-XWOLZI64Kq03fTrAM",
  "nickname": "ಠ.ಠ",
  "sex": 0,
  "language": "zh_CN",
  "city": "",
  "province": "",
  "country": "",
  "headimgurl": "http://thirdwx.qlogo.cn/mmopen/v",
  "privilege": [],
  "unionid": "omfguw3rWOvXtBq5qA5OurgHhJCw"
}
```

**说明**：如果公众平台没有绑定该公众号，那么不会返回unionid属性；

​	    如果公众平台有绑定该公众号，不区分授权类型（静默或非静默），该接口都会返回unionid

### 2 小程序

**小程序获取用户openid接口**

```java
https://api.weixin.qq.com/sns/jscode2session?appid=appid&secret=secret&js_code=js_code&grant_type=authorization_code
```

**返回结果**

```java
{
  "session_key": "p8qIINUbeMtJ6iH/AbpEhw==",
  "openid": "omw4u5R0iuJH8A_RO71DHWAJFwJU",
  "unionid": "omfguw3rWOvXtBq5qA5OurgHhJCw"
}
```

**说明** ：小程序没有区分静默授权和非静默授权（可以认为只有非静默授权即授权时会弹框），

​	    如果公众平台有绑定该公众号，该接口都会返回unionid

### 3 unionId获取的条件

1. 调用接口wx.getUserInfo，从解密数据中获取UnionID。注意本接口需要用户授权，请开发者妥善处理用户拒绝授权后的情况。
2. 如果开发者帐号下存在同主体的公众号，并且该用户已经关注了该公众号。开发者可以直接通过wx.login获取到该用户UnionID，无须用户再次授权。
3. 如果开发者帐号下存在同主体的公众号或移动应用，并且该用户已经授权登录过该公众号或移动应用。开发者也可以直接通过wx.login获取到该用户UnionID，无须用户再次授权

### 4 总结

#### 4.1 unionid一致性结论

​	只要开放平台绑定了小程序和公众号，那么同一用户相对于这些绑定过公众号和小程序的unionid是一致的，

如果是新用户（从未绑定过公众号或小程序）授权时选择拒绝，那么无法获取到unionid