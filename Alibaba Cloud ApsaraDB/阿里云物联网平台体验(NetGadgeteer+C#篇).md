# 阿里云物联网平台体验(NetGadgeteer+C#篇)
目前对接阿里云物联网平台有多种软件和硬件，可以有多种不同语言来实现对接，比如c/c++，Java，JS，Python，C#等等，不过C#版本只有PC对接云平台的代码，嵌入式的目前还没有看到，所以本篇文章是基于STM32F429芯片，采用C#语言对接阿里云物联网平台高级版。

下面是对接阿里云物联网平台的硬件，.Net Gadgeteer套件，有14个不同接口，可以对接近百种模块。 

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/平台体验.png" />
</div></br>

我们今天选用的是温湿度模块，LED模块，USB模块和主板模块，如下图所示：

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/平台体验1.png" />
</div></br>

1、 USB Device模块插入2#接口

2、 DHT11模块插入14#接口

3、 LED模块插入12#接口

4、 以太网模块插入4#接口

第一步：我们需要在阿里云物联网平台创建一个产品及对应设备

和阿里云官方示例不一样的是，我们额外增加了一个属性LED，具备读写能力，枚举型变量，0-表示关灯，1-表示开灯

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/平台体验5.png" />
</div></br>

这个定义好后，我们创建设备，并且获取设备的三元组。

第二步： 基于官方MQTT的C#代码库M2Mqtt.NetMf42嵌入式版本，实现Alink协议。

（1）   上传数据到云端


```js

byte[] bytData = new byte[4];

float T = 0;

float H = 0;

int ret = gs.IOControl((int)(6*16+11)); //PG11

if (ret != -1)

{

    bytData[0] = (byte)(ret & 0xFF);

    bytData[1] = (byte)(ret >> 8 & 0xFF);

    bytData[2] = (byte)(ret >> 16 & 0xFF);

    bytData[3] = (byte)(ret >> 24 & 0xFF);

 

    H = Byte2Float(bytData[0], bytData[1]);

    T = Byte2Float(bytData[2], bytData[3]);

    Debug.Print("H = " + H.ToString() + " T = " + T.ToString());

 

    string payload_json = "{" +

"\"id\": " + DateTime.Now.Ticks + "," +

"\"params\":{" +

    "\"temperature\":" + T + "," +

    "\"humidity\":" + H + "," +

"}," +

"\"method\":\"thing.event.property.post\"" +

"}";

    string Data = Guid.NewGuid().ToString();

    mqttClient.Publish(post_topic, Encoding.UTF8.GetBytes(payload_json), MqttMsgBase.QOS_LEVEL_AT_LEAST_ONCE, false);

Debug.Print(payload_json);

}

```
读取模块的温度T，和湿度H，然后推送到物联网平台。

（2）   下发控制命令到设备

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/平台体验2.png" />
</div></br>

云端单击“发送指令”，则设备的MqttMsgPublishReceived事件会接收到如下格式的数据：


```js
{"method":"thing.service.property.set","id":"196011725","params":{"LED":1},"version":"1.0.0"}
```

声明LED对象后，我们就可以根据这个信息来开关LED灯(如下)


```js
OutputPort led = new OutputPort((Cpu.Pin)(7*16+9),false);
```

然后在MqttMsgPublishReceived事件里做如下处理：


```js

string json = "";

if (e.Message.Length > 0)

{

    //{"method":"thing.service.property.set","id":"196011725","params":{"LED":1},"version":"1.0.0"}

    json = new string(System.Text.UTF8Encoding.UTF8.GetChars(e.Message));

    Debug.Print("Message:" + json);

 

    string strLED  =json.Substring(json.IndexOf("LED")+5,1);

    led.Write((strLED == "1"));

}

```

第三步：运行代码

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/平台体验3.png" />
</div></br>

运行后，打开阿里云物联网平台的网页，可以看到如下画面：

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/平台体验4.png" />
</div></br>

下发相关的指令，也会发现LED灯亮和灭。

本文相关的代码文件：<a href="https://pan.baidu.com/s/1PrgFq9HwiUhK-gtIRw7fKw?spm=a2c4e.11153940.blogcont680189.10.51f87565QWp05S">yfalink.rar</a>

 
