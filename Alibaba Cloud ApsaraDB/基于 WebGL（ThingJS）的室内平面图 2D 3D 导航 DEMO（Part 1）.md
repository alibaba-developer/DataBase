#  基于 WebGL（ThingJS）的室内平面图 2D/3D 导航 DEMO（Part 1）

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/室内平面图1.png" align="center" />
</div>
<h4>前言</h4>
利用CampusBuilder来搭建自己的虚拟世界过程有这样一个问题:如何快速聚焦到虚拟场景的某一位置。当然我们可以创建几个按钮对应查找我们需要去的位置（参照物）并聚焦，但是按钮并不是很炫酷也不能很好的反馈给我们一些信息。接下来我们就用平面导航图来解决这一问题。



<h4>实现</h4>
第一步，使用CampusBuilder搭建模拟场景，CampusBuilder操作简单，分分钟就可以上手。这里为每一个房间都创建一个小球作为视点参照物体并勾选预览时隐藏，这样不会对我们的场景造成影响，也便于我们聚焦到指定房间。注意：要将我们每个房间中的设备框选之后组合在一起，为下一阶段的做准备。




 <div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/室内平面图2.png" align="center" />
</div>


第二步，把我们编辑好的场景加载到ThingJS中。


```js
//加载场景代码
var app = new THING.App({
	// 场景地址
	"url": "http://www.thingjs.com/./uploads/wechat/S2Vyd2lu/scene/Campus04",
});
//场景相关
//************************************************************************************/
app.on('load', function () {
    app.camera.flyTo({
        'position': [36.357131498969785, 61.953024217074265, 69.12160670337104],
        'target': [-1.3316924326803257, -4.9370371421622625, 33.619521849828544],
        'time': 2000,
    });
});
```



第三步，为平面图创建一块面板，并调整一下面板的位置以及大小。
图片下载地址：
链接：https://pan.baidu.com/s/1gmNjIj2ekbw1rO3MoujHqQ 提取码：i0c1


```js
//面板相关
//************************************************************************************/
var panel = new THING.widget.Panel({
    closeIcon: false,
    dragable: false, 
    retractable: true,
    opacity: 0.9,
    hasTitle: true,
});
panel.width = 600;
panel.position = [0, 200];
var dataObj = {
    iframe: ''
};
var iframe = panel.addIframe(dataObj, 'iframe').caption('').setHeight("290px");
```

第四步，编写iframe页。写完记得将这个页面和图片上传到页面资源，资源 => 页面资源 => 按钮（上传） 。

```js
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
    <style>
      	.total_image {
        	margin : 20px;
        }
        .total_image img{
            cursor: pointer;
            transition: all 0.6s;
            width: 50px;
        }
        .total_image img:hover{
            transform: scale(1.5);
            position:relative;
            z-index:100;
        }
    </style>
</head>
<body>
<div class="total_image" style="width: 500px;height: 280px;background-size: 100% auto">
    <img class="model_imag" src="发电室1.jpg" style="float: left;display: block;width: 85px;height: 84px" 
			onclick="onClick('PowerGenerationGroup_01','viewPoint_1')" >
              
    <img class="model_imag" src="发电室2.jpg" style="float: left;display: block;width: 78px;height: 84px" 
			onclick="onClick('PowerGenerationGroup_02','viewPoint_2')" >
              
    <img class="model_imag" src="发电室3.jpg" style="float: left;display: block;width:170px;height: 84px" 
			onclick="onClick('PowerGenerationGroup_03','viewPoint_3')" >
              
    <img class="model_imag" src="发电室4.jpg" style="float: left;display: block;width:167px;height: 84px" 
			onclick="onClick('PowerGenerationGroup_04','viewPoint_4')" >
              
    <div style="display: block;float: left;width: 100px;height: 145px;background-color:white">
        <img class="model_imag" src="办公室1.jpg" style="float: left;display: block;width:100px;height: 60px" 
			onclick="onClick('Office','viewPoint_5')" >
        <img class="model_imag" src="返回.png" style="float: left;display: block;width:100px;height: 80px" onclick="initViewPoint()">
    </div>

    <img class="model_imag" src="发电室5.jpg" style="float: right;display: block;width:123px" 
			onclick="onClick('PowerGenerationGroup_05','viewPoint_8')" > 
              
    <img class="model_imag" src="会议室1.jpg" style="float: left;display: block;width: 138px;height: 145px"  alt="" 
			onclick="onClick('BoardRoom_01','viewPoint_6')">
              
    <img class="model_imag" src="会议室2.jpg" style="float: left;display: block;width: 138px;height: 145px"  alt="" 
			onclick="onClick('BoardRoom_02','viewPoint_7')" > 
</div>

<script>
    function onClick(viewPoint,target){
        window.parent.onClick(viewPoint,target);
    }
	function initViewPoint(){
        window.parent.initViewPoint();
    }
</script>
</body>
</html>
```


