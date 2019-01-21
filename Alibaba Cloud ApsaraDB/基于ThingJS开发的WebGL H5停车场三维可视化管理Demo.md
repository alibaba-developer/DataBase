# 基于ThingJS开发的WebGL H5停车场三维可视化管理Demo

<h4>前言</h4>
随着社会的发展，城市中的汽车越来越多。车辆集中存放管理的场所被人类提出车辆进出的秩序、车辆存放的安全性、车辆存放管理的有偿性等要求。停车场系统应用现代机械电子及通讯科学技术，集控制硬件、软件于一体。随着科技的发展，停车场管理系统也日新月异，目前最为专业化的停车场系统为免取卡停车场。下面我们就用ThingJs平台来搭建一个3d可视化的停车场管理系统。

点击查看：<a href="http://www.thingjs.com/guide/sampleindex.html?spm=a2c4e.11153940.blogcont688168.15.6e806f7fxUKDjp&name=/uploads/wechat/oLX7p05lsWJZUIxnIWsNXAzJ40X8/%E5%81%9C%E8%BD%A6%E5%9C%BA%E7%AE%A1%E7%90%86.js">DEMO </a>

<h4>效果</h4>
停车场总览
<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/三维可视化管理Demo1.png" align="center" />
</div>
车辆信息
<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/三维可视化管理Demo2.png" align="center" />
</div>
车辆行动轨迹监控
<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/三维可视化管理Demo3.png" align="center" />
</div>
车位信息展示
<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/三维可视化管理Demo4.png" align="center" />
</div>
下面我们就用ThingJs平台来搭建一个3d可视化的停车场管理系统。

第一步
使用CampusBuilder来搭建一个模拟停车场。CampusBuider很好用在以往的文章中也多次提及过，丰富的模型库任你选择快速搭建3D场景。
<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/三维可视化管理Demo5.png" align="center" />
</div>

第二步
初始化摄像机的位置并添加鼠标滑过，左键单击，右键单击，左键双击等事件。鼠标滑过，车勾边变红色，车位勾边边蓝色。左键单击，车或车位弹出信息牌。右键单击，关闭当前信息牌，镜头初始化。getCarData() 与 getParkData() 为模拟数据，没有几个售出的车位和车就用了switch。


