> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/0uZuoZp18qzdMPDIC1oNjQ)

> 这是笔者通过 AI 把抖音 Web 的反爬签名 `a_bogus` 从零还原成纯 Python 的完整过程。中间三次咬错「真凶」、放弃过一条看似正确的硬解路线、手写了一台栈式虚拟机的解释器、逐字节逼出二十多个 bug，最后拿真 douyin 真实接口验证通过。
> 
> 写这篇既是给自己留个复盘，也想把「逆一台自研 VM」这件事的完整判断链摊开——哪一步为什么这么走、哪一步走错了又怎么掰回来。
> 
> 适合读的人：做 Web 逆向、接口自动化，或者对 JSVMP 这类自研虚拟机保护感兴趣的同行。文中默认你熟悉 base64、XOR、哈希这些基础，但 VM 那部分会从零讲起。
> 
> **全文内容仅用于安全研究与学习交流，请勿用于任何违反目标站点条款或相关法规的用途。**

一、一个挂在每个请求后面的参数
---------------

抖音 Web 端每个 API 请求的 query 后面，都挂着一个 `a_bogus`，长这样：

```
xJsfkFWjQqmjFd/b8CG6Ca1lcHj/rp8yRGkOWfNrexKbKH0OquYQuxegnxwzs8VmTmpkhq17AVF/bExc0av03onkzmkkuQ7WPs5C9Wvo/qqVP0JsgHDFC0szowBGMbsLaQ9Xilf5XsMw6DOlIH50Ap5Gy5zERQbpbNeAdou9tEWXDCSkin3iOCkpqgRa


```

188 到 192 个字符，字符集是 `A-Za-z0-9/-` 再加个 `=`，一眼看上去像自定义 base64。不带它去请求接口，服务器要么给你空 body，要么甩你一个验证挑战。所以想写抖音的接口自动化，第一道坎就是它。

这件事的起点是一个听起来很简单的问题：这个字符串到底是哪段 JS 算出来的？本以为很快能定位，结果光是这个问题本身就耍了我三次——先后咬定过三个「真凶」，两个是错的。

而终点，是我一开始没敢奢望的：纯 Python、零浏览器依赖，逐字节复现 bdms.js 的输出，query、时间戳、随机数三个输入全自由，拿去请求真实 douyin 接口，服务器 200 接受。下面按顺序讲，怎么从前者走到后者。

二、三套 SDK 各签各的
-------------

先把地形讲清楚，不然后面绕的弯路会看不懂为什么绕。

抖音 Web 的安全 SDK 不是一个文件，而是三套东西同时在跑，各签各的：

*   `webmssdk.es5.js`（387 KB）管 `X-Bogus` / `frontierSign` / `msToken`。里面是 25 个 `HNOJ@?RC` 魔数开头的字节码块，加一台叫 `_$webrt_1668687510` 的 JSVMP 解释器，obfuscator.io 混淆。这套和抖音老的 `ac_signature` 是同一套 VM。
*   `runtime_bundler_34.js`（266 KB）是 `security-secsdk` 的运行时，挂钩 XHR/fetch 的 `onRequest`，里面有一台 `for(var v=[];;)try{switch(r[a++]){case…}}` 的 switch 派发 VM 加一整套 CryptoJS，产出 `x-secsdk-web-signature` 头。
*   `bdms_1.0.1.19_fix.js`（147 KB）是第三套，单行 minified，不是 obfuscator.io，相对可读，也最不起眼。

三个文件、三套签名机制，`a_bogus` 是其中一套算的。但它不在任何一个文件里以字面量形式出现——`grep -c 'a_bogus'` 三个文件全是 0。这是第一个信号：这个字符串是运行时构造出来的，藏在某台 VM 的字符串表里。

难点也就摊开了。首先是不知道哪套 VM 生成，三选一还都混淆；其次就算定位到，VM 字节码是加密压缩过的，得先解出来；最后一层是我当时完全没预料到的——就算把算法逆出来，它还会读一堆浏览器指纹和反 headless 探针，纯脚本环境天然对不上。第三层后来成了整个逆向里最硬的一关，这里先按下不表。

三、两次找错，一次实锤
-----------

### 先怀疑 runtime_bundler，抓到 0 opcode

最先怀疑 `runtime_bundler_34.js`。理由很直接：它是 secsdk 的运行时，明摆着挂钩了 XHR/fetch 的 `onRequest`，而 `a_bogus` 是加在请求 URL 上的，谁挂钩请求谁最可疑。

它 line 2062 有一台 switch 派发 VM：`for(var v=[];;)try{switch(r[a++]){case…}}`，IP 是 `a`、栈是 `p`、字符串表是 `t`（XOR 编码），56 KB 字节码，magic 是 `504B0101`（`PK\x01\x01`，伪装成 ZIP 头但不是标准 ZIP）。结构像抖音老的 acrawler，但更大更新。

给这台 VM 的 switch 派发插桩、想抓 opcode 轨迹的时候，我先踩了个基础设施级的坑。插桩后 route 掉真文件、注入自己的版本，结果 a_bogus 生成期间抓到 0 opcode，一度以为 secsdk 跑在 Web Worker 里（`__RT_MARK=0`）。折腾半天才发现真相是 HTTP 缓存——playwright 的 route 没命中被缓存的 JS。加上 `--disk-cache-size=0` 再配 `route.fetch()` 绕缓存重写之后，route 命中了、`__RT_MARK=1`（主线程），worker 全 0。

但即使插桩确实生效，a_bogus 生成期间的 switch-VM 还是 0 opcode。同时字符串表 dump 出来是整套 CryptoJS（MD5/SHA256/HMAC/Base64/Base64url/3DES）加 `x-secsdk-web-signature`。结论翻转：这台 switch-VM 只在 init 时疯跑一次（加载后累计 112504 opcode，解密 CryptoJS 串、建签名闭包），请求期一个 opcode 都不跑。它签的是 `x-secsdk-web-signature` 头，跟 a_bogus 没关系。

第一个真凶排除。回头看，这里犯的错是把「挂钩了请求」直接等同于「算了 a_bogus」——secsdk 挂钩请求只是为了叠自己那套头，顺手而已。

### 再怀疑 webmssdk，还是 0

转向 `webmssdk.es5.js`。它里面有 `bogus` / `msToken` / `X-Bogus` / `frontierSign` / `MD5` 这些扎眼的字面量，25 个字节码块全是 `HNOJ@?RC` magic，解释器是 `_$webrt_1668687510`，跟之前逆过的 ac_signature 完全同一套 JSVMP。那套 VM 熟门熟路，心想这回稳了。

做了两个置空实验：把 webmssdk 用空文件 route 掉，a_bogus 照样生成 192 字符，说明它不依赖 webmssdk；给 webmssdk 的 `_$webrt` VM 插桩，a_bogus 生成期间 0 opcode。两条独立证据把 webmssdk 也排除了。它只管 X-Bogus / frontierSign / msToken，跟 a_bogus 是两码事。

这时候有点被打击。三个文件排掉两个，剩下的 bdms 是最不起眼的那个，连那些扎眼的字面量都没有。甚至一度得出过一个错误的中间结论——「a_bogus 不走任何 VM，是 runtime_bundler 里的可读 JS 用 CryptoJS 算的」，这个结论后来也被推翻了。

这段弯路的教训后来反复回来提醒我：手里有多个可疑对象、每个都混淆时，不要靠「哪个看起来最像」来选，靠「谁手上有赃物」来选。所谓赃物，就是那个 188 字符的字符串本身。该做的不是逐个 VM 插桩猜，而是直接盯住 a_bogus 落地的那一刻，反查是谁塞的。

### hook `URLSearchParams.append`，抓栈实锤

想通这一层就简单了。`a_bogus` 是 query 参数，query 参数最终会经过 `URLSearchParams.append("a_bogus", ...)`。那就不猜了，直接 hook 这个方法，在它被调用、且 key 是 `a_bogus` 的那一刻，打一份完整调用栈：

```
const _append = URLSearchParams.prototype.append;
URLSearchParams.prototype.append = function (k, v) {
  if (k === 'a_bogus') console.trace('A_BOGUS APPEND', v);
  return _append.call(this, k, v);
};


```

