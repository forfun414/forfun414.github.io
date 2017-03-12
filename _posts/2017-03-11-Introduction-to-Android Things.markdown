---
layout: post
title: Introduction to Android Things
date:   2017-03-011 21:00:08 +0800
categories: jekyll update
---

# Introduction to Android Things/Android Things入门

Android Things是Google为IoT专门打造的操作系统，来源于Android，所以具备了Android的大部分特性。
但是IoT不需要像手机或者平板一样支持多应用切换及消息通知等特性，所以Android Things相比Android更为简单。
传统Android开发其实涉及两部分：OS开发和APP开发，而Android Things基本上侧重在APP开发，OS的开发由Google完成了，并直接提供针对开发板的镜像。

这种方式为整个系统的兼容性带来了非常大的好处：

不会由于第三方OS的开发带来稳定性和兼容性问题；
同时OS镜像可以通过OTA完成升级，大大减少了系统碎片化的可能以及安全性的问题。这可以从Android亲儿子Nexus手机一直都能体验最新版Android系统可见一斑。

不过这也带来了一些不足：

开发依赖于开发板，但是目前支持的开发板有限，现在有Intel Edison，Intel Joule，NXP Pico i.MX6UL，以及Raspberry Pi 3；
不完全开源就意味着不能随意定制，特别是对于硬件有特殊定制需求的应用场景。


本文主要是针对https://developer.android.com/things 的总结和提炼，旨在帮助读者全面的了解Android Things。
要求读者对Android有基本的了解，文中代码经过精简，意在概括性的说明相关内容。

主要包括：
## 相对于Android的主要变化
## Peripheral I/O API
## User Driver API
最后这两部分API是Android Things开发中最重要的两个部分。


#1 相对于Android的主要变化
## 显示为可选项
Iot设备可能不一定需要像Android设备那样需要显示界面，所以显示是可选项。所有的消息和事件都仍然会发送给前台应用。

## 不包括状态栏和导航键
Iot的交互相对简单，所以Android Things去掉了状态栏和导航键，这样前台应用可以占用全部的交互界面。

## 不包括notifiations
由于没有状态栏，所以也没有消息通知。

## 桌面应用设置方式调整
桌面应用同样接收action.MAIN，但是filteer不一样，并且需要包括category:CATEGORY_DEFAULT and IOT_LAUNCHER.,
不过为了方便Android Studio调试，要额外增加原Androi的filter。

```
<application
    android:label="@string/app_name">
    <activity android:name=".HomeActivity">
        <!-- Launch activity as default from Android Studio -->
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>

        <!-- Launch activity automatically on boot -->
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.IOT_LAUNCHER"/>
            <category android:name="android.intent.category.DEFAULT"/>
        </intent-filter>
    </activity>
</application>
```

## 不包括RuntimePermission
应用安装时就获取需要的各种权限。

#2 Peripheral I/O API
用来控制io设备，包括GPIO, PWM, I2C, SPI, UART等。主要是通过PeripheralManagerService的各个接口完成。

##1 获取IO列表

```
getGpioList();
getPwmList();
getI2CBusList();
getSpiBusList();
  //片选信号CS0, CS1, and CS2将以"SPI0.0", "SPI0.1", and "SPI0.2"形式返回
getUartDeviceList();
```

##2 接口打开和设置
```
PeripheralManagerService manager = new PeripheralManagerService();
mGpio = manager.openGpio(GPIO_NAME);
mGpio.setDirection(Gpio.DIRECTION_IN);
mGpio.setActiveType(Gpio.ACTIVE_HIGH);
```

##3 接口使用
```
mGpio.getValue()
mGpio.setValue(true); 
```

##4 回调方式
回调是和直接使用相并列的一种使用方式。

```
mGpio.setDirection(Gpio.DIRECTION_IN);
mGpio.setActiveType(Gpio.ACTIVE_LOW);
mGpio.setEdgeTriggerType(Gpio.EDGE_BOTH);
mGpio.registerGpioCallback(mGpioCallback);

private GpioCallback mGpioCallback = new GpioCallback() {
    @Override
    public boolean onGpioEdge(Gpio gpio) {
        gpio.getValue())

        // Continue listening for more interrupts
        return true;
    }
```


