## Isolate
Dart中的`Isolate`对应一个线程/进程，一个Dart环境中可以运行多个`Isolate`
Dart的入口方法（默认是`main()`）执行在main isolate中，当需要做复杂计算或IO操作时，可以通过`Isolate.spawn()`方法创建新的`Isolate`，`Isolate`之间通过`ReceivePort`/`SendPort`通信:

![enter image description here](https://raw.githubusercontent.com/Ryan-Hu/LearnFlutter/master/images/dart-internal-isolate.svg)

```dart
import 'dart:io';
import 'dart:async';
import 'dart:isolate';

Isolate isolate;

void start() async {
  ReceivePort receivePort= ReceivePort(); //port for this main isolate to receive messages.
  isolate = await Isolate.spawn(runTimer, receivePort.sendPort);
  receivePort.listen((data) {
    stdout.write('RECEIVE: ' + data + ', ');
  });
}

void runTimer(SendPort sendPort) {
  int counter = 0;
  Timer.periodic(new Duration(seconds: 1), (Timer t) {
    counter++;
    String msg = 'notification ' + counter.toString();  
    stdout.write('SEND: ' + msg + ' - ');  
    sendPort.send(msg);
  });
}

void stop() {  
  if (isolate != null) {
      stdout.writeln('killing isolate');
      isolate.kill(priority: Isolate.immediate);
      isolate = null;        
  }  
}

void main() async {
  stdout.writeln('spawning isolate...');
  await start();
  stdout.writeln('press enter key to quit...');
  await stdin.first;
  stop();
  stdout.writeln('goodbye!');
  exit(0);
}
```
与线程不同，`Isolate`之间不能共享内存，因此也不需要内存资源的锁机制，可以提高内存回收的效率

## Event & Microtask
`Isolate`是个单线程，他的运行机制是Event Loop，与安卓端的主线程非常类似，不过在`Isolate`中有两个队列，分别对应的是普通的**event**和优先级较高的**microtask**：

![enter image description here](https://raw.githubusercontent.com/Ryan-Hu/LearnFlutter/master/images/dart-internal-queue.svg)

当**microtask**队列中有任务时，优先运行这些任务，当**microtask**队列为空时，才会运行普通**event**队列中的任务

![enter image description here](https://raw.githubusercontent.com/Ryan-Hu/LearnFlutter/master/images/dart-internal-looper.svg)

