## Bindings
Bindings是整个Flutter的基石，为Flutter提供底层基础功能
![enter image description here](https://raw.githubusercontent.com/Ryan-Hu/LearnFlutter/master/images/binding-overview.svg)

* `ServicesBinding`：负责Flutter与原生端的通信（`BinaryMessenger`）
* `SchedulerBinding`：驱动整个Build/Layout/Paint Pipeline的运行
* `RendererBinding`：负责Layout/Paint流程
* `GestureBinding`：负责原生端手势的处理
* `SemanticsBinding`：负责Flutter辅助功能（类似安卓端的Accessibility机制）
* `WidgetsBinding`：集成了以上5种binding，并负责Build流程

## ServicesBinding
`ServicesBinding`通过`BinaryMessenger`机制负责所有原生与Flutter的数据通信，其中初始化了一个`_defaultBinaryMessenger`对象，其默认实现是`_DefaultBinaryMessenger`类，这个类负责：
* 接收从`ui.Window.onPlatformMessage`传过来的消息，并分发给注册的`MessageHandler`
* 调用`ui.Window.onPlatformMessage`把Flutter端的消息发给原生

![enter image description here](https://raw.githubusercontent.com/Ryan-Hu/LearnFlutter/master/images/binding-services.svg)

## SchedulerBinding
`SchedulerBinding`是整个Flutter应用的发动机，他负责驱动Flutter应用的Build/Layout/Paint过程，`SchedulerBinding`把Dart的Event Loop分成了5个阶段：
* idle
* transientCallbacks
* midFrameMicrotasks
* persistentCallbacks
* postFrameCallbacks

![enter image description here](https://raw.githubusercontent.com/Ryan-Hu/LearnFlutter/master/images/binding-scheduler.svg)

## RendererBinding
`RendererBinding`负责整个测量（Layout）和绘制（Paint）流程，
`RendererBinding`通过调用`SchedulerBinding.addPersistentCallback()`将自身的`_handlePersistentFrameCallback`方法注册为在**persistentCallback**阶段执行:
```dart
void _handlePersistentFrameCallback(Duration timeStamp) {  
  drawFrame();  
  //...
}

/// 这里实现了每一帧的绘制
void drawFrame() {  
  //...
  pipelineOwner.flushLayout();  
  pipelineOwner.flushCompositingBits();  
  pipelineOwner.flushPaint();  
 if (sendFramesToEngine) {  
    renderView.compositeFrame(); // this sends the bits to the GPU  
  pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.  
  //...
  }  
}
```

#### PipelineOwner
`PipelineOwner`是Layout和Paint的入口，管理整棵`RenderObject`树，树的根节点是一个特殊的`RenderObject`对象 -- `RenderView`
![enter image description here](https://raw.githubusercontent.com/Ryan-Hu/LearnFlutter/master/images/binding-rendering-pipeline.svg)
`PipelineOwner`内部维护了4个`List<RenderObject>`来存储被标记为dirty的`RenderObject`对象：

* **_nodeNeedsLayout**：这个列表保存了需要重新Layout的`RenderObject`，当`RenderObject.markNeedsLayout()`方法被调用时，如果当前`RenderObject`是`relayoutBoundary`，则将自己加入` _nodeNeedsLayout`，否则将离自己最近的`relayoutBoundary`祖先加入` _nodeNeedsLayout`

```dart
abstract class RenderObject {

	//...

	void markNeedsLayout() {  
		if (_relayoutBoundary != this) {
			//如果自己不是relayoutBoundary则让父节点markNeedsLayout  
			markParentNeedsLayout();  
		} else {  
			//把自己标记为需要layout
			_needsLayout = true;  
			if (owner != null) {//这里的owner就是PipelineOwner
				owner._nodesNeedingLayout.add(this);  
				owner.requestVisualUpdate();  
			}  
		}  
	}

	//...
}
```

在`PipelineOwner.flushLayout()`阶段，`_nodeNeedsLayout`里的`RenderObject`根据在树中的深度排序后依次调用每个元素的`_layoutWithoutResize()`方法，来完成**Layout**过程
```dart
class PipelineOwner {

	//...

	void flushLayout() {
		//...
		while (_nodesNeedingLayout.isNotEmpty) {  
		
			final List<RenderObject> dirtyNodes = _nodesNeedingLayout;  
			_nodesNeedingLayout = <RenderObject>[];  
			//对dirtyNodes排序后遍历
			for (final RenderObject node in 
				dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {  
				
				if (node._needsLayout && node.owner == this) {
					//调用RenderObject._layoutWithoutResize()进行layout
					node._layoutWithoutResize();  
				}
			}  
		}
		//...
	}

	//...
}
```

* **_nodeNeedsCompositingBitsUpdate**：这个列表里保存的是需要更新Compositing标志位的`RenderObject`，当`RenderObject.markNeedsCompositingBitsUpdate()`方法被调用时，将自己加入`_nodeNeedsCompositingBitsUpdate`，同时如果父节点和自己都不是repaintBoundary，则将父节点也加入`_nodeNeedsCompositingBitsUpdate`

```dart
abstract class RenderObject {

	//...

	void markNeedsCompositingBitsUpdate() {
	
		if (_needsCompositingBitsUpdate)  
		  return;
		  
		_needsCompositingBitsUpdate = true;  
		
		if (parent is RenderObject) {  
			final RenderObject parent = this.parent as RenderObject;  
			if (parent._needsCompositingBitsUpdate)  
			    return;  
			if (!isRepaintBoundary && !parent.isRepaintBoundary) {  
				//如果自己不是repaintBoundary并且parent不是repaintBoundary则标记parent
			    parent.markNeedsCompositingBitsUpdate();  
				return;  
			}  
		}
		
		// parent is fine (or there isn't one), but we are dirty  
		if (owner != null)  {
			owner._nodesNeedingCompositingBitsUpdate.add(this);
		}
	}

	//...
}
```

在`PipelineOwner.flushCompositingBits()`阶段，`_nodesNeedingCompositingBitsUpdate`里的`RenderObject`根据在树中的深度排序后依次调用每个元素的`_updateCompositingBits()`方法，来完成compositing标识位的更新过程，之后就进入到**Paint**阶段了

```dart
class PipelineOwner {

	//...

	void flushCompositingBits() {  
	
		//深度排序
		_nodesNeedingCompositingBitsUpdate.sort(
		(RenderObject a, RenderObject b) => a.depth - b.depth);  
		
		for (final RenderObject node in _nodesNeedingCompositingBitsUpdate) {  
			if (node._needsCompositingBitsUpdate && node.owner == this) {
				//调用RenderObject._updateCompositingBits()进行compositing标识位的更新
				node._updateCompositingBits(); 
			}
		}  
		_nodesNeedingCompositingBitsUpdate.clear();  
	}
	
	//...
}
```

> 关于compositing：
> RenderObject中有一个属性needsCompositing，用来表示该节点或其子节点在绘制(paint)sh