这里有个 capture 要点：`new URL(u).searchParams` 用的是原生 URLSearchParams，所以 hook 要 patch `URLSearchParams.prototype`，不能只 patch 某个子类实例。

栈打出来，函数落在 `bdms_1.0.1.19_fix.js` 的 `d` / `X` 函数里。第三个文件，那个最不起眼的 bdms，才是真凶。它挂钩 XHR.send，SPA 发请求时算 a_bogus 塞进 query。

给 bdms 里那台 `for(;;){var t=o[a++];if(t<38)if(t<19)...}` 的二分派发 VM 插桩，抓一份 golden-trace。所谓 golden-trace，就是把 VM 每执行一条指令时的 `[funcId, ip, op, 栈指针]` 逐条记下来，攒成一条完整轨迹——它是这篇文章后面反复用到的「标准答案」，逐指令校验时拿它做基准。一次签名抓到 64608 个 opcode（后来精确切割后是 63450）。这次插桩有效、a_bogus 正常生成，真凶锁定。加载地址是 `https://p-pc-weboff.byteimg.com/tos-cn-i-9r5gewecjs/bdms_1.0.1.19_fix.js`，playwright 里用 `route("**/bdms*")` 拦。

从怀疑 runtime_bundler 到锁定 bdms，浪费了大半天在「逐个 VM 插桩验证」上。如果一开始就用「hook 落地点反查栈」这个思路，二十分钟就能到位。这条经验后来在最后攻 detail 时又派上一次用场：当目标是「某个具体值从哪来」时，盯值、别盯代码。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fJBlDTU8pLGTkOibU1Npq18xR0fribQVnuuaFiaqlGDxqHIHld60962bfmaibz1IdKTG7Qerqqy1ZOdwmRbL2WJYXSI2MJjHmADwvC0Uu4XibNlQ/640?from=appmsg&watermark=1#imgIndex=0)a_bogus 真凶定位

_图：三套 SDK 逐个排查，前两个签名期 0 opcode 被排除，最不起眼的 bdms 才是真凶。_

四、不硬解算法，重写这台 VM
---------------

抓到 64608 个 opcode 的轨迹之后，开始面临一个选择。

一条路是硬逆算法：顺着 golden-trace 和原生调用记录，把这台 VM 在算什么反推出来——RC4 类字节运算、自定义 base64、哈希、指纹折叠。之前逆 ac_signature 就是这么干的，抓了 22 个 bug，跨了好几个 session。另一条路是重写这台 VM：不管它算什么，把这台自研栈式虚拟机的解释器逐字节忠实地用 Python 实现一遍，喂给它同样的字节码和输入，让它自己算出 a_bogus（感觉也只有 AI 才能干这事了）。

我选了第二条。理由是：硬逆的工作量随算法复杂度线性增长，而且没有中途验证锚点——你逆错一个字节，要到最后校验才发现，然后不知道错在哪；而重写 VM 的工作量是固定的（就 77 个 opcode，写完就完了），并且有 golden-trace 这个完美的验证锚点——可以让 Python 解释器逐 opcode 和浏览器抓的轨迹对比，第一处分歧就是 bug 所在。前者是「逆一个未知函数」，后者是「抄一台已知机器」。这个选择后来被证明是对的。

但它有个前提：需要能反复、廉价地拿到 ground-truth 来校验。而 playwright 跑真浏览器又慢又要开窗口，迭代一次几十秒。所以动手写解释器之前，第一件事是把真值签名器建起来。

### 把真 bdms.js 搬进 Node 跑

把 `bdms.js` 用 Node 的 `vm` 模块跑了起来。这一步是整个项目的杠杆点，它让整套迭代不再需要真机或浏览器，可以无限次、秒级地重跑、改冻结值、抓任意中间态。思路是造一个受控的浏览器环境沙箱：

```
const vm = require('vm');
const sandbox = {};
// 注入 Node vm context 默认没有的：atob / URLSearchParams / TextDecoder / setTimeout ...
sandbox.atob = (s) => Buffer.from(s, 'base64').toString('binary');
sandbox.URLSearchParams = URLSearchParams;
// 冻结熵源
sandbox.Math = Object.create(Math); sandbox.Math.random = () => 0.123456789;
const RealDate = Date;  // 静态 now() 和无参 new Date() 都钉到固定时刻，有参构造透传
function PatchedDate(...a) { return a.length ? new RealDate(...a) : new RealDate(1700000000000); }
PatchedDate.now = () => 1700000000000; PatchedDate.prototype = RealDate.prototype;
PatchedDate.parse = RealDate.parse; PatchedDate.UTC = RealDate.UTC;
sandbox.Date = PatchedDate;
// 浏览器 stub：navigator / screen / document / location / XMLHttpRequest ...
vm.createContext(sandbox);
vm.runInContext(bdmsSrc, sandbox);


```

第一次跑就崩，然后是一串「缺 native」的连环坑，每一个都顺带暴露了 bdms 的行为。Node 26 的 `navigator` 是只读 getter，不能直接 `g.navigator = ...` 赋值，必须用 `vm.createContext` 沙箱全套自己造。bdms 会 `(window.webviewBridge || window.parent.webviewBridge).callBrowserWindow('getClientInfo')`——这是 native App 里的 JSBridge，纯 web 下 `webviewBridge` 是 undefined，读 `.callBrowserWindow` 直接抛；真浏览器里这个异常被 caller 的 async try/catch 接住、回退到纯 web 指纹，所以给了个 reject 版的 bridge，让它走回退路径。`z[592]` 是一整套 WebGL 指纹——`getContextAttributes` 加十几个 `getParameter`（BLUE_BITS / MAX_TEXTURE_SIZE / UNMASKED_RENDERER_WEBGL）加两个 extension，得把 WebGL context 全 stub 出来，返回一套确定的 Mac Chrome 值。

最后一个坑卡了一会：`bdms.init({aid:6383})` 之后触发 XHR，a_bogus 怎么也不生成。反汇编发现 wrapped open（`z[105]`）会先跑一个 `config.paths.include` 的正则匹配（`z[98]`），不匹配就直接调原始 open、不签。douyin 的 init 配了这套规则，直接照着传：

```
bdms.init({ aid: 6383, paths: { include: [/aweme/, /passport/, /webcast/], exclude: [] } });


```

a_bogus 出来了。更关键的是——`xJsfkFWjQqmjFd/b8C…`——前缀和真浏览器 golden-trace 里的 a_bogus 完全一致。后半段因为指纹和 msToken 环境不同而不同，但前缀吻合证明 query 派生字节这条路是对的、且和环境无关。本地 oracle 建成。

一个后面反复用到的确定性细节：进程内第 1 次 sign 是 warm-up（msToken/state 未 settle），第 2 次起才稳定，校验要用稳态值。至此有了一台可以秒级重跑、逐字节确定的真值发生器，剩下的就是把它 Python 化。

五、读懂这台 VM
---------

写 Python 之前，得先把这台 VM 的结构彻底看懂。它的核心是四个互相嵌套的函数，都 hoisting 在 `X` 里，闭包共享一套寄存器 `o,i,u,s,c,a,f,l,p,v,h`：

*   `g(t,r,e,n)` @131148 是帧初始化。`t=[字节码, 参数个数, strict标志, try表]`，`r` 是 this，`e` 是参数，`n` 是闭包作用域。它设 `o=t[0]`（字节码）、`a=0`（IP）、`s=[n, locals]`（作用域链）、`f=0`、`l=undefined`，但不重置操作数栈 `v` 和栈指针 `p`。
*   `d()` @131642 是派发循环，`for(;;){var t=o[a++]; if(t<38)if(t<19)...}` 一棵二分树，是 VM 的心脏。
*   `X(t,r,e,n)` 跑一个函数：`g(t,r,e,n); do{try{d()}catch(t){f=3,l=t}}while(y()); return l`。注意它把 `d()` 的真实 JS 异常 catch 成 `f=3`（VM 级异常）。
*   `y()` 做异常 / 返回展开，按标志 `f`（1=return / 2 = 弹帧 / 3 = 异常）用 try 表和帧栈 `h` 做栈展开。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fJBlDTU8pLGCXwVtPF4fTXB41juAHY8PrArte2KQQvULEMSHHbto36aI10LAyNtUR2fS7nfcttbSPD6PfGrH8ue5MBE1asOt60kF5s7WticM/640?from=appmsg&watermark=1#imgIndex=1)bdms 栈式 VM 的 g/d/X/y 四件套帧机