```js

app.on('load', function (evt) {

    //初始化摄像机
    init_camera();

    //滑过勾边
    var campus = evt.campus;
    var objs = app.query('.Building').add(campus.things);
    objs.on('mouseon', function (ev) {
        if (ev.object.name.search("car") == 0) {
            this.style.outlineColor = '#ff0000';
        }
        if (ev.object.name.search("park") == 0) {
            this.style.outlineColor = '#0000ff';
        }
    });
    objs.on('mouseoff', function () {
        this.style.outlineColor = null;
    });

    //单击事件
    app.on('click', function (ev) {
        if (ev.button == 2) {
            destroy_ui();
            init_camera();
        }
        if (ev.object.name.search("car") == 0) {
            destroy_ui();
            getCarData(ev.object);
            create_ui_car();
        }
        if (ev.object.name.search("park") == 0) {
            destroy_ui();
            getParkData(ev.object);
            create_ui_park();
        }
    });

    //双击事件
    app.on('dblclick', function (ev) {

        if (ev.object.name.search("car") == 0) {
            app.camera.flyTo({
                'time': 1500,
                'object': ev.object,
                'position': [0, 0, 0],
                'complete': function () {
                }
            });
        }
        if (ev.object.name.search("park") == 0) {
            app.camera.flyTo({
                'time': 1500,
                'object': ev.object,
                'position': [0, 5, 0],
                'complete': function () {
                }
            });
        }
    });
});

//初始化摄像机
function init_camera() {
    // 摄像机飞行到某位置
    app.camera.flyTo({
        'position': [-67.95670997548082, 49.69517426520041, -42.88366089402964],
        'target': [-7.188588318222256, 14.094194791658271, -12.724756207211417],
        'time': 800,
        'complete': function () {
            console.log("Camera ready");
        }
    });
}
//创建面板
var panel;
var dataObj;
var carInfo;
var parkInfo;

function create_ui_car() {
    panel = new THING.widget.Panel({
        titleText: "车辆信息",
        closeIcon: true, // 是否有关闭按钮
        dragable: true,
        retractable: true,
        opacity: 0.9,
        hasTitle: true,
        titleImage: 'https://www.thingjs.com/static/images/example/icon.png'
    });
    panel.position = [0, 326];
    // 创建任意对象
    dataObj = {
        name: carInfo[0],
        info: carInfo[1],
        park: carInfo[2],
        plateNum: carInfo[3],
        state: carInfo[4],
        contactNum: carInfo[5]
    };
    // 动态绑定物体
    var name = panel.addString(dataObj, 'name').caption('车主姓名');
    var info = panel.addString(dataObj, 'info').caption('车主信息');
    var park = panel.addString(dataObj, 'park').caption('车位编号');
    var plateNum = panel.addString(dataObj, 'plateNum').caption('车牌号码');
    var contactNum = panel.addString(dataObj, 'contactNum').caption('联系电话');
    var state = panel.addString(dataObj, 'state').caption('车位状态');

}

function create_ui_park() {
    panel = new THING.widget.Panel({
        titleText: "车位信息",
        closeIcon: true, // 是否有关闭按钮
        dragable: true,
        retractable: true,
        opacity: 0.9,
        hasTitle: true,
        titleImage: 'https://www.thingjs.com/static/images/example/icon.png'
    });
    panel.position = [0, 326];
    dataObj = {
        park: parkInfo[0],
        name: parkInfo[1],
        state: parkInfo[2],
        date: parkInfo[3]
    };
    var park = panel.addString(dataObj, 'park').caption('车位编号');
    var name = panel.addString(dataObj, 'name').caption('车主姓名');
    var state = panel.addString(dataObj, 'state').caption('车位状态');
    var date = panel.addString(dataObj, 'date').caption('车位期限');

}

function destroy_ui() {
    if (panel) {
        panel.destroy();
        panel = null;
    }
}

function getCarData(obj) {
    switch (obj.name) {
        case "car_0":
            carInfo = ['张三', '28#1-302', 'A-06', '吉K49278', '未交费', '13159828222'];
            break;
        case "car_1":
            carInfo = ['李四', '18#2-1202', 'B-04', '吉A46154', '已交费', '13159828222'];
            break;
        case "car_2":
            carInfo = ['王五', '13#2-702', 'B-05', '吉D95868', '已交费', '13159828222'];
            break;
        case "car_3":
            carInfo = ['郭富贵', '3#3-802', 'B-09', '吉B46278', '已交费', '13159828222'];
            break;
        case "car_4":
            carInfo = ['薛展畅', '8#3-1302', 'C-03', '吉A44278', '未交费', '13159828222'];
            break;
        case "car_5":
            carInfo = ['李文忠', '6#2-302', 'C-05', '黑B77865', '已交费', '13159828222'];
            break;
        case "car_6":
            carInfo = ['李洪春', '8#2-402', 'D-08', '吉CJ87821', '未交费', '13159828222'];
            break;
        case "car_7":
            carInfo = ['孟旭浩', '9#2-801', 'D-16', '吉A4U278', '已交费', '13159828222'];
            break;
        case "car_8":
            carInfo = ['刘星辰', '4#2-502', 'D-20', '吉A98378', '已交费', '13159828222'];
            break;
        case "car_9":
            carInfo = ['张星辰', '4#1-302', 'E-04', '吉A98378', '已交费', '13159828222'];
            break;
        case "car_10":
            carInfo = ['张星辰', '8#2-302', 'D-01', '京A44378', '已交费', '13159228222'];
            break;
    }
}

function getParkData(obj) {
    switch (obj.name) {
        case "park_5":
            parkInfo = ['A-06', '张三', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_11":
            parkInfo = ['B-09', '郭富贵', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_16":
            parkInfo = ['B-05', '王五', '欠费', '2018.5.10-2020.5.11'];
            break;
        case "park_17":
            parkInfo = ['B-04', '李四', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_40":
            parkInfo = ['C-03', '薛展畅', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_44":
            parkInfo = ['C-05', '李文忠', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_68":
            parkInfo = ['D-08', '李洪春', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_78":
            parkInfo = ['E-04', '张星辰', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_59":
            parkInfo = ['D-16', '孟旭浩', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_67":
            parkInfo = ['D-20', '刘星辰', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_3":
            parkInfo = ['A-04', '刘地辰', '已交', '2018.5.10-2020.5.11'];
            break;
        case "park_54":
            parkInfo = ['D-1', '龙的辰', '未交', '2018.5.10-2020.5.11'];
            break;
        default:
            parkInfo = ['X-xx', 'XXX', '未售出', '2000.1.1-2020.1.1'];

    }
}
```

