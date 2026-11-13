---
title: JWT 原理与登录注册
tags: [nodejs, jwt, jsonwebtoken, authentication]
course: "A Complete NODE JS Course (#105-#107)"
created: 2026-11-13
---

# JWT 原理与登录注册

> [!abstract] 本篇定位
> 覆盖课程 #105–#107。先弄懂 **JWT 是什么、凭什么可信**，再实现 signup（签发）和 login（校验 + 签发）两个接口。

## 一、JWT 是什么（#105）

![JWT 登录流程图](img/course-auth02-jwt-flow.jpg)
*图 A2-01 · 视频 #105 · LOGIN 流程：POST /login → 密码正确则发 SECRET 签名的 JWT*

![JWT 签发与验签图](img/course-auth02-jwt-structure.jpg)
*图 A2-02 · 视频 #105 · SIGNING（HEADER+PAYLOAD+SECRET）与 ACCESS（TEST SIGNATURE）*

JWT（JSON Web Token）= 一串带签名的令牌，分三段 `xxx.yyy.zzz`：

| 段 | 内容 | 编码 |
| :--- | :--- | :--- |
| Header | `{"alg": "HS256", "typ": "JWT"}` | Base64Url |
| Payload | `{"id": "5c8...", "iat": 1699..., "exp": ...}` | Base64Url |
| Signature | `HMAC_SHA256(header + "." + payload, SECRET)` | Base64Url |

**信任模型**：服务器用只有自己知道的 SECRET 签名。客户端任何改动（改 payload 里的 id、改 exp）都会让签名对不上——服务器重算一遍签名即可验真，**不需要存 session、不需要查库确认令牌本身**。官方科普：[JWT Introduction](https://jwt.io/introduction)

> [!warning] 两个必然推论
> 1. Payload 只是 Base64——**不是加密**！任何人解码都能看到用户 id。禁止放敏感信息。
> 2. SECRET 泄露 = 全部用户可被伪造。SECRET 应当是环境变量里一长串随机字符（[[02-Express/05-环境变量与配置|环境变量篇]]），且不同环境不同 SECRET。

## 二、签发函数 signToken（#107）

![login/signToken 代码](img/course-auth02-login-code.jpg)
*图 A2-03 · 视频 #107 · authController：signToken 与 login 的错误分支*

```javascript
const jwt = require('jsonwebtoken');

const signToken = (id) =>
  jwt.sign({ id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRES_IN,   // 如 '90d'/'1h'
  });
```

- payload 只放 `id`——每次请求验签后**仍要用 id 查库**（权限可能变、密码可能被重置），令牌只是"身份线索"
- `expiresIn` 让令牌带 `exp`，过期后验签直接失败

## 三、signup：注册即登录（#106）

![signup 响应](img/course-auth02-signup.jpg)
*图 A2-04 · 视频 #106 · Postman 注册成功并拿到 token*

```javascript
exports.signup = async (req, res, next) => {
  const newUser = await User.create({ /* 白名单字段，见 [[06-认证与授权/01-用户模型与密码加密]] */ });
  const token = signToken(newUser._id);

  res.status(201).json({ status: 'success', token, data: { user: newUser } });
  // token 与数据一起返回：前端存 localStorage/内存，后续请求挂在 Authorization 头
};
```

## 四、login：验证凭据再发令牌（#107）

```javascript
exports.login = async (req, res, next) => {
  const { email, password } = req.body;

  // 1) 检查输入
  if (!email || !password) {
    return next(new AppError('Please provide email & Password for log in!', 400));
  }
  // 2) 查用户 —— select:false 的密码字段要显式加回来
  const user = await User.findOne({ email }).select('+password');
  // 3) 比对密码（user 可能不存在 → correctPassword 里 undefined 会 false，不会抛错）
  if (!user || !(await user.correctPassword(password, user.password))) {
    return next(new AppError('Incorrect email or Password!', 401));   // 401 = 未认证
  }
  // 4) 一切正常 → 发令牌
  const token = signToken(user._id);
  res.status(200).json({ status: 'success', token });
};
```

> [!important] 为什么错误消息故意含糊
> "邮箱不存在"和"密码错误"合并成一句 `Incorrect email or password`——分开提示等于帮攻击者枚举有效邮箱。这叫防用户枚举（user enumeration）。

## 五、之后怎么用

前端拿到 token 后，每个受保护请求带：
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```
服务器如何校验这个头、并把用户挂进请求对象 → [[06-认证与授权/03-路由保护与权限|路由保护与权限]]。

## 相关笔记

- 上一篇：[[06-认证与授权/01-用户模型与密码加密|用户模型与密码加密]]
- 下一篇：[[06-认证与授权/03-路由保护与权限|路由保护与权限]]
- 关联：[[01-Node.js基础/03-构建HTTP服务器|HTTP 是无状态的——JWT 正是为无状态认证而生]]
- 关联：[[04-Mongoose实战/05-虚拟属性与中间件|bcrypt.compare 挂在 Schema methods 上]]

## 附：本篇课程索引

| 集数 | 主题 |
| :--- | :--- |
| #105 | JWT 三段结构与信任模型 |
| #106 | signup 接口返回 token |
| #107 | signToken + login 全流程 |