第五步，完成onClick()和initViewPoint()方法。

```js
//事件相关
//************************************************************************************/
var currentModule = null;

//点击事件
function onClick(targetObj, viewPoint) {
    currentModule = app.query(targetObj)[0];
    currentModule.position = [0, 0, 0];
    currentModule.style.opacity = 1;
    app.camera.flyTo({
        'object': app.query(viewPoint)[0],
        'offset': [0, 13, 7],
        'time': 1000,
        complete: function () {
            currentModule.brothers.style.opacity = 0.3;
        }
    });
}
//返回事件
function initViewPoint() {
    currentModule.brothers.style.opacity = 1;
    currentModule = null;
    app.camera.flyTo({
        'position': [36.357131498969785, 61.953024217074265, 69.12160670337104],
        'target': [-1.3316924326803257, -4.9370371421622625, 33.619521849828544],
        'time': 1000,
    });
}
```

<h4>小结</h4>
第一部分我们主要完成了iframe与我们的3D场景的简单交互，这里也没有做什么特效只是做了一个点击事件。这里值得一提的是currentModule这个全局变量，开始我没有创建这个变量只是将我当前点击的物体obj.style.opacity = 1;obj.brothers.style.opacity = 0.3, 但是执行initViewPoint(){app.query(’.Thing’).style.opacity=1}无法将场景的opacity 属性还原（自己可以试一下，或者有解决方案留言）。第二部分我会给iframe页加上鼠标悬停事件让iframe页的img标签和我们场景中的obj一起动起来！


<h4>完整代码</h4>
可以粘到 ThingJS 网站在线开发环境运行http://www.thingjs.com/guide/?m=sample

```js
	//加载场景代码
var app = new THING.App({
	// 场景地址
	"url": "http://www.thingjs.com/./uploads/wechat/S2Vyd2lu/scene/Campus04",
});
//场景相关
//************************************************************************************/
app.on('load', function () {
    app.camera.flyTo({
        'position': [36.357131498969785, 61.953024217074265, 69.12160670337104],
        'target': [-1.3316924326803257, -4.9370371421622625, 33.619521849828544],
        'time': 2000,
    });
});
//面板相关
//************************************************************************************/
var panel = new THING.widget.Panel({
    closeIcon: false,
    dragable: false, 
    retractable: true,
    opacity: 0.9,
    hasTitle: true,
});
panel.width = 600;
panel.position = [0, 200];
var dataObj = {
    iframe: '/uploads/wechat/S2Vyd2lu/file/平面图导航/ifram.html'
};
var iframe = panel.addIframe(dataObj, 'iframe').caption('').setHeight("290px");

//事件相关
//************************************************************************************/
var currentModule = null;

//点击事件
function onClick(targetObj, viewPoint) {
    currentModule = app.query(targetObj)[0];
    currentModule.position = [0, 0, 0];
    currentModule.style.opacity = 1;
    app.camera.flyTo({
        'object': app.query(viewPoint)[0],
        'offset': [0, 13, 7],
        'time': 1000,
        complete: function () {
            currentModule.brothers.style.opacity = 0.3;
        }
    });
}
//返回事件
function initViewPoint() {
    currentModule.brothers.style.opacity = 1;
    currentModule = null;
    app.camera.flyTo({
        'position': [36.357131498969785, 61.953024217074265, 69.12160670337104],
        'target': [-1.3316924326803257, -4.9370371421622625, 33.619521849828544],
        'time': 1000,
    });
}
```
 