第三步
创建主面板添加空间统计，闸门管理，播放动画，出入登记等功能按钮，同时创建闸门管理子面板。


```js
//主面板
var toolbar = new THING.widget.Panel({ width: '163px' });
var mainDataObj = {
    spaceStatistics: false,
    gateManagement: false,
    video: false,
    registrationForm: false
}

//闸门管理面板
var gateToolbar = new THING.widget.Panel({ width: '163px' });
gateToolbar.position = [450, 0];
gateToolbar.visible = false;
var gateDataObj = {
    entrance: false,
    exit: false,
}

//面板按钮组件及事件
Loader.sync(['lib/iconfont.js'], function () {
    //主面板
    var button0 = toolbar.addImageBoolean(mainDataObj, 'spaceStatistics').caption('空间统计').url('#momoda_lc-icontubiao');
    var button1 = toolbar.addImageBoolean(mainDataObj, 'gateManagement').caption('闸门管理').url('#momoda_lc-icontubiao21');
    var button2 = toolbar.addImageBoolean(mainDataObj, 'video').caption('播放动画').url('#momoda_lc-icontubiao9');
    var button3 = toolbar.addImageBoolean(mainDataObj, 'registrationForm').caption('出入登记').url('#momoda_lc-icontubiao10');
    //闸门面板
    var button4 = gateToolbar.addImageBoolean(gateDataObj, 'entrance').caption('入口管理').url('#momoda_lc-icontubiao21');
    var button5 = gateToolbar.addImageBoolean(gateDataObj, 'exit').caption('出口管理').url('#momoda_lc-icontubiao21');
	
	//第四步中的功能实现
});
```

第四步
为上面创建的功能按钮实现功能。

