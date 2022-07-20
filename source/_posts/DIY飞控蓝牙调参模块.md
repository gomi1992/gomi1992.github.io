---
title: DIY飞控蓝牙调参模块
urlname: yho3hs
date: '2020-08-18 14:46:19 +0800'
tags:
  - 无人机
  - 电子
  - DIY
categories:
  - 无人机
abbrlink: c10e5fcd
---

SpeedyBee 蓝牙调参模块虽好，于是我 DIY 了一个。

<!-- more -->

# 为什么要 DIY 这玩意

- iphone 没法通过 otg 连接飞控
- 别人家卖的稍微有点贵

# 原理

单片机作为 USB Host，与飞控通信。
单片机连接蓝牙串口模块，于手机通信。

# 实现方案

speedybee 使用 STM32 F105 + TI CC2541 蓝牙。
STM32 F105 无须额外芯片即可作为 USB Host。

emmm

由于没有 F105 的开发板，我便采用了 arduino uno。
结合 USB Host Shield 和 蓝牙串口模块。

USB Host Shield 官方购买连接 [http://shop.tkjelectronics.dk/product_info.php?products_id=43](http://shop.tkjelectronics.dk/product_info.php?products_id=43)，官方库（适用于 Arduino）[https://github.com/felis/USB_Host_Shield_2.0](https://github.com/felis/USB_Host_Shield_2.0)。该板子使用了 max3421e 芯片。

直接组装，下载完代码后再连接蓝牙串口模块至 uart 接口（uno 仅有一个 uart）。

# 代码

前提是安装 USB Host Shield 的库。

```c
#include <cdcacm.h>
#include <usbhub.h>

//#include "pgmstrings.h"

// Satisfy the IDE, which needs to see the include statment in the ino too.
#ifdef dobogusinclude
#include <spi4teensy3.h>
#endif
#include <SPI.h>

class ACMAsyncOper : public CDCAsyncOper
{
  public:
    uint8_t OnInit(ACM *pacm);
};

uint8_t ACMAsyncOper::OnInit(ACM *pacm)
{
  uint8_t rcode;
  // Set DTR = 1 RTS=1
  rcode = pacm->SetControlLineState(3);

  if (rcode)
  {
    ErrorMessage<uint8_t>(PSTR("SetControlLineState"), rcode);
    return rcode;
  }

  LINE_CODING  lc;
  lc.dwDTERate  = 115200;
  lc.bCharFormat  = 0;
  lc.bParityType  = 0;
  lc.bDataBits  = 8;

  rcode = pacm->SetLineCoding(&lc);

  if (rcode)
    ErrorMessage<uint8_t>(PSTR("SetLineCoding"), rcode);

  return rcode;
}

USB     Usb;
//USBHub     Hub(&Usb);
ACMAsyncOper  AsyncOper;
ACM           Acm(&Usb, &AsyncOper);

void setup()
{
  Serial.begin(115200);
#if !defined(__MIPSEL__)
  while (!Serial); // Wait for serial port to connect - used on Leonardo, Teensy and other boards with built-in USB CDC serial connection
#endif
  Serial.println("Start");

  if (Usb.Init() == -1)
    Serial.println("OSCOKIRQ failed to assert");

  delay( 200 );
}

uint8_t datas[64];

void loop() {
  Usb.Task();
  if ( Acm.isReady()) {
    uint8_t rcode;
    uint8_t send_count = 0;

    while (Serial.available() && send_count < 64) {
      datas[send_count] = Serial.read();
      send_count++;
    }

    if (send_count) {
      rcode = Acm.SndData(send_count, datas);
      if (rcode)
        ErrorMessage<uint8_t>(PSTR("SndData"), rcode);
    }

    delay(20);

    uint8_t  buf[64];
    uint16_t rcvd = 64;
    rcode = Acm.RcvData(&rcvd, buf);
    if (rcode && rcode != hrNAK)
      ErrorMessage<uint8_t>(PSTR("Ret"), rcode);

    if ( rcvd ) {
      Serial.write(buf, rcvd);
    }

    delay(20);
  }
}
```
