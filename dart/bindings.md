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
      for (final RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {  
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

* **_nodeNeedsCompositingBitsUpdate**：这个列表里保存的是需要更新**compositing**标识的`RenderObject`，当`RenderObject.markNeedsCompositingBitsUpdate()`方法被调用时，将自己加入`_nodeNeedsCompositingBitsUpdate`，同时如果父节点和自己都不是repaintBoundary，则将父节点也加入`_nodeNeedsCompositingBitsUpdate`

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

在`PipelineOwner.flushCompositingBits()`阶段，`_nodesNeedingCompositingBitsUpdate`里的`RenderObject`根据在树中的深度排序后依次调用每个元素的`_updateCompositingBits()`方法，来完成**compositing**标识的更新过程，之后就进入到**Paint**阶段了

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
> 
> RenderObject中有一个属性`needsCompositing`，用来表示该节点或其子节点在绘制(paint)时使用某些操作（clip，transform等）是否需要使用一个单独的Layer
> 
> 如果某个`RenderObject`是`repaintBoundary`或`alwaysNeedCompositing`，则这个节点会使用单独的一层Layer来绘制（即拥有一个单独的canvas），并且当前节点及其所有祖先节点的`needsCompositing`都会被设置为true，具体可见`RenderObject._updateCompositingBits()`方法：
> ```dart
> class RenderObject {
>   //...
>     void _updateCompositingBits() {
>     
>       if (!_needsCompositingBitsUpdate)
>         return;
>         
>       final bool oldNeedsCompositing = _needsCompositing;
>       _needsCompositing = false;
>       
>       //遍历所有的child，如果有子节点needsCompositing，则当前节点也needsCompositing
>       visitChildren((RenderObject child) {
>         child._updateCompositingBits();
>         if (child.needsCompositing) {
>           _needsCompositing = true;
>         }
>       }
>       
>       //如果是repaintBoundary或者被设置为alwaysNeedsCompositing，则needsCompositing
>       if (isRepaintBoundary || alwaysNeedsCompositing) {
>         _needsCompositing = true;
>       }
>       
>       //needsCompositing改变后需要重新绘制
>       if (oldNeedsCompositing != _needsCompositing) {
>         markNeedsPaint();
>       }
>       
>       _needsCompositingBitsUpdate = false;
>     }
>   //...
> }
> ```
> 
> 那么当这个节点或其祖先节点在Layer上绘制时，某些操作不能直接绘制在当前canvas上，因为当前canvas的操作无法影响子节点Layer，这时就必须提供一个单独的ClipLayer或者TransformLayer，从而使这些操作能够影响子节点的Layer
> 
> 受`needsCompositing`影响的包括`PaintingContext`中的几个方法：
> * `PaintingContext.pushClipRect()`
> * `PaintingContext.pushClipRRect()`
> * `PaintingContext.pushClipPath()`
> * `PaintingContext.pushTransform()`
>
> 以`PaintingContext.pushTransform()`为例：
> ```dart
> class PaintingContext {
>   //...
>   TransformLayer pushTransform(bool needsCompositing, Offset offset, Matrix4 transform, PaintingContextCallback painter, { TransformLayer oldLayer }) {
>     if (needsCompositing) {
>       //如果需要compositing就创建一个TransformLayer
>       final TransformLayer layer = oldLayer ?? TransformLayer();
>       layer.transform = effectiveTransform;
>       pushLayer(layer,  painter,  offset,  childPaintBounds:MatrixUtils.inverseTransformRect(effectiveTransform, estimatedBounds),);
>       return layer;
>     } else {
>       //如果不需要compositing则直接在当前画布上操作
>       canvas..save()..transform(effectiveTransform.storage);
>       painter(this, offset);
>       canvas.restore();
>       return null;
>     }
>   }
>   //...
> }
> ```

* **_nodesNeedingPaint**：这个列表保存了需要重新Paint的`RenderObject`，当`RenderObject.markNeedsPaint()`方法被调用时，如果自己是repaintBoundary，则将自己添加到`_nodesNeedingPaint`，否则向上寻找最近的repaintBoundary添加到`_nodesNeedingPaint`：

```dart
class RenderObject {
  //...
  void markNeedsPaint() {
  
    if (_needsPaint)
      return;
    _needsPaint = true;
    
    if (isRepaintBoundary) {
      //如果自己是repaintBoundary，则将自己加入_nodesNeedingPaint列表
      if (owner != null) {
        owner._nodesNeedingPaint.add(this);
        owner.requestVisualUpdate();
      }
    } else if (parent is RenderObject) {
      //如果自己不是repaintBoundary，向上寻找repaintBoundary
      final RenderObject parent = this.parent as RenderObject;
      parent.markNeedsPaint();
    } else {
      if (owner != null) {
        owner.requestVisualUpdate();
      }
    }
  }
  //...
}
```
在`PipelineOwner.flushPaint()`阶段，`_nodesNeedingPaint`里的`RenderObject`根据在树中的深度排序后依次在每个元素上调用`PaintingContext.repaintCompositedChild()`方法，来完成**Paint**过程：
```dart
class PipelineOwner {
  //...
  void flushPaint() {
  
    final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
    _nodesNeedingPaint = <RenderObject>[];
    
    //遍历
    for (final RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
      if (node._needsPaint && node.owner == this) {
        if (node._layer.attached) {
          //绘制RenderObject
          //这里最终会调用RenderObject._paintWithContext()方法，执行真正的绘制
          PaintingContext.repaintCompositedChild(node);
        } else {
          node._skippedPaintingOnLayer();
        }
      }
    }
  }
  //...
}
```

>关于PaintingContext
> (见Paint部分)

* **_nodesNeedingSemantics**：待补充

## GestrueBinding

## SementicsBinding