```js

//空间统计
    var opacityFlag = true;
    button0.on('change', function () {
        if (opacityFlag) {
            opacityFlag = false;
            app.query(/park/).forEach(
                function (obj) {
                    var str = obj.name;
                    switch (str) {
                        case "park_5": break;
                        case "park_11": break;
                        case "park_16": break;
                        case "park_17": break;
                        case "park_40": break;
                        case "park_44": break;
                        case "park_68": break;
                        case "park_78": break;
                        case "park_59": break;
                        case "park_67": break;
                        case "park_33": break;
                        case "park_54": break;
                        case "park_3": break;
                        default:
                            obj.style.opacity = 0.3;
                    }
                }
            );
        } else {
            opacityFlag = true;
            app.query(/park/).forEach(
                function (obj) {
                    obj.style.opacity = 1;
                }
            )
        }
    });

    //闸门管理,入口管理，出口管理
    var gateToolbarFlag = true;
    var entranceFlag = false;
    var exitFlag = false;
    button1.on('change', function () {
        if (gateToolbarFlag) {
            app.camera.flyTo({
                'position': [-69.15232764795844, 12.556743445078443, -4.722896106654333],
                'target': [-6.75806618043438, 11.584727439263146, -5.077821719000649],
                'time': 1000
            });
            gateToolbarFlag = false;
            gateToolbar.visible = true;
        } else {
            init_camera();
            gateToolbarFlag = true;
            gateToolbar.visible = false;
        }


    });
    button4.on('change', function () {
        var entry = app.query('入口')[0];
        if (!entranceFlag) {
            entranceFlag = true;
            entry.rotateX(45.0);
            entry.moveY(2);
            entry.moveZ(-1);
        } else {
            entranceFlag = false;
            entry.rotateX(-45.0);
            entry.position = [0, 0, 0];
        }
    });
    button5.on('change', function () {
        var exit = app.query('出口')[0];

        if (!exitFlag) {
            exitFlag = true;
            exit.rotateX(-45.0);
            exit.moveY(9.2);
            exit.moveZ(4.3);
        } else {
            exitFlag = false;
            exit.rotateX(-315.0);
            exit.position = [0, 0, 0];
        }
    });

    //播放动画
    button2.on('change', function () {
        //飞向每一个摄像机的位置
        console.log("监控设备！");
        playCar();
    });

    //出入登记
    registrationFlag = true;
    button3.on('change', function () {
        //显示两块信息板，镜头飞向门禁
        // 摄像机飞行到某位置
        if (registrationFlag) {
            app.camera.flyTo({
                'position': [-13.229586070519874, 13.062016938601909, -14.789241424512456],
                'target': [-21.25078065116403, 11.949594230222267, -11.972835509196605],
                'time': 1000,
            });
            registrationFlag = false;
            create_ui_gate_exit();
            create_ui_gate_entry();
        } else {
            registrationFlag = true;
            entryUi.destroy();
            entryUi = null;
            exitUi.destroy();
            exitUi = null;
        }
    });
    
 ```
 
播放动画

```js
var car = app.create({
    type: 'Thing',
    name: 'car_10',
    url: 'http://model.3dmomoda.cn/models/c6ed424627234a298c1921950eb8534c/0/gltf/', // 模型地址 
    position: [-45.89714816093272, 0.043936770289323, 0.312388718621647], // 位置 
    angle: 90,
});
var points = [];
points.push([-45.89714816093272, 0.043936770289323, 0.312388718621647]);
points.push([-38.89714816093272, 0.043936770289323, 0.312388718621647]);
var radius = 2
for (var degree = 0, y = 0; degree <= 90; degree += 20) {
    var x = Math.sin(degree * 2 * Math.PI / 360) * radius - 35.89714816093272;
    var z = -Math.cos(degree * 2 * Math.PI / 360) * radius + 2.312388718621647;
    points.push([x, y, z]);
    console.log([x, y, z]);
}
points.push([-33.927532654908305, 0, 4.9650923632877861]);
points.push([-33.927532654908305, 0, 7.9650923632877861]);
points.push([-33.927532654908305, 0, 10.9650923632877861]);
points.push([-33.927532654908305, 0, 13.9650923632877861]);
var line = app.create({
    type: 'Line',
    color: 0xFFFF00, // 轨迹线颜色
    dotSize: 2, // 轨迹点的大小
    points: points,
});
line.visible = false;
function playCar() {
    var car = app.query('car_10')[0];
    var entry = app.query('入口')[0];
    entry.rotateX(45.0);
    entry.moveY(2);
    entry.moveZ(-1);
    car.movePath({
        'path': line.points, // 轨迹路线
        'time': 5000, // 移动时间
        'orientToPath': true, // 物体移动时沿向路径方向
    });
    setTimeout(function () {
        entry.rotateX(-45.0);
        entry.position = [0, 0, 0];
    }, 2000)
}　
```

出入登记

