---
title: timetagger定制
date: '2021-11-17 18:11:56 +0800'
tags: 
   - 时间管理
categories:
   - 无聊的开发
---

## 为什么选择 timetagger

树莓派 3B 可能只带的动这个应用，更新版本的可能跑得动其他的。像 kimai 等可能跑不起来。

## 为什么要定制

timetagger 没有提供登录功能，或者说登录功能太简陋，于是需要自行定制。

## 如何定制

### 定制界面

把官方自带的界面 copy 到自己 pages 目录下。

修改 login.md，所有的登录请求都是到这个界面下完成的，所以需要修改它，给它增加登录表单，具体细节看源码。

### totp 工具

```
import pyotp

username = "your_username"
gtoken = pyotp.random_base32(64)
data = pyotp.totp.TOTP(gtoken).provisioning_uri(username, issuer_name="MyTimeTagger")
print("gtoken", gtoken)
print("qrcode data", data)
```

生成代码参考[python 实现 google authenticator 认证 - Pythia 丶陌乐](#root/ZycHBYUoADWo/Vx6AUcpGHodQ/ZXufmBwkSU3E/zXZRGpQeDzVc/wE4RlKFYqoH1/KUQrJURuWW9M)。

其中 gtoken 就是 totp 密钥，在 run.py 中用到。

qrcode data 是用来生成二维码的内容，随便找个工具，用这个内容生成二维码，然后使用 authy 这样的工具扫码即可。

### 运行代码

修改静态界面资源到自己的 pages 目录。

引入 pyotp 包。

增加一个用户字典，保存用户名、密码、totp 密钥信息。

修改 webtoken_for_localhost 函数。增加用户登录验证。

## 部署

timetagger 数据默认放在用户目录下的 \_timetagger 目录下。

可以使用 ln 命令引到其他目录下。

## 小结

完整代码[https://github.com/gomi1992/timetagger_app](https://github.com/gomi1992/timetagger_app)

这不是 totp 的正确使用方式。

应该先验证用户名密码，再给界面去输入 totp。

但是目前登录逻辑上还是跟这个流程保持一致。

timetagger 的简单定制到此为止。