# 3 User Driver API
User Driver API为用户空间提供硬件消息注入的接口，从而可以将硬件事件按Android中已有的消息类型被其他应用处理。
说人话就是，它是用户空间的驱动开发API，而你需要做的就是利用这些API开发出用户空间的驱动程序，从而提供标准的input，gps及sensor消息事件。

##1 key input按键输入
注册
```
mButtonInputDriver = new ButtonInputDriver(
                    GPIO_PIN_NAME,
                    Button.LogicState.PRESSED_WHEN_LOW,
                    KeyEvent.KEYCODE_SPACE);
                    

mButtonInputDriver.register();
```

驱动处理和封装
keyinput太简单，驱动不需要额外处理，这部分省略

将GPIO端口作为空格键注册，当硬件件状态改变时，发送空格这一按键事件，从而可以以Android标准input事件上报。       


##2 motion input滑动输入
注册
```
InputDriver mDriver = InputDriver.builder(InputDevice.SOURCE_TOUCHPAD)
        .setName(DRIVER_NAME)
        .setVersion(DRIVER_VERSION)
        .setAbsMax(MotionEvent.AXIS_X, 255)
        .setAbsMax(MotionEvent.AXIS_Y, 255)
        .build();

UserDriverManager manager = UserDriverManager.getManager();
manager.registerInputDriver(mDriver);
```

驱动处理和封装
```
mDriver.emit(x, y, pressed)
```
触摸事件x y等坐标上报后，通过emit接口完成事件注入，从而可以以Android标准motion event事件上报。


##3 GPS
注册
```
GpsDriver mDriver = new GpsDriver();
UserDriverManager manager = UserDriverManager.getManager();
manager.registerGpsDriver(mDriver);
```

驱动处理和封装
```
Location location = parseLocationFromString(rawGpsData);
mDriver.reportLocation(location);
```
收到原始的gps数据，经过封装和处理，通过reportLocation接口完成事件注入，从而可以以Android gps框架事件上报。

##4 sensor
初始化userdriver
```
UserSensorDriver mDriver = new UserSensorDriver() {
    // Sensor data values
    float x, y, z;

    @Override
    public UserSensorReading read() {
        return new UserSensorReading(new float[]{x, y, z});
    }
};
```

注册
```
UserSensor accelerometer = UserSensor.builder()
        .setName("GroveAccelerometer")
        .setVendor("Seeed")
        .setType(Sensor.TYPE_ACCELEROMETER)
        .setDriver(mDriver)
        .build();
```

非默认传感器的注册稍微有些不一样
```
UserSensor custom = UserSensor.builder()
        .setName("MySensor")
        .setVendor("MyCompany")
        .setCustomType(Sensor.TYPE_DEVICE_PRIVATE_BASE,
                "com.example.mysensor",
                Sensor.REPORTING_MODE_CONTINUOUS)
        .setDriver(mDriver)
        .build();
```
        

驱动处理和封装
```
    public UserSensorReading read() {
        return new UserSensorReading(new float[]{x, y, z});
    }
```

其实主要封装是在UserSensorReading里面实现，它是在userdriver初始化部分定义的。这里的read是以应用程序为视角的，非驱动视角。
每当应用程序发起读sensor请求时，Android Things框架就会调用到用户空间的这个read函数，这里再从真正的硬件去获取实际的外部数据。

## 这里需要特别说明的是，它不仅可以用来处理Android已支持的传感器，也可以处理IoT实际使用的新类型传感器，如血压计等，这也是userspacedriver最重要的意义，通过这些接口完成Android Things对新型传感器的快速兼容。
当然这里面最关键的是driver怎么实现，这部分依赖Peripheral Driver Library，可以参考https://github.com/androidthings/contrib-drivers

更多开源代码参考https://github.com/androidthings