```js
//出入登记
function create_html_entry() {
    var sign1 =
        `<div class="sign1" id="board1" style="font-size: 12px;width: 230px;text-align: center;background-color: rgba(0, 0, 0, .6);border: 3px solid #eeeeee;border-radius: 8px;color: #eee;position: absolute;top: 0;left: 0;z-index: 10;display: none;">
			<div class="s1" style="margin: 5px 0px 5px 0px;line-height: 32px;overflow: hidden;">
				<span class="span-l icon" style="float: left;width: 30px;height: 30px;"></span>
				<span class="span-l font" style="float: left;margin: 0px 0px 0px 3px;">车辆进入</span>
				<span class="span-r point" style="float: right;width: 12px;height: 12px;background-color: #18EB20;border-radius: 50%;margin: 10px 5px 10px 0px;"></span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;width:70px;margin: 0px 10px 0px 10px;">进车时间</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">车牌号</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">9:15</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉K49278</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">10:15</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉A46154</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">10:17</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉D95868</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">10:25</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉B46278</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">10:39</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">黑B77865</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">11:19</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉CJ87821</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">11:21</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉A4U278</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">11:35</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉A98378</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">12:50</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉A98778</span>
			</div>
			<div class="point-top" style="position: absolute;top: -7px;right: -7px;background-color: #3F6781;width: 10px;height: 10px;border: 3px solid #eee;border-radius: 50%;"></div>
		</div>`
    $('#div3d').append($(sign1));
}
function create_html_exit() {
    var sign2 =
        `<div class="sign2" id="board2" style="font-size: 12px;width: 230px;text-align: center;background-color: rgba(0, 0, 0, .6);border: 3px solid #eeeeee;border-radius: 8px;color: #eee;position: absolute;top: 0;left: 0;z-index: 10;display: none;">
			<div class="s1" style="margin: 5px 0px 5px 0px;line-height: 32px;overflow: hidden;">
				<span class="span-l icon" style="float: left;width: 30px;height: 30px;"></span>
				<span class="span-l font" style="float: left;margin: 0px 0px 0px 3px;">车辆进入</span>
				<span class="span-r point" style="float: right;width: 12px;height: 12px;background-color: #18EB20;border-radius: 50%;margin: 10px 5px 10px 0px;"></span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;width:70px;margin: 0px 10px 0px 10px;">出车时间</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">车牌号</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">7:15</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">黑B77865</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">8:45</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉A4U278</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">8:57</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉A98378</span>
			</div>
			<div class="s2" style="margin: 5px 0px 10px 0px;line-height: 18px;font-size: 10px;overflow: hidden;">
				<span class="span-l font1" style="float: left;margin: 0px 10px 0px 10px;">10:01</span>
				<span class="span-l font2" style="float: left;width: 140px;background-color: #2480E3;">吉A98778</span>
			</div>
			<div class="point-top" style="position: absolute;top: -7px;right: -7px;background-color: #3F6781;width: 10px;height: 10px;border: 3px solid #eee;border-radius: 50%;"></div>
		</div>`
    $('#div3d').append($(sign2));
}
function create_element(str) {
    var srcElem = document.getElementById(str);
    var newElem = srcElem.cloneNode(true);
    newElem.style.display = "block";
    app.domElement.insertBefore(newElem, srcElem);
    return newElem;
}
var entryUi = null;
function create_ui_gate_entry() {
    create_html_entry();
    entryUi = app.create({
        type: 'UIAnchor',
        position: [-39.89714816093272, 3.043936770289323, 2.312388718621647],
        element: create_element("board1"),
        offset: [0, 2, 0],
        pivot: [0.5, 1] // 界面的重心
    });
}
var exitUi = null;
function create_ui_gate_exit() {
    create_html_exit();
    exitUi = app.create({
        type: 'UIAnchor',
        position: [-34.89714816093272, 6.059100472147456, -14.950719696627075],
        element: create_element("board2"),
        offset: [0, 2, 0],
        pivot: [0.5, 1] // 界面的重心
    });
}

```
