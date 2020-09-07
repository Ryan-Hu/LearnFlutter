
## ui.Window
`ui.Window`是Flutter的Dart部分和Native Engine之间的沟通的桥梁
`ui.Window`提供了各种回调供dart端监听native端的事件，并且提供给dart端向native发送信息的接口，各种binding类建立在`ui.Window`提供的基础功能之上，为为上层的Flutter模块提供服务

![enter image description here](https://raw.githubusercontent.com/Ryan-Hu/LearnFlutter/master/images/window-overview.svg)

`ui.Window`主要由以下方法：

* 界面设置：
	 - onMetricsChanged
	 - onLocaleChanged
	 - onTextScaleFactorChanged
	 - onPlatformBrightnessChanged
	 
* 绘制渲染：
	- scheduleFrame
	- onBeginFrame
	- onDrawFrame
	- render

* 触摸事件：
	- onPointerDataPacket

* 复制功能：
	- onSemanticsEnabledChanged
	- onSemanticsAction
	- onAccessibilityFeaturesChanged
	- updateSemantics

* 原生通信：
	- onPlatformMessage
	- sendPlatformMessage

* 性能监控
	- onReportTimings
