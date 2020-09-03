---


---

<h2 id="isolate">Isolate</h2>
<p>Dart中的<code>Isolate</code>对应一个线程/进程，一个Dart环境中可以运行多个<code>Isolate</code><br>
Dart的入口方法（默认是<code>main()</code>）执行在main isolate中，当需要做复杂计算或IO操作时，可以通过<code>Isolate.spawn()</code>方法创建新的<code>Isolate</code>，<code>Isolate</code>之间通过<code>ReceivePort</code>/<code>SendPort</code>通信:</p>
<p><img src="https://raw.githubusercontent.com/Ryan-Hu/DOC/master/flutter-dart-internal-isolate.svg" alt="enter image description here"></p>
<pre class=" language-dart"><code class="prism  language-dart"><span class="token keyword">import</span> <span class="token string">'dart:io'</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token string">'dart:async'</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token string">'dart:isolate'</span><span class="token punctuation">;</span>

Isolate isolate<span class="token punctuation">;</span>

<span class="token keyword">void</span> <span class="token function">start</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token keyword">async</span> <span class="token punctuation">{</span>
  ReceivePort receivePort<span class="token operator">=</span> <span class="token function">ReceivePort</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">//port for this main isolate to receive messages.</span>
  isolate <span class="token operator">=</span> <span class="token keyword">await</span> Isolate<span class="token punctuation">.</span><span class="token function">spawn</span><span class="token punctuation">(</span>runTimer<span class="token punctuation">,</span> receivePort<span class="token punctuation">.</span>sendPort<span class="token punctuation">)</span><span class="token punctuation">;</span>
  receivePort<span class="token punctuation">.</span><span class="token function">listen</span><span class="token punctuation">(</span><span class="token punctuation">(</span>data<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    stdout<span class="token punctuation">.</span><span class="token function">write</span><span class="token punctuation">(</span><span class="token string">'RECEIVE: '</span> <span class="token operator">+</span> data <span class="token operator">+</span> <span class="token string">', '</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

<span class="token keyword">void</span> <span class="token function">runTimer</span><span class="token punctuation">(</span>SendPort sendPort<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  int counter <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
  Timer<span class="token punctuation">.</span><span class="token function">periodic</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">Duration</span><span class="token punctuation">(</span>seconds<span class="token punctuation">:</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token punctuation">(</span>Timer t<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    counter<span class="token operator">++</span><span class="token punctuation">;</span>
    String msg <span class="token operator">=</span> <span class="token string">'notification '</span> <span class="token operator">+</span> counter<span class="token punctuation">.</span><span class="token function">toString</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  
    stdout<span class="token punctuation">.</span><span class="token function">write</span><span class="token punctuation">(</span><span class="token string">'SEND: '</span> <span class="token operator">+</span> msg <span class="token operator">+</span> <span class="token string">' - '</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  
    sendPort<span class="token punctuation">.</span><span class="token function">send</span><span class="token punctuation">(</span>msg<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

<span class="token keyword">void</span> <span class="token function">stop</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>  
  <span class="token keyword">if</span> <span class="token punctuation">(</span>isolate <span class="token operator">!=</span> <span class="token keyword">null</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
      stdout<span class="token punctuation">.</span><span class="token function">writeln</span><span class="token punctuation">(</span><span class="token string">'killing isolate'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
      isolate<span class="token punctuation">.</span><span class="token function">kill</span><span class="token punctuation">(</span>priority<span class="token punctuation">:</span> Isolate<span class="token punctuation">.</span>immediate<span class="token punctuation">)</span><span class="token punctuation">;</span>
      isolate <span class="token operator">=</span> <span class="token keyword">null</span><span class="token punctuation">;</span>        
  <span class="token punctuation">}</span>  
<span class="token punctuation">}</span>

<span class="token keyword">void</span> <span class="token function">main</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token keyword">async</span> <span class="token punctuation">{</span>
  stdout<span class="token punctuation">.</span><span class="token function">writeln</span><span class="token punctuation">(</span><span class="token string">'spawning isolate...'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token keyword">await</span> <span class="token function">start</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  stdout<span class="token punctuation">.</span><span class="token function">writeln</span><span class="token punctuation">(</span><span class="token string">'press enter key to quit...'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token keyword">await</span> stdin<span class="token punctuation">.</span>first<span class="token punctuation">;</span>
  <span class="token function">stop</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  stdout<span class="token punctuation">.</span><span class="token function">writeln</span><span class="token punctuation">(</span><span class="token string">'goodbye!'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token function">exit</span><span class="token punctuation">(</span><span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>与线程不同，Isolate之间不能共享内存，因此也不需要内存资源的锁机制，可以高效回收内存</p>
<h2 id="event--microtask">Event &amp; Microtask</h2>
<p><code>Isolate</code>是个单线程，他的运行机制是Event Loop，与安卓端的主线程非常类似，不过在<code>Isolate</code>中有两个队列，分别对应的是普通的<strong>event</strong>和优先级较高的<strong>microtask</strong>，当<strong>microtask</strong>队列中有任务时，优先运行这些任务，当<strong>microtask</strong>队列为空时，才会运行普通<strong>event</strong>队列中的任务</p>
<p><img src="https://raw.githubusercontent.com/Ryan-Hu/DOC/master/flutter-dart-internal-queue.svg" alt="enter image description here"></p>