_图：X 驱动 g 建帧、d 派发循环、y 展开返回，四个函数闭包共享同一套寄存器。_

有三个语义细节，是后来写 Python 时反复回来核对的，也是最容易错的。

其一，操作数栈跨帧连续共享。`g()` 只重置 `o/a/s`，不动 `p/v`。所以当 op 0（CALL）调一个 VM 函数时，`p-=argc` 消掉参数，被调函数的第一次 push 就接在共享栈顶上。一次 `d()` 调用横跨多层嵌套帧连续执行——op 0 进新帧后不 return，继续循环。这是它高效的地方，也是最反直觉的地方。

其二，「返回」分两步。op 52（`return <expr>`）设 `f=1, l=a+U`（跳到本函数出口），走完 finally；真正弹帧、把返回值推给 caller 的是 op 76（`f=2, l=v[p--]` → `y()` 弹 `h`）。一开始按旧笔记以为最大 opcode 是 75，结果 golden-trace 里 op 76 出现了 256 次、正好等于 256 次帧入口——opcode 实际到 76，op 76 才是函数返回，op 75 是 push null。

其三，最高频 opcode 是 74（作用域读元素），一次签名 21733 次。`N=o[a++],x=o[a++]; U=s; N 次 U=U[0]; push U[x]`——沿作用域链上 N 层、取第 x 个槽。scope 帧结构是 `[parent, localsObj, arg0, arg1, …]`：`s[0]` 是父链、`s[1]` 是 locals、`s[2+]` 是 slot。

### 怎么从二分派发树读出一个 opcode

上面提了 77 个 opcode，但「怎么从混淆的派发树把每个 opcode 的语义读出来」是这台 VM 里最硬的活，讲不清楚整张 opcode 表就是黑盒。

`d()` 的派发不是 switch，而是一棵手写二分树。下面是从 bdms.js @131642 现取的原文开头：

```
for(;;){var t=o[a++];
  if(t<38)
    if(t<19)
      if(t<9)
        if(t<4)
          if(t<2)
            if(0===t){var r=o[a++];p-=r;var e=v.slice(p+1,p+r+1),n=v[p--],d=v[p--];
                      if("function"!=typeof n)return f=3,void(l=...);
                      var y=V.get(n);if(y)h.push([o,i,u,s,c,a,f,l]),g(y[0],d,e,y[1]);
                      else{var m=n.apply(d,e);v[++p]=m}}          // ← op 0
            else{var w=v[p--];v[p]=v[p]<=w}                        // ← op 1
          else if(2===t)w=v[p--],v[p]=v[p]>w;                      // ← op 2
          else{var x=o[a++],S=v[p--],P=[];for(var j in S)P.push(j);s[x]=[P,S]}  // ← op 3
        else if(t<6)
          if(4===t){...do{j=P[0].shift()}while(...);...}          // ← op 4
          else{x=o[a++];var A=Z[x],E=b(A,i);v[++p]=E,v[++p]=A}    // ← op 5
        else if(t<7)w=v[p--],v[p]=v[p]!==w;                       // ← op 6
        else ...


```

读法其实很机械：`t` 是 opcode，每层 `if(t<X)` 就是一次二分。要读 op N 的语义，顺着 `t<...` 一路走到它的叶子，那段花括号里的 JS 就是它的语义。拿 op 0（CALL）举例：走 `t<38→t<19→t<9→t<4→t<2→0===t`，叶子是 `r=o[a++]`（读 1 个操作数，即实参个数 argc）、`p-=r`（栈上弹掉 r 个参数）、`n=v[p--]`（弹函数）、`d=v[p--]`（弹 this）；然后 `V.get(n)` 判断是 VM 函数还是原生——是 VM 函数就 `h.push(当前帧)` 保存现场、`g(...)` 进新帧（注意不 return，继续外层 for 循环），是原生就 `n.apply(d,e)` 直接调、结果压栈。op 1（LE）是 `0===t` 的 else，`w=v[p--];v[p]=v[p]<=w`，弹一个、栈顶 `<=` 它。其余同理顺着树走到叶子读出。

这树有 70 多个叶子，`if(t<X)` 边界极易看错。人肉核了两遍边界，还是留了几处存疑（附录 E 的全表里标 §）。核对方法是写完 opcode 表后用它反汇编，看开头能不能对上 golden-trace（后文验证）。派发树全文另存了 `artifacts/VM_DISPATCH.txt`，机器可读的最终表是 `pyvm/disasm.py`（`NAMES`/`OPERANDS`）——这张校正表和早期手写表有出入，比如 op6 早期误标成「压大常量」，读派发原文才确认是 `!==`。

### opcode 语义表：4 个关键的，加完整参考

整棵派发树读完，得到全 77 个 opcode 的语义表（`pyvm/disasm.py` 的校正版）。但正文后面真正反复用到的其实只有四个——op 0（CALL）、op 74（最高频的作用域读）、op 52 与 op 76（返回分两步）。所以这里只列这四个，完整 77 条挪到了文末附录 E，需要查某个 op 时再去翻。

记号先约定好，后面一直用：栈是 `v`、指针是 `p`、字节码是 `o`、IP 是 `a`、常量表是 `Z`、作用域链是 `s`；`v[p--]` 出栈，`v[++p]=x` 入栈，`o[a++]` 取操作数。「操作数个数」指这个 op 从 `o[a++]` 读几个立即数。

<table><thead><tr><th>op</th><th>名字</th><th>操作数</th><th>语义</th></tr></thead><tbody><tr><td>0</td><td>CALL</td><td>1</td><td><code>r=argc; 弹 r 参+函数+this; VM 函数→进帧(不 return) / native→apply 压栈</code></td></tr><tr><td>52</td><td>RET_OFFSET</td><td>1</td><td><code>U=op; f=1; l=a+U</code>（return 到偏移、走 finally——「返回分两步」的第一步）</td></tr><tr><td>74</td><td>SCOPE_READ</td><td>2</td><td><code>N,x=op,op; 沿 s 上 N 层; v[++p]=U[x]</code>（全表最高频，一次签名 21733 次）</td></tr><tr><td>76</td><td>RET</td><td>0</td><td><code>f=2; l=pop</code>（真弹帧、函数返回——返回的第二步）</td></tr></tbody></table>

六、把字节码解出来
---------

opcode 表有了，但字节码本身不是明文，藏在一个 38780 字符的 base64 blob 里（`"UEsC..."`，`UEsC` 就是 `PK\x02` 的 base64，@92097）。解码逻辑在 `J()` 函数里，逐行翻成 Python：

```
raw   = base64.b64decode(blob)                 # atob
key   = sum(raw[4:8]) % 256                    # 取 byte 4-7 求和当种子
xored = bytes((raw[8+i] ^ ((key + (key%10)*i) % 256)) & 0xFF   # 逐字节 XOR，位置相关
               for i in range(len(raw)-8))
data  = zlib.decompress(xored, -15)            # raw DEFLATE（无 zlib 头）


```

最容易踩的是那个 `-15`。bundle 里的解压函数是 `C(t,r)=k(t,{i:2},...)`，那个 `{i:2}` 一开始骗了我，看着像「跳过 2 字节 zlib 头」。核了 `k` 的实现才确认：`b=e.p||0` 是从比特 0 开始读的，`{i:2}` 只是个非流式标志。所以是从字节 0 开始的裸 DEFLATE，对应 `zlib.decompress(xored, -15)`。

解出来 86939 字节，接着是一个变长整数流。`W(r)` 是标准的有符号 LEB128（要注意 JS 的 32 位符号语义），`K(r)` 是个自定义的 UTF-8 式 codepoint 串解码（byte ≥ 0xF8 作终止哨兵）：

```
nZ = W()                     # 常量个数
Z  = [K() for _ in range(nZ)]      # 常量/字符串表
nfn = W()                    # 函数个数
for _ in range(nfn):
    argc, strict = W(), bool(W())
    trys = [[W(),W(),W(),W()] for _ in range(W())]   # try 表
    bc   = [W() for _ in range(W())]                 # 字节码——每个数是一个 varint！
    z.append([bc, argc, strict, trys])


```

