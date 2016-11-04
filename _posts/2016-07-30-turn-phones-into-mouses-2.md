---
layout: post
title: "把手机变成电脑的移动触控板 (二)"
date: 2016-07-30 22:54:06 +0800
comments: true
tags: [Android, Java, Socket]
description: 使用手机通过局域网控制电脑鼠标
comments: true
toc: true
---

在[把手机变成电脑的移动触控板 (一)](http://tangni.me/2016/07/turn-phones-into-mouses)里，介绍了手机通过socket 连接控制电脑的实现。那篇里写到了通过**组播**来使手机和电脑互相自动发现，本篇介绍另一种连接方式：通过手机扫描二维码的方式连接。

### 基本想法
因为上篇介绍过，手机要连接电脑，必须先知道电脑的ip，所以，这里的想法是：电脑端（server）程序获取自己的ip，把ip编码生成一个二维码并显示在屏幕上；手机端（client）扫描这个二维码得到电脑的ip，然后建立socket连接，剩余的控制部分同上篇。

### 实现
#### Server
server和client都是使用[** zxing **](https://github.com/zxing/zxing)的库。server端由java实现，因为上一篇介绍时，只有组播一种连接方式，现在新增一种二维码扫描的方式，所以定义了连接模式：

```java
    // 连接模式
    private enum Mode {
        DISCOVERY, // 自动发现模式
        QR_SCAN     // 扫描连接模式
    }
```
在server端程序线程的run方法其实时判断连接模式：

```java
    if (mMode == Mode.DISCOVERY) {
        try {
            // 自动发现模式，等待client发来组播
            waitForClientWave(MULTI_CAST_ADDR, LOCAL_UDP_PORT);
        } catch (IOException e1) {
            e1.printStackTrace();
        }
    } else {
        // 扫描模式，生成并显示二维码图案
        generateAndShowQRCode();
    }
```

二维码的生成也是使用** zxing **，参考[这个链接](http://www.journaldev.com/470/generate-qr-code-image-from-java-program)，稍作了修改。* createQRImage() *方法返回一个java.awt.image.BufferedImage 对象，用于界面显示。界面代码如下：

```java
package com.mobcontrol.server.ui;

import java.awt.image.BufferedImage;

import javax.swing.ImageIcon;
import javax.swing.JFrame;
import javax.swing.JLabel;
import javax.swing.JPanel;

public class UIFrame extends JFrame {

    /**
     * 
     */
    private static final long serialVersionUID = 1831203428568086441L;

    public void showImage(BufferedImage image) {
        if (image == null) {
            return;
        }
        this.setTitle("Mobile Control");
        this.setSize((int)(image.getWidth() * 1.5f), (int)(image.getHeight() * 1.5f));
        this.setDefaultCloseOperation(JFrame.HIDE_ON_CLOSE);

        JLabel jLabel = new JLabel(new ImageIcon(image));
        JPanel jPanel = new JPanel();
        jPanel.add(jLabel);
        this.add(jPanel);
        this.setVisible(true);
    }

}
```
到此，server已经完成了生成并显示二维码的工作。

#### Client
手机端使用了** journeyapps **的[这个库](https://github.com/journeyapps/zxing-android-embedded)，是在zxing库基础之上的一次封装，封装成一个Android library project，可以在Android studio里面作为module引用。
都是开源的代码，我只是拿过来用了，就不多介绍了。总之扫描二维码解析得到其中的ip后就可以创建socket连接电脑了。

完毕，谢谢观赏