关键点是字节码数组里每个数本身是一个 varint 展开后的整数，不是原始字节，所以跳转偏移、操作数都已经是完整整数了。最终产物是 `Z` = 1001 个常量串、`z` = 796 个函数（`python3 -c "from pyvm.decode import decode_blob; Z,z,_=decode_blob(); print(len(Z),len(z))"` 输出 `1001 796`）。

怎么验证解码和 opcode 表都对？不用另找真值，直接看解出的字节码开头能不能对上 golden-trace。轨迹头三条是 `[74,0,...] [30,3,...] [38,5,...]`：op 74 读 2 操作数（IP 0→3）、op 30 读 1 个（3→5）、op 38 读 1 个且 `o[6]=1`。在 796 个函数里找 `bc[0]==74 and bc[3]==30 and bc[5]==38`，命中的函数 `bc[6]` 正好是 1。解码加 opcode 操作数个数表逐 op 对上了，这是第一个里程碑。

![](https://mmbiz.qpic.cn/mmbiz_png/fJBlDTU8pLFX2bACMzIribtWUAre9iaJLkCjKibliadJlKKGydmic78w26X3AswoibyjAPDUUkDnVCDOYShKlzont9w6JKv46ibrib8hBDxZl0gKtkI/640?from=appmsg&watermark=1#imgIndex=2)字节码解码管线

_图：从加密 blob 到 1001 个常量 + 796 个函数的五步解码管线。_

七、找到核心：z[104] → z[107] → z[103] → z[150]
----------------------------------------

VM 有了、字节码有了、能反汇编了，接下来找哪个函数是核心签名。靠反汇编器加 Node oracle 插桩，一层层往下剥。用 `pyvm/disasm.py` 反汇编（`disasm.disasm(z[103][0], Z, 0, 40)` 的真实输出）：

```
z[103] 反汇编开头(9 参 wrapper)：
    0:  60 READ_GLOBAL    [202]  ; 'performance'
    2:  18 DUP
    3:  30 GETPROP        [173]  ; 'now'
    5:   0 CALL           [0]
    7:  54 SCOPE_WRITE    [0, 5]
   11:  74 SCOPE_READ     [2, 12]      ← 沿作用域上2层取slot12 = 核心函数 z[150]
   14:   0 CALL           [0]
   17:  74 SCOPE_READ     [0, 2]       ← 读参数(query...)
   ...


```

一层层剥出来的调用链是这样的。`z[104]` 是 XHR 拦截安装器，把 `XMLHttpRequest.prototype.{open,setRequestHeader,send}` 换成 `z[105]/z[106]/z[107]`。`z[107]`（wrapped send）拿到请求：`new URL(url).searchParams`，没有 msToken 就补 msToken，没有 a_bogus 就调 `z[103](searchParams.toString(), xhr)` 算出来 append 上去。`z[103]` 是 wrapper，读 `navigator.userAgent`、处理 body/content-type，然后调真正的核心 `z[150](1, 0, 8, query, body, ua, pageId, aid, "1.0.1.19-fix.01")`——9 个参数（上面反汇编里 `SCOPE_READ [2,12]` 取的就是 z[150]，`CALL` 调它）。`z[150]` 就是核心签名，返回 188 字符 a_bogus。

后文会冒出一堆 `z[NNN]`，为了不被绕晕，先把关键的几个记下来，其余的用到时再说：

<table><thead><tr><th>函数</th><th>角色</th></tr></thead><tbody><tr><td><code>z[150]</code></td><td>核心签名函数，后面所有拆解都围着它转</td></tr><tr><td><code>z[107]</code></td><td>拦截 <code>XHR.send</code> 的 wrapper，请求出去前在这里被塞进 a_bogus</td></tr><tr><td><code>z[130]</code></td><td>base64 编码与那几张字母表</td></tr><tr><td><code>z[148]</code></td><td>位交织器（第十一章装配层的主角之一）</td></tr><tr><td><code>z[280]</code></td><td>那台 RC4 变体（第十一章的「真凶」）</td></tr></tbody></table>

其余带编号的函数大多是一次性的探针或辅助（比如后面那批反检测子函数），用到时会随手点明，读者不必记住具体数字。

我给 Node oracle 打了个补丁，在 z[150] 入口那一刻切一刀、到它自己的 RET（ip 1833）为止，把这中间的一切都抓下来。切出来的子树意外地干净：63450 个 opcode、55 个函数（子树核心 14 个：`130,142,150,151,246,251,272,274,275,277,280,699,729,730`）；native 调用集中在 `charCodeAt`（1070 次）、`charAt`（344 次）、`fromCharCode`（254 次），全是纯字节和字符串运算；子树内 0 次 canvas / WebGL / Date.now 直接调用，指纹和时间戳是从闭包作用域和全局进来的，不是运行时现算。

这意味着一件大好事：核心签名是一个自包含的 14→55 函数子树，只需要移植这一块，不用移植整个 bootstrap。原来估计的「多天工程」一下收窄成「移植一个函数子树」。

这里也纠正一个曾经的误判：切割 trace 时第一次用「帧深度计数」做边界，结果被一个通过 native `map` 回调跑的 VM 闭包 z[151] 的 op 76 误触了终止，trace 被截断成 25554 op。后来改成「以 z[150] 自己的 RET@ip1833 为边界」才拿到完整的 63450。边界条件在有 native 回调的 VM 里特别容易错。

![](https://mmbiz.qpic.cn/mmbiz_png/fJBlDTU8pLGzSm160A7XP5uS37wE2FjBD7MaSA22Xfwy0VM9YZibV6dPugdvYjqGY9FZpBnZzWicGia7b01hHibUMzRbGTNiaiam9hPrsMdib2MdE4/640?from=appmsg&watermark=1#imgIndex=3)核心签名数据流

_图：a_bogus 的高层数据流。z[150] 内部「位交织 + RC4 变体」这一段的具体装配，第十一章脱 VM 时会逐字节拆开。_

### 核心哈希是国密 SM3

顺手记一个意外。z[150] 会 `new (scope(2,13))(...)` 造一个对象然后调它的 `.sum()`。抓了这个对象，它的原型上挂着 `reset / write / sum / _compress / _fill`，实例字段是 `reg / chunk / size`，是一个分块哈希。关键是它的初始寄存器：

```
this.reg[0]=1937774191, this.reg[1]=1226093241, ...   // reg = new Array(8)


```

`1937774191` 就是 `0x7380166F`，正是 SM3 国密哈希的标准 IV。再看 `_compress` 里 `r=new Array(132)`、`for(n=16;n<68;n++)` 的消息扩展——132 字消息调度、8 个寄存器、64 字节块，是 SM3，不是 MD5、不是 SHA256。而且这个哈希类是 bundle 里的原生 JS class，不是 VM 函数，它通过闭包作用域以 `nativefn` 的形式喂进 z[150]（`scope(2,3)`）。

这是个好消息：SM3 是公开标准，照着 GB/T 32905 写了一版 Python（`pyvm/sm3.py`），过了标准测试向量 `SM3("abc") = 66c7f0f4...`。这也确立了一个模式——这台 VM 会 call out 到 bundle 里的原生 JS 辅助类，移植时得在作用域里按名字注入 Python 实现。所幸最后发现核心路径上只有 SM3 这一个原生类。

八、搬进 Python：用 golden-trace 逼 bug
--------------------------------

真正的工程量在这里。写一台 Python 解释器（`pyvm/interp.py`）：JS 值模型（`UNDEF` 哨兵、`JSObject`、`JSArray`、`JSClosure`、`NativeFn`）、JS 运算语义（32 位位运算的 ToInt32/ToUint32、JS 的浮点取余、`+` 的字符串拼接、`===`/`==`）、`g/d/X/y` 帧机、全 77 opcode、那十来个 native，加上从快照重建 z[150] 闭包作用域的 loader。

调试方法就是 golden-trace：解释器每执行一个 opcode，就和 Node 抓的 `[funcId, ip, op, p]` 轨迹逐条比，第一处对不上就是 bug 所在。跑法是 `python3 -m pyvm.interp`，加载 z[150]、用 `core_scopechain.json` 重建闭包 scope、跑、逐 op 对 `core_trace.json`、报第一处分歧。分歧输出长这样，逼 bug 时反复盯的就是这行：

```
DIVERGE @ step 80:  python [funcId=730, ip=44, op=74, p=12]
                    golden [funcId=730, ip=44, op=74, p=13]   ← 栈指针 p 差 1
→ 去反汇编 z[730] ip44 附近，发现 arguments.length 读成了 undefined


```

第一次跑，5 个 opcode 就崩。然后是一场逼 bug 的马拉松，match 数字一格一格往上爬：5 → 41 → 80 → 196 → 284 → 344 → 25842 → 25929，每逼出一个 bug 就前进几十到几百个 opcode。按逼出顺序，这些 bug 是：

1.  scope 帧是 Python list，不是对象。op 74/54/61 沿作用域链走 `U=U[0]`，作用域帧用 Python list 存，父链和 getter 对象又是 `JSObject`，得写个 `_sidx(U,i)` 兼容 list 和 JSObject 混合索引。（5→41）
2.  `window.onwheelx._Ax`。z[150] 开头查这个 bdms 在 init 时注入 window 的完整性标记，值 `"0X21"`，而且非可写（一道校验）。给 `JSObject` 加只读键集合。（41→80）
3.  VM 异常要能被 VM 的 try 表接住。`document.all.__proto__` 在真浏览器也 undefined、读它会抛，但这个抛是被 VM try 表接住跳 catch 的。修法是 `try{d()}except VMError: f=3`，交给 `y()` 展开。
4.  原型链：`JSObject` 加 `proto`，get_prop 走原型；`__proto__`、非枚举、只读三套语义都要补。
5.  正则跨 realm bug，这个最阴。UA 浏览器检测那块规则是一堆 `new RegExp`，存在作用域快照里。序列化快照的函数跑在 Node 域、正则是 vm sandbox 域实例，`x instanceof RegExp` 恒 false（不同 realm），于是正则被当普通对象序列化——但 `source`/`flags` getter 值被抓到了。loader 里按「有 source+flags+exec 三个键」认出来、用 Python `re` 重建。这一修直接修好整个浏览器 UA 检测。
6.  Promise 微任务。`navigator.storage.estimate().then(cb)` 的 cb 在 sign 返回之后才在微任务里跑，golden-trace 里根本没它的 opcode。所以这个 `.then` 必须注册但不执行。
7.  screen 指纹串。z[150] 拼的指纹串是 `screen.width|height|availWidth|availHeight|window.outerWidth|outerHeight|innerWidth|innerHeight|platform`，window 少挂了内外宽高，串长度对不上、后续字节全错。补上一次对一大片。
8.  最后一块是 `arguments.length`。逼到 25929/63450 卡在一个变长函数分支。根因是 `g()` 初始化 locals 对象时会 `Object.defineProperty(v,"length",{value:e.length})`，给它挂一个非枚举的 `length` 等于实参个数。漏了这一步，所有 `arguments.length` 读出来 undefined，变长函数分支全走错。

补上 `arguments.length` 这一行的瞬间：

```
no trace divergence; steps consumed: 63450 / 63450
EXACT MATCH: True


```

整条 63450 op 的 golden-trace 逐 opcode 全绿，a_bogus 逐字节等于 Node oracle，换任意 query 也逐字节相等。这是第二个、也是决定性的里程碑。

回头看这场马拉松，最值得记的一点是：golden-trace 把「逆一个未知算法」变成了「修一个有明确报错位置的解释器」。每个 bug 都有精确的 `(funcId, ip, op)` 坐标，不需要猜「哪里错了」，只需要看「这个 opcode 该怎么做」。这就是当初选「重写 VM」而非「硬解算法」的全部价值所在。

### 让 query / ts / rand 都变成自由输入

byte-exact 复现的是采集会话那套冻结值。要做成可用的签名器，得让输入自由。

query 天然自由，它就是 z[150] 的第 4 个参数，换字符串就行，带 msToken 的长真实 query 也逐字节校验通过。ts 稍费劲：改 Date 全局后只有一部分字节跟着变、另一部分还是旧值。用「抓两个不同 ts 的作用域快照做 diff」定位到 ts 只折叠在三处——Date 全局、作用域里的 ts 字面量（`frame[0][3]`）、以及 `navigator.vendorSubs.ink`（bdms 注入的时间戳蜜罐，实测公式恒为 `ts-1`）。`ABogus._retime` 直接改 JSON 快照里的 ts/ts-1 字面即可，多个 ts 值校验全过。

rand 一开始以为子树里没有，native 统计是 0。错了——插桩计数发现 sign 期间 `Math.random` 被调了 39 次（native 记成匿名 `function` 了），字母表是运行时用这 39 个随机值现算的。所以让 rand 自由极简单：`Interp(rand=...)` 接常量或零参 callable。用真随机 callable，每次 39 个不同值、每次一个唯一签名，跟真浏览器一样；拿「Node 和 Python 用同一确定性随机序列」校验，常量和序列都逐字节相等。至此：

```
from pyvm.a_bogus import ABogus
s = ABogus()
s.sign(query)                                       # 默认 now + 真随机 = 每次唯一
s.sign(query, ts=1700000000000, rand=0.123456789)   # 可复现


```

query / ts / rand 三输入全自由，全部对 Node 逐字节验证过。

九、校验浏览器，打真服务器
-------------

到这一步，需要证的其实只是「Python == bdms.js 在这套 stub 环境下的输出」。但真浏览器的指纹和 Node stub 不一样，这个等式还不够硬，所以做了两道更硬的验证。

### B 验证：和真浏览器指纹校验，撞上反 headless 天花板

第一道是把本地 bdms.js 路由进真 Chrome 加真 douyin，冻结 ts/rand，抓真浏览器的 a_bogus 加真指纹的作用域 / 全局快照，再让纯 Python 用这套真快照复现（`code/capture_browser.py`）。

结果是 86%（162/188）逐字符相等，而且 query 派生字节加整个尾段 100% 相等，26 个不同字符全部挤在反 headless 指纹区（位置 18-84 加尾校验）。期间修了几个真 bug，都是「数据快照没抓全真浏览器的活状态」。`Object.prototype.toString.call(navigator)` 真浏览器返 `[object Navigator]`，我返 `[object Object]`，而 z[730] 用它反 spoofing，得序列化时抓每个对象的 `[object X]` 类名。`navigator.userAgent` 的 getter 在 `Navigator.prototype` 上，但要求 `this` 是 navigator 实例，在原型上直接调抛 "Illegal invocation"。`document.createElement` 在 `Document.prototype` 上（HTMLDocument→Document 好几层），且 Document.prototype 自有属性超 120 个、createElement 排后面被截断，把上限提到 600 才抓全。

但剩下的 26 字节，最终没能全部抹平。原因是这些字节来自 bdms 的反自动化探针：`new Error().stack` 的引擎栈格式、canvas 元素身份、DOM 原型链身份、行为事件时序，它们读的是活的 JS/DOM 环境，一个纯数据快照抓不全。此外还注意到一个现象：同样冻结 ts/rand，两次独立浏览器会话的 a_bogus 也不完全一样——指纹本身就带跨会话的活状态成分。

这是用数据快照复现的天花板，不是算法缺口。处理「数据」的那部分（query、结构、尾段，也就是签名真正保护的内容）和真浏览器逐字节一致，差的全是活环境探针。当时的判断是：要 100% 撞上一个活浏览器，本质是再造一个浏览器，另一个量级，且没必要。这句「没必要」后来被 detail 打了脸，那是后话。

### A 验证：真 douyin 服务器接受，盖章

B 证的是算法对，但没证服务器认不认这套 stub 指纹。这个只有一个办法能证：拿纯 Python 的 a_bogus 直接打真 douyin 接口，看服务器接不接受（`code/verify_server_focused.py`）。

第一个坑是选错了端点。`hot/search/list` 这类光凭 cookie 就放行、根本不强制校验 a_bogus，带不带、篡改不篡改都返 200，没有鉴别力。改成截获 SPA 浏览时发出的所有带 a_bogus 的真实请求，逐个测「去掉 a_bogus 会不会失败」，找到一个强校验端点 `/aweme/v1/web/channel/hotspot`。在它上面连跑 3 次，每次用一个全新的纯 Python 签名（真随机加当前时间戳）：

<table><thead><tr><th>变体</th><th>真 douyin 服务器响应（3 次一致）</th></tr></thead><tbody><tr><td>不带 a_bogus</td><td>拒绝——空 body，len=0</td></tr><tr><td>a_bogus 篡改 1 个字符</td><td>拒绝——空 body，len=0</td></tr><tr><td>纯 Python a_bogus</td><td>接受——200, status_code=0, 返回 9-10 条真实视频 feed</td></tr></tbody></table>

三条信息叠在一起才有说服力：不带会被拒，说明这个端点真查 a_bogus；篡改会被拒，说明服务器校验的是 a_bogus 的内容、不是它存不存在；而每一个全新的纯 Python 签名都被接受并返回真实数据。纯 Python a_bogus，真 douyin 服务器端到端接受，盖章完成。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fJBlDTU8pLE90DoNQvzhesJMVADyZIvyX8S9DX5Yc3S4OPu39Fz3GlMQLNfP0FliagXdRWBCyS2MAfiaSm25dSuMUvNvVibibicV3ic6mcWian2970/640?from=appmsg&watermark=1#imgIndex=4)三层验证

_图：算法层逐字节、指纹层 86%、服务器层接受——三层各验证不同的东西。_

十、翻过反 headless 墙，攻破 aweme/detail
--------------------------------

上一章末尾留下一句「要 100% 撞上活浏览器没必要」。`aweme/detail`（视频详情接口）偏偏就查那段对不上的字节——`channel/hotspot` 盖了章、大多数端点都通了，唯独它，纯 Python 拿不到数据。这是整个 a_bogus 逆向里最硬的一关。结果出乎意料：不用再造浏览器也能翻过去，补十来个浏览器环境探针加硬编码 3 个「干净 Chrome 通过反检测」的常量就够了。

![](https://mmbiz.qpic.cn/mmbiz_png/fJBlDTU8pLEw0PbYiaZb5Saq2uqjSJhOZ31pI8ksVuFaWQic2IMricnSHVKDS5AyicH6wA1qVdaJQyly0rBEglgwYfYA9micg9oMCB4IFggPNEyw/640?from=appmsg&watermark=1#imgIndex=5)detail 攻破全景

_图：这一关的五步全景，也是下面五个小节的路线图。_

这一关比较长，先把路线摆出来，下面五个小节正是照它走：**第一步**证伪交接文档那套 verifyFp 理论，坐实「门就是 a_bogus 值本身」；**第二步**把差异从模糊的一片收窄到 13 个确定的指纹字节，并证明它们跨会话稳定；**第三步**拿真浏览器的 trace 当尺子，逐 op 把浏览器探针补出来；**第四步**补不平的那几位，用 3 个稳定常量直接注入；**第五步**打真服务器盖章，最后发现连浏览器快照都能省掉。记住这条线，中间任何一节读到一半，都能知道自己走到哪了。

### 第一步 · 门在哪：先证伪一个未验证的假设

现象很干净：同一套链路（ttwid 加纯 Python a_bogus 加随机 msToken），打 `comment/list`、`hot/search`、`channel/hotspot`、用户视频，全 `sc=0` 返真数据；唯独打 `/aweme/v1/web/aweme/detail/`，服务器返空 body、200（礼貌地拒绝）。而浏览器签的 a_bogus 打 detail 就能返数据。

我手上有一份别人的交接文档，它给了个理论：detail 交叉校验 a_bogus 内嵌的指纹段与请求里的 `verifyFp`/`uifid` 是否同源自洽，下一步计划是逆 `verifyFp` 从 `s_v_web_id` cookie 的派生。但我没有照着做，因为这个理论从没被实验坐实过。逆向里最容易浪费时间的，就是照着一个未验证的假设往下挖。所以第一件事不是逆 verifyFp，而是先证伪或坐实它。

思路很直接：如果 detail 真的交叉校验 a_bogus vs verifyFp，那拿一个有效的浏览器 a_bogus，去破坏 verifyFp（换错、删掉），detail 就该拒；反过来，如果无论怎么动 verifyFp 都照返数据，那 verifyFp 就和门无关。`detail_crosscheck.py` 就做这件事：一个浏览器会话里让页面 SDK 正常签一个 detail 请求，拦截拿到签好的完整 URL（含 a_bogus）加这个会话的全部 cookie，然后 a_bogus 保持不动，只变异其它一切，用 `requests` 回放。真值表（浏览器 a_bogus 全程不变）：

<table><thead><tr><th>变异</th><th>结果</th></tr></thead><tbody><tr><td>M0 原样回放</td><td>OK 返数据</td></tr><tr><td>M1 query 里 verifyFp/fp 换成不匹配的随机值</td><td>OK</td></tr><tr><td>M2 query 里删掉 verifyFp/fp</td><td>OK</td></tr><tr><td>M3 删掉 uifid</td><td>OK</td></tr><tr><td>M4 <code>s_v_web_id</code> cookie 随机化</td><td>OK</td></tr><tr><td>M5 只留 ttwid（连 s_v_web_id 都删）</td><td>OK</td></tr></tbody></table>

`verifyFp` / `fp` / `uifid` / `s_v_web_id` cookie 对 detail 完全无关。交接文档的整个理论方向，连同「逆 verifyFp 派生」的计划，是错的，作废。这正是上半程「盯值别盯代码」的孪生版：面对一个未验证的假设，先设计一个能证伪它的实验，别先照它施工。二十分钟的 `detail_crosscheck.py`，省掉了可能好几天的无用功。

证伪了 verifyFp，那门到底是什么？`detail_decisive.py` 把变量隔离到只剩一个：同一个浏览器会话里拿到浏览器签好的 detail URL，拆成三段 `path?<before>&a_bogus=<AB>&<after>`，其中 `<before>` 是 a_bogus 实际签的内容，`<after>` 是之后追加的 `verifyFp/fp/uifid/timestamp/x-secsdk-web-signature`，然后只换 a_bogus 一个变量回放：

<table><thead><tr><th>测试</th><th>a_bogus 来源</th><th>before/after/cookie</th><th>结果</th></tr></thead><tbody><tr><td>A 对照</td><td>浏览器</td><td>同会话原样</td><td>OK 返数据</td></tr><tr><td>B 决定性</td><td>pyvm 重签同一个 before</td><td>同会话原样</td><td>空 200</td></tr></tbody></table>

before-query 一样、verifyFp/uifid/cookie 全一样，只换 a_bogus 来源，浏览器过、pyvm 不过。门就是 a_bogus 这个值本身。顺手还纠正了一处认知：打印 `<before>` 的参数顺序发现 `a_bogus` 排在 `verifyFp/fp/uifid` 之前，所以 a_bogus 压根不签 verifyFp/uifid，它们是 a_bogus 之后才追加的，这从机制上也印证了前面的结论。

而 pyvm 的 a_bogus 在 `channel/hotspot`（同样强校验 a_bogus 的端点）是被接受的。所以 detail 一定对 a_bogus 做了 channel/hotspot 没有的额外校验——校验的正是上一章那 26 个活环境探针字节。

### 第二步 · 收窄到 13 个 query 无关的指纹字节

也许有个捷径：pyvm 用的是逆向时刻的旧指纹快照（`core_globals.json`），如果抓当前浏览器的真实指纹喂 pyvm 是不是就对了？`detail_snapshot_test.py` 试了：route 本地插桩 bdms（跑我们的字节码加真浏览器指纹），冻 ts+rand，抓当前浏览器的 `browser_scopechain.json`/`browser_globals.json`，喂 pyvm 签 detail query 回放——用当前浏览器快照签出的 a_bogus 与浏览器逐字符匹配 166/188，仍空 200。

即便用当前浏览器的真实指纹静态快照，还差 22 个字符，detail 照拒。这 22 和 B 验证那 26 是同一堵墙，只是不同会话、不同 query 的测量，都落在反 headless 活探针区。这坐实了「数据快照天花板」的判断——那些字节来自 z[150] 运行时读的活探针（`new Error().stack`、canvas 身份、DOM 原型链身份），不是静态快照能装下的。捷径堵死，只能把这些探针在 pyvm 里补出来。

动手补之前，先把敌人量化清楚：具体哪些字节差、差的字节是 query 相关还是 query 无关。`detail_byte_diff.py` 让浏览器冻 ts+rand 签两个不同 aweme_id 的 detail query，pyvm 用同一份浏览器快照签同样的 before，然后在 payload 层对比（不是 char 层）——把 188 字符的 a_bogus 用冻 rand=0.123456789 的自定义字母表 `Dkdpgh2ZmsQB80/MfvV36XI1R45-WUAlEixNLwoqYTOPuzKFjJnry79HbGcaStCe` 按标准 base64 解回 141 字节 payload，逐字节 diff。

两个 query 的 payload 差异都落在同 17 个位置：`[14,15,22,23, 27,28,31, 29,38,39,44,47,54,55,61,63, 140]`。其中两个 query 值相同的（即 query 无关、纯指纹）有 13 个：`[14,15,22,23,29,38,39,44,47,54,55,61,63]`；跨 query 变的是 `[27,28,31]`（query/msToken 派生）和 `[140]`（尾校验）。detail 的门就收窄成 13 个 query 无关的指纹 payload 字节，范围一下从「188 字节里那模糊的 26 个」精确到具体 13 个。「query 无关」这一半 `detail_byte_diff` 已经顺手证了——那两个 aweme_id 是同一个浏览器会话签的，13 字节两个 query 逐字节相同，说明它们不随 query 变。

还有个 make-or-break 的问题：跨不同浏览器会话，它们稳定吗？如果每次开浏览器都换一套（带某种 session nonce），那怎么补都白搭。于是另开一次浏览器（`capture_browser.py` 抓 hot/search 那次，和 detail 这次是不同时间、不同 query 的两次独立会话，同样冻 rand=0.123456789），解出同样 13 个位置的值，和这次 detail 会话对比：

```
detail 会话（本次）: {14:133,15:247,22:179,23:48,29:235,38:216,39:186,44:71,47:9,54:251,55:47,61:149,63:14}
另一独立会话       : 逐字节完全相同


```

13 字节既 query 无关、又跨会话逐字节相同，稳定。这是决定性的好消息：它们是确定性的「干净 Chrome 指纹」，不是每会话随机——能抓一次注入，甚至能硬编码。

### 第三步 · golden-trace 逐 op 补出浏览器探针

浏览器快照已经把 scope/globals 都对齐了，pyvm 却仍错 13 字节，说明这 13 字节来自 z[150] 执行期读的活 native 返回值，快照没装下。要定位就得逐个 native 对比。

好在上半程的解释器内建了 native 校验钩子 `Interp._nat_check`（本来用来对 Node 的 `core_natives.json`）。复用它，喂浏览器的 native trace 当 golden，pyvm 签同一个 before，报第一处返回值分歧（`capture_native_diff.py`）：

```
*** FIRST NATIVE DIVERGENCE @ call #6 ***
  expected(browser): createElement(["canvas"]) -> "__obj{}"
  got     (pyvm)   : createElement(["canvas"]) -> "__obj{width,height,toDataURL,getContext}"
  context #7 toString() -> "function toDataURL() { [native code] }"
          #8 indexOf(["[native code]"]) -> 23


```

读出来的意思是 bdms 做 canvas 反 spoof 检测：`document.createElement('canvas')` 后取 `canvas.toDataURL`，调 `.toString()`，`indexOf("[native code]")` 看是不是原生（防止 toDataURL 被 hook 篡改）。pyvm 的 canvas 桩把 `toDataURL/getContext/width/height` 挂成自有属性（序列化成 `{width,height,...}`），而真浏览器里它们在 `HTMLCanvasElement.prototype` 上（自有枚举键为空，序列化成 `{}`），这个结构差异让 bdms 的 `for-in`/`getOwnPropertyDescriptor`/`hasOwnProperty` 之类走了不同分支。这就是根：pyvm 的浏览器环境模拟不够真，探针一个个走偏，得逐个补。

要高效迭代，需要搭了一台专用 harness `trace_step.py`：pyvm 用浏览器指纹快照跑，对着浏览器的 opcode golden-trace（`browser_trace.json`，由 `capture_native_diff.py` 顺带抓下）逐 op 校验，一旦分歧就打印 `step / 浏览器[funcId,ip,op,p] vs pyvm[...]` 加反汇编分歧点附近加最终 a_bogus 字节匹配数。这条 trace 有 64339 op，和上半程 Node 签 channel query 的 63450-op core_trace 是两条不同的 trace（query 不同、运行环境不同、探针分支走法不一样），op 数对不上是正常的。剩下的就是「修一个探针 → 重跑 → 看分歧往后跳」，跟上半程「golden-trace 把逆算法变成修解释器」是同一套打法，只不过这次对的是真浏览器的 trace。

一路推进的实录，每步都是「trace_step 报分歧 → 反汇编看它查什么 → 在 `interp.py` 补上 → 分歧后移」：

1.  canvas 反 spoof（step 203 前）：`_make_element('canvas')` 把方法挂到原型上（自有枚举键为空，匹配浏览器 `{}`）；`_fn_prop` 给 `NativeFn.toString` 返 `"function <name>() { [native code] }"`。
2.  instanceof（op 13）加 PluginArray（z[733]，step 203）：反汇编看到 `typeof PluginArray!=='undefined' && navigator.plugins instanceof PluginArray`，而 pyvm 的 op13 是硬桩恒 False。补 op13 走原型链（`x.proto` 链里找 `C.prototype`）、`_make_globals` 加 `PluginArray`（带 klass 化 prototype）、`apply_globals` 把 `navigator.plugins` 强制设成 PluginArray 实例。
3.  window.eval（z[756]，step 233）：探针查 `window.eval` 存在，补上。
4.  一批浏览器全局构造器（z[744]，step 407）：查 `window.Audio` / `window.CanvasRenderingContext2D` 等一整排。补一批真 Chrome 确有的构造器（Audio/CanvasRenderingContext2D/WebGL*/RTCPeerConnection/MediaSource/Notification/Worker/WebSocket，加 URL/Blob/TextEncoder，每个带 klass 化 prototype 供 instanceof）。这一批清掉一大片探针，分歧一步从 407 跳到 26761。
5.  document.all = HTMLAllCollection（z[716]，step 26761）：经典反 headless——`document.all===undefined`（应 false）、`document.all.__proto__===HTMLAllCollection.prototype`、`document.all.toString()==='[object HTMLAllCollection]'`。pyvm 的 `document.all` 是 UNDEF，读 `.__proto__` 抛异常跳 catch。补 `HTMLAllCollection` 全局加注入 `document.all` 为其实例。
6.  访问器描述符（z[724]，step 27075）：`Object.getOwnPropertyDescriptor(window,'screen')` 真浏览器返访问器描述符 `{get,set,enumerable,configurable}`（screen 是 window 上的 getter），pyvm 返数据描述符 `{value,writable,...}`，迭代描述符键时分支不同。给 `JSObject` 加 `accessors` 集合，`get_desc` 对访问器键返 `{get,set,...}`，把 window 的 `navigator/screen/location/history/document/innerWidth` 标为访问器。

补完这 6 类，trace 推进到 step 27841，已经回到 z[150] 主签名函数本体（ip692），所有探针子函数全部通过，剩下的分歧在主函数内部。

### 第四步 · 注入 3 个反检测常量，trace 全绿

z[150] ip692 的分歧，反汇编是 `scope[37]['4'] & 2` 后 `JMP_FALSE`。`scope[37]` 是个 5 元素整数数组，是前面那批探针结果聚合成的反检测位域。dump 出来：

```
pyvm    scope[37] = [0,0,0,0,3]     # element[4]=3 (bit0+bit1)
browser scope[37] = [0,0,0,0,1]     # element[4]=1 (只 bit0)  —— capture_scope37.py 抓


```

pyvm 多检测出了 bit1（某个探针还没补到位）。与其继续逐位补，还不如直接注入这个稳定的位域（它 query/ts/rand 无关）：给解释器加 `scope_patch` 机制，在 `(funcId=150, ip=682)` 把 `s[37]` 替成 `[0,0,0,0,1]`。注入后再跑 trace_step，trace 走完 64339/64339 零分歧，且 `scope[19]`（32 字节指纹摘要）与浏览器完全一致。

但字节匹配只有 171/188。这是个反直觉却关键的现象：trace（控制流）全绿，输出字节却还差。因为 golden-trace 只记录 `[funcId,ip,op,p]`（走了哪些指令），不记录值，剩余差异来自「沿同一条控制流、但 native 返回值不同」。

trace 全绿意味着 native 序列现在逐个对齐，可以干净地逐 native 对比返回值。`native_align.py`（按 argc 对齐加注入 scope[37]）报出两处标量返回值差异。一是 getTime：浏览器返真实 2026 时间戳，pyvm 返冻结值。根因是捕获脚本的 bug——`capture_browser.py` 只冻了 `Date.now`，没冻 `new Date()` 构造器（正是前面提醒过的「冻结要冻全」）。修捕获脚本 override 整个 `Date` 构造器（`new Date()` 无参也返 ts），改完重抓一遍浏览器，两边 getTime 都成冻结值，到 172/188。二是 `document.all.toString()`：应返 `[object HTMLAllCollection]`，pyvm 返 `[object Object]`，因为 `x.toString()`（直接调）走的是对象默认 toString，没读 klass。修 `_obj_method` 的 toString 为 `"[object %s]" % (obj.klass or "Object")`，到 174/188。这两个都不是活时序，是确定性值，修完就稳。

这一步会冒出好几个 `scope[NN]`，先把关键的几个摆一下角色，免得被编号绕晕（它们都 query/ts/rand 无关）：

<table><thead><tr><th>slot</th><th>角色</th></tr></thead><tbody><tr><td><code>scope[8]</code></td><td>32 位环境检测值，小端拆成 payload 的 4 字节</td></tr><tr><td><code>scope[16]</code></td><td>反检测位域（z[699] 的输出），经 z[142] 拆成 3 字节进 payload</td></tr><tr><td><code>scope[37]</code></td><td>反检测位域（z[733] 聚合的 5 元素数组）</td></tr><tr><td><code>scope[19]</code></td><td>32 字节指纹摘要（trace 全绿时它已和浏览器一致）</td></tr></tbody></table>

剩下的指纹字节还来自别的槽。与其一个个 native 猜，更好的处理方式是系统化处理：dump z[150] 在接近 RET（ip1820）时整个 scope 帧（所有小数值槽），浏览器 vs pyvm 全量 diff（`capture_frame.py`）。排除下游哈希 / payload 槽后，根差异槽有三个。`scope[16] = 5`（pyvm 是 37），追进去是 z[699] 这个又一个反检测位域函数的输出（`scope[16]` 经 z[142] 拆成 3 字节 `[scope16>>8, scope16&255, scope17&255]` 进 payload）。`scope[8] = 6241`（pyvm 是 0），是一个 32 位环境值，被 z[150] 按小端拆成 4 字节（`6241 = 0x1861` → `[97, 24, 0, 0]`）：`scope[67]=6241&255=97 / scope[68]=(6241>>8)&255=24 / scope[69]=(6241>>16)&255=0`，最高字节也是 0。第三个是已知的 `scope[37] = [0,0,0,0,1]`。

逐个注入验证字节匹配收敛。注入点要落在写入之后、消费之前，且必须是合法 op 边界——我一度把 slot 8 注入在 ip970，结果那是 SCOPE_WRITE 指令的操作数中间、根本没触发，改 ip971 才生效。下表基线已含上面 getTime/toString 两处修复，看的是 3 个注入各自的增量：

<table><thead><tr><th>注入</th><th>字节匹配</th></tr></thead><tbody><tr><td>仅 scope[37]</td><td>174/188</td></tr><tr><td>+ scope[16] @(150,193)</td><td>181/188</td></tr><tr><td>+ scope[8]=6241 @(150,971)</td><td>186/188</td></tr></tbody></table>

剩余 payload 差异只有 `[27, 140]`。byte 27 来自 `slot 18 = SM3(SM3(query+"dhzx"))` 这个双 SM3 query 哈希，诡异的是 pyvm 算的是数学上正确的双 SM3、浏览器的值却不同（浏览器的 SM3 对象在两次 sum 之间的状态语义有微妙差别），但它是 query 派生字节，不是指纹。byte 140 是尾校验，payload 全对后自动解决。到这里停下来，问一个更重要的问题，而不是继续死磕这 2 字节。

### 第五步 · 盖章：detail 不查 byte 27，186/188 就够

字节精确不是目的，服务器接受才是。detail 校验的是那 13 个指纹字节，byte 27 是 query 哈希、byte 140 是校验和，如果这俩 detail 不 gate，186/188 就够。于是把 3 个注入产品化（`scope_patch = {(150,193):{16:5},(150,682):{37:[0,0,0,0,1]},(150,971):{8:6241}}`），fresh ts 加每次真随机签 detail query，注册 ttwid，打真 detail：

```
#1 OK-DATA desc='#睡眠音乐 #治愈心灵的音乐' digg=1046507
#2 OK-DATA  #3 OK-DATA        (连 3 次全新签名全过)


```

成了。detail 不查 byte 27，186/188（那 13 个指纹字节全对）就被真服务器接受。再从 `related` 拉 4 个真实当前视频逐个打 detail——白鹿工作室、剧影、曾舜晞、TF 家族，全 `sc=0` 返真数据，泛化性确证。纯 Python a_bogus，真 douyin 服务器 detail 端到端接受，任意视频。

### 收尾 · 其实连浏览器快照都不需要

到这一步用的还是当前浏览器抓的指纹快照（`browser_*.json`），那个「抓一次」的步骤还在。能不能连它都去掉？

一个念头：detail 检查的 13 字节稳定、机器无关（前面证过跨会话相同），而服务器接受的是「干净 Chrome」这个类别，不是某台特定机器（A 验证里连 stub 指纹都被 channel/hotspot 接受）。那用基础 ABogus 自带的 `core_*` 快照（逆向时烤死的 Mac Chrome 设备档常量）加 3 个注入常量，是不是就够？实测：core 快照加 3 注入常量，签 detail query 打真服务器——柠檬、新浪军事、念初、每日经济新闻，全 `sc=0` 返真数据。

结论是不需要任何浏览器捕获。a_bogus 全端点（含 detail）是一个完全自包含的纯 Python 算法：运行时零浏览器、零 Node、零指纹捕获，只靠烤死的常量（`core_*` 快照加 3 个反检测常量，像哈希的 IV 表）加 `interp.py` 的探针模拟。那 3 个注入是机器无关的规范常量（对 core 快照和真浏览器快照都有效、都被服务器接受），唯一约束是 query 的设备参数要和 core 档一致（Mac/Chrome/1366x900）。`code/capture_detail_fp.py` 仍保留，但只有做字节级研究、或换设备档 / 线上 bdms 轮换时才需要重抓，正常用不到。

既然带反检测的这套才是真正完整的 a_bogus（等价真浏览器输出、通吃全端点），命名也归位了：`ABogus`（`pyvm/a_bogus.py`）是真正的 a_bogus 算法，默认加载 3 个反检测常量，宽松 / 严格浅 / 严格深端点全部实测 `sc=0`（hot/search、comment/list、aweme/post、channel/hotspot、aweme/detail、related），生产统一用它；`ABogusCore`（继承基类、无注入）是 stub 指纹的 byte-exact 逆向参照，对 Node oracle 逐字节吻合，只作验证 / 研究，不打服务器；`ABogusDetail` 是 `ABogus` 的向后兼容别名。客户端 `douyin_client.py` 的 `DouyinClient` 用 `ABogus`，一个客户端零浏览器打通全部端点。

到这里，a_bogus 才算真正逆完：从「它到底在哪生成」，到「栈式 VM 逐字节复现」，到「channel/hotspot 强校验端点盖章」，再到「最硬的 aweme/detail 也被服务器接受」，全端点、纯 Python、零浏览器、零指纹捕获。

十一、最后一步：脱掉 VM，写成扁平纯算法
---------------------

> 至此，整个逆向分析已经接近结束，接下来只是将 vm 这套还原为纯算法，希望大家能有所收获。接下来的代码部分为付费内容，有需要的朋友可以付费查看。