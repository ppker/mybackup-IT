> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/bk4Vp5kT9av8aHvMENVsgQ)

> 这篇是笔者通过 AI 把抖音 web 的 `__ac_signature` 从头啃下来的过程记录。目标很直接：`window.byted_acrawler.sign("", nonce)` 吐出来的那串 `__ac_signature`（长这样 `_02B4Z6wo00f01vmnX1wAAIDD9wE8WneXenb5h1vAANtH2e`），笔者要用纯 Python 还原出来，而且对着线上真实 acrawler 逐字节对得上。
> 
> 免责声明：本文所涉及的内容仅供学习、交流，请勿将其用于非法用途！！

速览：这篇怎么读
--------

**一句话。** 抖音 web 每个接口都要一个叫 `__ac_signature` 的签名，它由页面里一段被虚拟机保护的 JS 生成。这篇要做的，就是把这段黑盒 JS 彻底看穿，用一个纯 Python 函数复现它，且对线上逐字节一致。

**它为什么难？** 三样叠一起：

① 算法不是明文 JS，是编译成字节码、跑在一台自定义虚拟机（JSVMP）上；

② 它要读浏览器指纹（canvas / WebGL / UA），脱离真浏览器根本算不出；

③ 时间戳过了雪崩哈希，输入动一丁点、输出就全变，没法靠猜。

**拨开虚拟机，其实很朴素。** 核心就是拿 `(时间戳, url, UA, nonce, 指纹)` 这几个数据，反复做一种逐字符的字符串哈希，再把几个哈希结果按位切一切、查一张自定义 base64 表，拼成 47 个字符。

**主线一条线，两条腿走：**

1.  **侦察地形**（第 2 章）——门在哪、字节今天有没有把入口换掉。
2.  **造一台可控签名机**（第 4 章）——把线上代码关进一个完全受控的浏览器，锁死时间和随机源，做到同输入必得同输出。这是后面一切差分的地基。
3.  **黑盒看输出**（第 5 章）——先不碰算法，光靠 "改一位输入、看哪几位输出跟着变"，把 47 个字符切成 7 段、认出编码方式。
4.  **白盒挖算术**（第 6 章）——黑盒到雪崩哈希就到头了，于是钻进虚拟机给它插桩、把每条指令执行时的栈拍成快照，再从快照里把哈希公式和被哈希的原文一个个抠出来。
5.  **拼起来验证**（第 7、9 章）——黑盒的结构 × 白盒的算术，一拼就是完整算法，最后跑 100 组逐字节对拍。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fJBlDTU8pLEbz3cvMAMb8RyNzx3XoPCKAWVtzic6sa87ZQcoicAUCaktkM4nNOmEB0TTXCVf4lZw3ProaRDMN7nSOWnPLJGZLtkWLHjmDvpds/640?from=appmsg&watermark=1#imgIndex=0)逆向方法论主线

整套路子一句话记住：**黑盒定结构（输出怎么排），白盒挖算术（每段怎么算）。**

0. 环境搭建
-------

```
# 1) Python 3.9+；建虚拟环境（可选）
python3 -m venv venv && source venv/bin/activate

# 2) 装 Playwright 并下载它自带的 Chromium
pip install playwright
playwright install chromium        # 下载浏览器内核到 ~/Library/Caches/ms-playwright

# 3) 本机若装了 Google Chrome，后面用 channel="chrome" 会更像真人（强烈建议）
ls "/Applications/Google Chrome.app" 2>/dev/null && echo "有 Chrome"

# 4) 建工作目录（本仓库对应 ac_signature/，脚本统一从该目录根运行）
mkdir -p ~/douyin/web/ac_signature/artifacts/scripts && cd ~/douyin/web/ac_signature


```

为什么非要真浏览器？因为 `__ac_signature` 的算法要读浏览器指纹（`navigator`/`canvas`/`WebGL`），这些只有真浏览器给得出真值。所以笔者这套思路不是 "把算法抠出来单独跑"，而是让线上代码在一个完全受控的浏览器里跑，当成一个黑盒签名机来用（喂 nonce、ts，吐签名）。

1. 目标：`__ac_signature` 是什么
--------------------------

抖音 web 的一大堆接口（首页、搜索、用户页、详情）都要 Cookie / 参数里带 `__ac_signature`。它的生成链路是这样：

```
服务端下发 __ac_nonce
        │
        ▼
window.byted_acrawler.sign("", __ac_nonce)   ← JSVMP 字节码，读 (ts, nonce, 浏览器指纹)
        │
        ▼
__ac_signature（47 字符，_02B4Z6wo00f01…） → 写回 Cookie → reload 进真实页面


```

难点在于算法不是明文 JS，而是被编译成字节码、跑在一台自定义虚拟机（JSVMP）上；指纹又深度参与哈希，没法简单 hook 出一个独立函数直接调。

2. 哪个 URL 直接吐 acrawler
----------------------

```
for u in https://www.iesdouyin.com/ https://www.douyin.com/user/self https://sso.douyin.com/; do
  echo "== $u =="; curl -sS -D - "$u" -o /tmp/p.html \
    -A "Mozilla/5.0 (…) Chrome/127.0.0.0 Safari/537.36" 2>&1 | grep -iE '^set-cookie|content-length'
  grep -oiE '__ac_nonce|byted_acrawler|_wafchallengeid' /tmp/p.html | sort -u
done


```

命中：

```
== https://www.douyin.com/user/self ==
content-length: 72914
set-cookie: __ac_nonce=06a671e43007373ac5f57; Path=/; Max-Age=1800; Secure; SameSite=None
set-cookie: __ac_nonce=06a671e43007373ac5f57; Path=/; Max-Age=1800
__ac_nonce
byted_acrawler


```

`/user/self` 直接 set 了 `__ac_nonce`，HTML 里也有 `byted_acrawler`——这就是签发 `__ac_signature` 的那个经典跳转页。

3. 目标解剖 + 抽取脚本
--------------

### 3.1 把 `/user/self` 抓全并劈出内联脚本

```
curl -sS "https://www.douyin.com/user/self" -o artifacts/user_self.html \
  -A "Mozilla/5.0 (…) Chrome/127.0.0.0 Safari/537.36"

# split_scripts.py —— 把 HTML 里所有 <script> 劈成独立文件
import re
html=open("artifacts/user_self.html",encoding="utf-8",errors="replace").read()
for i,(attrs,body) in enumerate(re.findall(r'<script\b([^>]*)>(.*?)</script>', html, re.S|re.I)):
    marks=[m for m in ["_$jsvmprt","byted_acrawler",".sign(","484e4f4a"] if m in body]
    open(f"artifacts/scripts/s{i:02d}.js","w").write(body)
    print(f"s{i:02d}.js  len={len(body):>7}  marks={marks}")

# 输出
s00.js  len=  71725  marks=['_$jsvmprt', '484e4f4a']     # JSVMP 解释器 + 字节码
s01.js  len=   1090  marks=['byted_acrawler', '.sign(']  # 胶水：init + sign + 写cookie + reload


```

### 3.2 胶水层 s01.js 印证调用链

```
window.byted_acrawler.init({aid:99999999,dfp:0});
var __ac_nonce = _f2("__ac_nonce"),
    __ac_signature = window.byted_acrawler.sign("", __ac_nonce);   // ← 核心调用
_f3("__ac_signature", __ac_signature);          // 写 cookie
window.location.reload();                        // 算完就刷新


```

### 3.3 核心 s00.js 确认是 JSVMP

```
import re
s=open("artifacts/scripts/s00.js").read()
h=re.findall(r'[0-9a-f]{500,}', s)[0]           # 唯一超长 hex
print("字节码:", len(h)//2, "字节")             # => 30179 字节
print("magic:", bytes.fromhex(h[:16]))          # => b'HNOJ@?RC'


```

`484e4f4a403f5243` = ASCII `HNOJ@?RC`，是 acrawler JSVMP 的固定特征。派发循环长这样（第 6 章要给它插桩）：

```
if(!I)for(;O<E;){var j=parseInt(""+b[O]+b[O+1],16);O+=2;var A=3&(x=13*j%241);…}
//  栈=S  栈指针=R  两级 2-bit 派发=13*j%241  字符串表 XOR 解码=r^i.p[P]


```

顺手数了下 s00.js 里的指纹关键词——`createElement/getContext/toDataURL/webgl/userAgent/[native code]` 全是 0。也就是说这份线上 acrawler 不自造环境，直接读真实浏览器。

### 3.4 路线确定

到此可以定性了：标准 JSVMP，超大文件 + 派发解释器 + 30KB 字节码 + magic 齐活。笔者不打算反编译字节码，两条腿走：

*   黑盒（第 4-5 章）：把线上 acrawler 跑成一个可控签名机，从签名差分出输出结构。
*   白盒（第 6 章）：给解释器插桩、栈快照，把哈希算术从字节码里挖出来。

整条主线五步（就是开头速览里那张方法论图），后面逐一展开。

4. 造签名机：让签名确定、可控、可复现
--------------------

要做差分和还原，前提是同输入必得同输出。线上直接跑有三个不确定性挡路：

① `Date.now()` 取当前时间；

② 可能用 `Math.random()`/`crypto.getRandomValues()`；

③ s01 算完立刻 `reload`，而且签名依赖 `location`（url 会进哈希），不能随便找个空白页跑。

笔者的解法是：拦截 douyin 那个 URL，只把 s00.js 挂在 douyin 域名下的一个极简页里跑，同时把随机源全锁死。

```
# oracle.py
import time
from playwright.sync_api import sync_playwright
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36"
S00=open("artifacts/scripts/s00.js").read()
PAGE=("<!doctype html><html><head><meta charset=utf-8></head>"
      "<body><canvas id=c></canvas><script>"+S00+"</script></body></html>")
INIT=r"""
// 可控时间戳
window.__T=1700000000000;
(function(){var R=Date;function F(a,b,c,d,e,f,g){switch(arguments.length){
 case 0:return new R(window.__T);case 1:return new R(a);case 2:return new R(a,b);
 case 3:return new R(a,b,c);case 4:return new R(a,b,c,d);case 5:return new R(a,b,c,d,e);
 case 6:return new R(a,b,c,d,e,f);default:return new R(a,b,c,d,e,f,g);}}
 F.now=function(){return window.__T;};F.parse=R.parse;F.UTC=R.UTC;F.prototype=R.prototype;window.Date=F;})();
// 可控随机数
Math.random=function(){return 0;};
try{if(window.crypto&&crypto.getRandomValues)crypto.getRandomValues=function(a){for(var i=0;i<a.length;i++)a[i]=0;return a;};}catch(e){}
"""
with sync_playwright() as p:
    br=p.chromium.launch(channel="chrome",headless=False,args=["--disable-blink-features=AutomationControlled"])
    ctx=br.new_context(user_agent=UA,locale="zh-CN",timezone_id="Asia/Shanghai")
    # 关键①：随机源确定化，页面脚本前生效
    ctx.add_init_script(INIT)
    # 关键②：拦截，返回自己的极简页
    ctx.route("https://www.douyin.com/user/self",
              lambda r:r.fulfill(status=200,content_type="text/html; charset=utf-8",body=PAGE))
    pg=ctx.new_page(); pg.goto("https://www.douyin.com/user/self",wait_until="load"); time.sleep(0.5)
    def sign(n,t): return pg.evaluate("([n,t])=>{window.__T=t;return window.byted_acrawler.sign('',n);}",[n,t])
    print(sign("06a61b9b600a388fc6b29",1700000000000))
    br.close()


```

三个点缺一不可：`channel="chrome"` 拿真人指纹，`ctx.route` 保住 douyin 的 `location`，`add_init_script` 让覆盖在页面脚本之前生效。

跑一批 `(nonce, ts)` 看看：

```
sign(06a61b9b600a388fc6b29, 1700000000000) = _02B4Z6wo00f01vmnX1wAAIDD9wE8WneXenb5h1vAANtH2e
sign(06a61b9b600a388fc6b29, 1700000001000) = _02B4Z6wo00f01RF.hLwAAIDAH9nnuolw7lkRX4AAACEO27
sign(06a61b9b600a388fc6b29, 1700000000000) = _02B4Z6wo00f01vmnX1wAAIDD9wE8WneXenb5h1vAANtH2e   ← 与第1条同
sign(0123456789abcdef01234, 1700000000000) = _02B4Z6wo00f01vmnX1wAAIDD9wE8WneWN2L5h1vAANtH5e
sign(ffffffffffffffffffff0, 1721000000000) = _02B4Z6wo00f015Ukl1QAAIDCm4L0UMMWz.eVBJPAAIPlb6


```

光这几条就能看出三件事：

① 可复现（第 1、3 条同输入完全相同）；

② ts 一动就雪崩（+1 秒几乎全变，说明 ts 过了哈希混淆）；

③ nonce 只影响局部（同 ts 换 nonce 只有后段变）。

5. 黑盒：看清输出长什么样
--------------

这一章不借任何旧资料，只用签名机的输入输出，测出 __ac_signature 这 47 个字符里谁是常量、谁随 ts、谁随 nonce，再用编码分析切出精确的字段边界。三步走：

① 先粗看哪几位在变（5.1）；

② 再认出编码是自定义 base64（5.2）；

③ 最后精确切成 7 段（5.3）。

### 5.1 第一步：差分出大致结构

采两组样本：固定 nonce 变 ts（6 个 ts）、固定 ts 变 nonce（6 个 nonce）。逐列看哪些字符会变：

```
def diffcols(sigs):
    # 返回"会变"的列下标集合
    return {i for i in range(len(sigs[0])) if len({s[i] for s in sigs})>1}
# 随 ts 变的列
tsc = diffcols(ts变的6条)   
# 随 nonce 变的列
nc  = diffcols(nonce变的6条)
kind="".join("B" if i in tsc&nc else "T" if i in tsc else "N" if i in nc else "C" for i in range(47))


```

真实输出：

```
idx : 01234567890123456789012345678901234567890123456
sig : _02B4Z6wo00f01vmnX1wAAIDD9wE8WneXenb5h1vAANtH2e
kind: CCCCCCCCCCCCCCTTTTTTCCCCTTTTTTTTBBBBTTTTCCTTTBB     (C常量 T随ts N随nonce B两者)

连续段：
  [ 0:14] '_02B4Z6wo00f01'  常量        ← 版本+固定前缀
  [14:20] 'vmnX1w'          随ts
  [20:24] 'AAID'            常量  ← ★ 内嵌常量：说明这里是某个"固定高位"透出来的
  [24:32] 'D9wE8Wne'        随ts
  [32:36] 'Xenb'            ts+nonce
  [36:40] '5h1v'            随ts
  [40:42] 'AA'              常量  ← ★ 又一处固定高位
  [42:45] 'NtH'             随ts
  [45:47] '2e'              ts+nonce      ← 尾（依赖前面全部）


```

大致结构出来了：常量前缀 | 一大片 ts 区 | 中间掺了 nonce 的区 | 一个依赖全部的尾。

值得单独记一笔的是 `AAID`、`AA` 这种字段内部的常量子串。它们其实是某个固定高位比特透出来的信号——后面会看到 `AAID` 正是 64 位里那个固定头 `8240` 的高位，`AA` 则是 `h_url%65521` 的值恒小于 65521、高位永远是 0，编码出来就是 base64 的 `A`。以后再看到字段里嵌着一段死活不变的常量，第一反应就该是 "这里藏了个固定的高位常量"。

差分能给的也就到粗地形为止了。比如 ts 区里 10 个字符全在变，你看不出它其实是 5+5 两段。要精确到字段，得靠下一步的编码分析。

### 5.2 第二步：认出编码为自定义 base64（值→字符）

把大量签名里出现过的字符去重统计，恰好 64 种（`A-Z a-z 0-9` 加两个符号）——这基本就是 6 位一组的 base64 系编码了。不过 "64 种字符" 只给出字符集合，不给索引顺序（谁是 0 谁是 63）。顺序这么定：

*   末两位是哪两个符号，观测直接给答案。把所有签名出现过的字符去重列出来，那 64 种恰好是 `A-Za-z0-9` 外加 `-` 和 `.`——没有 `+`、`/`、`_`。所以字母表末两位就是 `-` 和 `.`；`+/`、`-_` 那两套字母表直接排除（一个含 `.` 的签名根本不可能由它们产出）。
    
*   前 62 个用标准 base64 顺序 `A-Za-z0-9`（`A=0…Z=25, a=26…z=51, 0=52…9=61`），几乎所有 base64 变体都这么约定。
    
*   只剩一个自由度：`-` 和 `.` 谁是 62 谁是 63。拿一条含 `.` 的签名一判就定——用 §4 里 `ts=1700000001000` 那条 `_02B4Z6wo00f01RF.hLwAA…`，它的 V0 = `sig[14:19]` = `RF.hL`（正好带 `.`）。这条 ts 的内部 `v3l>>2 = 286783563`（管线算出来的，见 §6）。两种顺序各 `dec` 一次比对：
    
    ```
    dec('RF.hL',  '-'=62 '.'=63) = 286783563  == v3l>>2 ?  True    ← 只有这套成立
    dec('RF.hL',  '.'=62 '-'=63) = 286779467  == v3l>>2 ?  False
    
    
    ```
    
    于是第 62 位是`-`，第 63 位是`.`（注意不是 URL-safe 的`_`）。第 6 章从 VM 的字符串表 / 编码闭包里也能直接读到同一张表，正好双向印证。
    

```
ALPH = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-."
#        0..25 大写      26..51 小写         52..61 数字      62='-' 63='.'


```

怎么确认它是「值→字符，每 6 位一组」，而不是标准 `base64(字节数组)`？两个判据：

*   前缀后首个变化字符会按字母表顺序递变——固定 nonce 微调 ts，某段最低位字符沿 `…9-.` 顺序进位，说明是「整数低 6 位查表」。
*   段长和位宽自洽——常量前缀后是 5、5、6、5、5、5 共 31 字符的 body；`5*6=30`、`6*6=36` 位，恰好是 30/30/36/30/30/30 位的 6 个字段（为什么正好这么切，§6.5 反解坐实）。

```
def enc(v, nb):    # 整数 v → nb 位，高到低每 6 位一个字母
    return "".join(ALPH[(v >> (nb-6-6*i)) & 63] for i in range(nb//6))
def dec(s):        # 逆：字符串 → 整数
    v=0
    for c in s: v=(v<<6)|ALPH.index(c)
    return v


```

### 5.3 第三步：切出精确字段（14 + 5+5 + 6 + 5+5 + 5 + 2）

把粗地形按 base64 的 6 位边界对齐，就是最终字段图：

```
_02B4Z6wo00f01 │ V0  V1 │  V2  │ V3  V4 │ V5 │ tail
└── PREFIX(14)─┘  (5+5)   (6)    (5+5)   (5)   (2)
                  30+30    36     30+30   30    校验     14+5+5+6+5+5+5+2 = 47
def seg(sig):
    return dict(V0=sig[14:19],V1=sig[19:24],V2=sig[24:30],
                V3=sig[30:35],V4=sig[35:40],V5=sig[40:45],tail=sig[45:47])


```

对上大致结构：V0V1 是 ts 区（其中 V1=`sig[19:24]`=`wAAID` 透出固定头 `8240` 的高位，所以内嵌 `AAID`），V2 是指纹区（后面会证是 `FP_CONST ⊕ v3l`），V3、V4 是掺 nonce 区，V5 因为 `h_url` 是小值所以 `AA` 开头，tail 依赖全部。字段结构测绘到此为止，但每个字段 "具体怎么算" 还是黑盒——交给第 6 章。

6. 白盒：把算术从字节码里读出来
-----------------

黑盒到这就到头了：ts 是雪崩哈希（+1 秒全变），没法逆推。想拿到算法，只能进 VM 看内部算术。办法是给解释器的派发循环插桩，逐 opcode 快照栈，再让 Python 从快照里零假设认出哈希。

这一章是全篇最长的，先给张分步地图，免得读着读着迷路——六小步，一步一个产出：

*   **6.0** 在 71KB 混淆里找到该下手插桩的地方；
*   **6.1** 插桩，把每条指令执行时的栈拍成快照；
*   **6.2** 从快照里反解出哈希用的乘子（不预设、纯观测）；
*   **6.3** 顺着哈希链，把 "到底哪些字符串被哈希了" 一条条读出来；
*   **6.4** 把整条管线拼出来；
*   **6.5** 收最后一环——那些内部大整数是怎么切成签名字段的。

读不动的时候，每小节看开头一两句结论就够，公式和代码细节可以回头再抠。

### 6.0 先在 71KB 混淆里找到插桩点

动手插桩前先回答四个问题：解释器在哪、栈 / 指令指针叫什么、哪里会切走优化路径、关键常量是不是明文。全程 grep，不用读懂整个文件。

**Q1 · 确认里面有解释器，字节码是数据。** 搜 `_$jsvmprt`：

```
import re
s=open("artifacts/scripts/s00.js").read()
print([m.start() for m in re.finditer(r'_\$jsvmprt', s)])          # => [56, 9860]


```

两处：`56` 是 `_$jsvmprt=function(b,e,f){…}`（定义，也就是解释器本体），`9860` 是调用 `_$jsvmprt("484e4f4a…30KB hex…", [natives])`（第 1 参是 hex 字节码 / 数据，第 2 参是 natives 表）。要插桩的就是那个 function。

**Q2 · 找派发循环，认出栈。** JSVMP 派发循环的通用长相：一个 `for` 每轮读 2 个 hex 字符算出 opcode。搜 `parseInt(…16)`：

```
print([m.start() for m in re.finditer(r'parseInt\("\"\+b\[O\]\+b\[O\+1\],16\)', s)])   # => [2812]
print(s[2790:2960])
w,S=[],R=0; … var x,z,O=e,E=O+2*f;
if(!I)for(;O<E;){var j=parseInt(""+b[O]+b[O+1],16);O+=2;var A=3&(x=13*j%241); … S[++R]=… C=S[R--] …}


```

一眼就能对号：`S=[]` 是栈、`R` 是栈指针（满屏 `S[++R]`/`S[R--]`）、`O` 是指令指针（字节偏移，`O+=2` 每次吃 2 hex）、`j` 是当前 opcode、`13*j%241` 是两级派发。锚点就定在 `var j=parseInt(""+b[O]+b[O+1],16);O+=2;`。

**Q3 · 找优化路径 I 及其切换点，不然会漏抓循环。** 派发被 `if(!I)for(...)` 包着，那必然有条镜像的 `if(I)for(...)`：

```
print(s[s.find('j=B[O]')-16:s.find('j=B[O]')+40])   # if(I)for(;O<E;){j=B[O];O+=2; … }  @6273


```

也就是说 VM 有两套等价派发：`!I`（每次现解析 hex，慢）和 `I`（直接读预解码字节数组 `B` 里的 opcode，快）。字符级哈希是紧循环，也就是反复后向跳转，VM 会切到 I 路径优化——你只插桩 hex 锚点的话，哈希每一轮迭代全漏掉。所以得先禁掉切换点，逼它全程走 hex。搜 `I=1`：

```
print([m.start() for m in re.finditer(r'I=1,F\(b,e,2\*f\),O\+=2\*z-2;break', s)])   # => [4705, 5119]


```

两处后向跳转（`s(b,O)<0` 即跳转偏移为负时 `I=1,F(...)` 切 I 路径），把 `I=1,F(b,e,2*f),O+=2*z-2;break` 换成 `O+=2*z-2;continue`（留在 hex 循环继续）。再加上 K 里第二次调用的切换点（`B[e]?G(…,1):G(…,0)` → 强制走 `G(…,0)`）。

**Q4 · 确认关键常量不是明文，所以非动态挖不可。**

```
print(s.count('65599'), s.count('65521'))   # => 0 0


```

乘子、模都不是明文，藏在字节码常量表里，运行时用 `r^i.p[P]` XOR 解出来。这就是为什么必须栈快照——常量和算术只在运行时现形。

四问答完，插桩点全部定位（锚点 + 两处 I 切换 + K 切换），进 6.1。

### 6.1 给派发循环插桩

按 6.0 定位到的点，先禁掉 I 路径（3 处替换）逼全程走 hex，再在锚点后注入栈快照。

```
s00=open("artifacts/scripts/s00.js").read()
# ① 禁后向跳转切 I 路径（2 处）
s00=s00.replace("I=1,F(b,e,2*f),O+=2*z-2;break", "O+=2*z-2;continue")
# ② 禁 K 里第二次调用切 I 路径
s00=s00.replace("return i&&!B[e]&&F(b,e,2*f),B[e]?G(b,e,f,0,o,c,null,1)[1]:G(b,e,f,0,o,c,null,0)[1]",
                "return G(b,e,f,0,o,c,null,0)[1]")
# ③ 锚点后注入：记录 [opcode j, 指令偏移, 栈指针R, 栈顶S[R], 次顶S[R-1]]
anchor='var j=parseInt(""+b[O]+b[O+1],16);O+=2;'
s00=s00.replace(anchor, anchor+
  'if(glb.__tr){var _a=S[R],_b=S[R-1];glb.__tr.push([j,O-2,R,'
  '(typeof _a==="number"?_a:null),(typeof _b==="number"?_b:null)]);}', 1)


```

把这段 s00 套进第 4 章的签名机（`glb` 就是签名机页面的全局 `window`；签名前先 `glb.__tr=[]`，跑完再读回 `window.__tr`），跑一次 `sign("06a61b9b600a388fc6b29", 1700000000000)`，签完把 `window.__tr` dump 成 `artifacts/hash_trace.json`——这整套 "插桩 + 跑 + 导出" 就是 `code/trace_hash.py`（轨迹每行 = `[opcode j, 偏移O, 栈指针R, S[R], S[R-1]]`）：

```
sig (no-trace) : _02B4Z6wo00f01vmnX1wAAIDD9wE8WneXenb5h1vAANtH2e
sig (traced)   : _02B4Z6wo00f01vmnX1wAAIDD9wE8WneXenb5h1vAANtH2e  opcodes: 10209
禁I路径后签名不变 & 插桩不改签名: True


```

禁 I 路径后签名逐字节不变，反过来也证明了 I 路径就是纯优化，笔者只需要实现 hex 路径就够了。整套「禁快路 → 锚点拍栈快照 → 出轨迹」的流程如下：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fJBlDTU8pLE9k7UsZicichj2Z3jZBTTqTcAtbBVFTJ97vxDUwibzQJjoIicnHQbbzZPTmdHnoUnEkUW4W34IDMyMSNqsH02iaibPMibCcccH7yYLYo/640?from=appmsg&watermark=1#imgIndex=1)JSVMP 插桩流程

### 6.2 零假设①：反解哈希乘子 K

先把轨迹读进来，抽出「每 opcode 的栈顶序列 `tops`」和「所有数值栈值 `allvals`」（后面都靠它俩）：

```
import json
from collections import Counter
MASK=0xFFFFFFFF
trace=json.load(open("artifacts/hash_trace.json"))["trace"]   # 每行 = [opcode j, 偏移O, R, S[R], S[R-1]]

tops=[]        # 每 opcode 的栈顶 S[R]（非数值记 None）
allvals=[]     # 所有数值栈值（S[R] 与 S[R-1]）
for j,O,R,a,b in trace:
    tops.append(int(a)&MASK if isinstance(a,(int,float)) and float(a).is_integer() else None)
    for v in (a,b):
        if isinstance(v,(int,float)) and float(v).is_integer(): allvals.append(int(v)&MASK)


```

哈希如果是「逐字符 `h = f(h, c)`」，中间态会反复压栈。两条线索去认它。

一是数魔数——乘子 / 模会作为操作数反复压栈，统计高频大常量：

```
for v,n in Counter(allvals).most_common():
    if v>1000 and n>=5: print(v, "x", n)
# 输出：
#   65599  x263   ← 反复出现的大常量，疑似乘子
#   8240   x6     ← 疑似固定高位头
#   65521  x5     ← 疑似模


```

这三个数后面反复冒头，先混个脸熟：`65599` 是 SDBM 系那个经典字符串哈希乘子；`65521` 是最大的 16 位质数，acrawler 老拿它当模，把 32 位哈希压回 16 位塞进小槽；`8240` 是个固定往高位填的头。各自的身份后面会一一坐实。

二是解方程反解 K——设 `h_{i+1} = ((h_i ^ c) * K) & MASK`，c 是可打印字符。相邻栈顶 `(a,b)`、对每个 c 解 `K = b · inv(a^c) mod 2³²`（`inv` 是模 2³² 逆元，仅当 `a^c` 为奇数才存在）。先看天真版怎么翻车：

```
def inv(a):                       # a 的模 2^32 逆（a 必奇），牛顿迭代 6 次足够覆盖 32 位
    x=1
    for _ in range(6): x=(x*(2-a*x))&MASK
    return x

kc=Counter()
vals=[v for v in tops if v is not None]
for i in range(len(vals)-1):
    a,b=vals[i],vals[i+1]
    for c in range(32,127):
        t=(a^c)&MASK
        if t&1: kc[(b*inv(t))&MASK]+=1
print(kc.most_common(3))
# => [(0, 21964), (1, 313), (2863311531, 221)]   ← K=0 刷屏！


```

翻车的原因是栈里全是 `0`：`b=0` 时对任意 `a` 都解出 `K=0`，真值被淹了。修一下——只取 `a,b` 都是大值（真哈希态）的配对，把退化的排掉：

```
kc=Counter()
seq=[(i,v) for i,v in enumerate(tops) if v is not None and v>0xFFFF]   # 只留大值
for idx in range(len(seq)-1):
    i,a=seq[idx]
    for jdx in range(idx+1, min(idx+6, len(seq))):     # 向后看 5 个
        k,b=seq[jdx]
        if k-i>30: break
        for c in range(32,127):
            t=(a^c)&MASK
            if t&1: kc[(b*inv(t))&MASK]+=1
print(kc.most_common(2))
# => [(65599, 482), (1, 276)]   ← K=65599 碾压


```

两条线索都指向 K = 65599（SDBM/gawk 系那个经典字符串哈希乘子）。乘子是从轨迹里零假设反解出来的，没用任何先验。

### 6.3 零假设②：让链条自己拼出被哈希的字符串

有了 K，就能顺着 `h_{i+1}=((h_i^c)*K)&MASK` 在栈顶序列里贪心找最长链，把每一步的 c 读出来——它们会拼成被哈希的原文：

```
K=65599
def extend(h, pos):               # 从值 h @ pos 贪心延伸最长可打印链
    s=""
    while True:
        nxt=None
        for c in range(32,127):
            want=((h^c)*K)&MASK
            for k in range(pos+1, min(pos+50,len(tops))):   # 近处找 want
                if tops[k]==want: nxt=(c,k,want); break
            if nxt: break
        if not nxt: break
        c,k,want=nxt; s+=chr(c); h=want; pos=k
    return s,h

# 驱动：从每个位置都试着起一条链，收 >=4 的、去重，再滤掉"是更长链后缀"的碎片
chains=[]; seen=set()
for i,v in enumerate(tops):
    if v is None: continue
    st,end=extend(v,i)
    if len(st)>=4 and (v,st) not in seen:
        seen.add((v,st)); chains.append((v,st,end))
chains.sort(key=lambda x:-len(x[1]))
final=[c for c in chains if not any(c[1]!=d[1] and c[1] in d[1] for d in chains)]
for v,st,end in final: print(f"seed={v:>10}  {st!r}  -> {end}")


```

挖出的全部长链（每条都过 `H(seed,s)==end` 校验；`seed` 带 ±1 的碎片是贪心起点的噪声，取最长的看）：

```
seed=          0  '1700000000www.douyin.com/user/self'  -> 1349395609  ✓
seed=          ?  '35393725126615'                       -> 2642226918  ✓
seed= 2642226918  'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like …'  ✓


```

读出来的这几条直接把整条管线暴露了：

1.  `seed=0 → "1700000000www.douyin.com/user/self"`：这是 `str(ts_s)` 紧接着 `host+pathname`。第二段哈希拿第一段结果当种子，链条从 0 一路走完，等价 `H(0, str(ts_s)+url)`——这就当场证明了链式种子，也直接读出 url = `location.host + location.pathname`（`www.douyin.com` + `/user/self`）。
2.  `"35393725126615"` = `str(v3)`，其中 `v3 = (8240<<32)|low`（正是 5.1 里那个固定高位 `8240`），它的哈希是 `h2 = 2642226918`。
3.  `seed=2642226918("=h2") → "Mozilla/5.0 (Macintosh; …"`：h_ua 用 h2 当种子，哈希的是带 `Mozilla/` 前缀的完整 UA。（nonce 也从 h2 出发，同理。）

这套「反解乘子 → 顺链读原文」画成图是这样：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fJBlDTU8pLFy6mM6qwH9RpLeHEBy8Fl6gXA1icq5ZBN7jE8OqvliaDYZtTYDkBXKwI99F7nffaWqibyPhYWNUxjIvy2YLbiblDREI4CuCnibp0Mg/640?from=appmsg&watermark=1#imgIndex=2)从栈快照反解哈希链

### 6.4 把管线拼出来（全部来自轨迹）

```
ts_s   = ts_ms // 1000
h_ts   = H(0, str(ts_s))                     # 链①前半
h_url  = H(h_ts, host + pathname)            # 链①后半（种子=h_ts）→ 轨迹读出 1349395609
low    = ts_s ^ (h_url % 65521) * 65521      # 构造见下方反解
v3     = (8240 << 32) | low                  # 高位头 8240（轨迹里的常量 + str(v3) 链②）
h2     = H(0, str(v3))                        # 链②终值 2642226918
h_ua   = H(h2, 完整UA)                        # 链③（种子=h2）
h_non  = H(h2, nonce)                        # 链④（种子=h2）
v3i    = ((h_ua % 65521) << 16) | (h_non % 65521)   # h_ua/h_non 各取 %65521 后高低 16 位拼（拼法在 §6.5 反解）


```

轨迹只直接给出 `str(v3)="35393725126615"`（即 `v3`，拆出 `v3h=8240`、`v3l=low`），但 `low` 是怎么来的？黑盒反解交叉验证一下：`v3l`、`h_url%65521` 都已知（前者是 v3 低 32 位，后者是 V5），代入 `ts_s' = v3l ^ (h_url%65521)*65521`：

```
# ts=1700000000000 -> ts_s' = 1700000000   （精确等于真 ts_s！）
# ts=1721000000000 -> ts_s' = 1721000000


```

精确还原出真 ts_s，说明 `low = ts_s ^ (h_url%65521)*65521`、`ts_s=ts_ms//1000`、头 `8240` 全都对。到这，核心哈希 `H(seed,s)=((h^c)*65599)&0xFFFFFFFF`、链式管线、所有输入字符串，全从 VM 挖出来了。整条管线（含每步真实中间值）如下：

![](https://mmbiz.qpic.cn/mmbiz_png/fJBlDTU8pLE6Xq5SgjwdPy3sWgxAUd8YTSHjF2JlsYshS0ibB2sibyxCufw0vXYF1CBYkfPY1E2tXC1aYp9NrQTLsBItJSmCqQ4Ke7HsV89ls/640?from=appmsg&watermark=1#imgIndex=3)哈希管线数据流

### 6.5 反解字段公式：内部大整数怎么切进 body

还差最后一环：管线算出 `v3(=v3l|v3h<<32)`、`v3i`、`h_url` 这些内部大整数，它们怎么变成 V0..V5 的？拿一条已知签名，`dec` 出每个字段的整数，直接和内部值比对，比特切法就现形了。用 §4 那条 `sign("06a61b9b600a388fc6b29",1700000000000)`（内部 `v3l=0xbe69d7d7, v3h=8240, v3i=0x9de5de9d, h_url%65521=56135`）：

```
dec(V0)=798651893   == v3l>>2                       ✓  → V0 = enc(v3l>>2, 30)           # v3l 高30位
dec(V1)=805306883   == ((v3l&3)<<28)|(v3h>>4)       ✓  → V1 = enc(((v3l&3)<<28)|(v3h>>4),30)
dec(V2)=4257238806  ,  dec(V2) ^ v3l = 1135188161   →   V2 = enc(FP_CONST ^ v3l, 36)     # 那个常数=FP_CONST(第8章)
dec(V3)=662271911   == v3i>>2                        ✓  → V3 = enc(v3i>>2, 30)
dec(V5)=56135       == h_url%65521                   ✓  → V5 = enc(h_url%65521, 30)
dec(V4)=468065647   == (((v3i&0xF)<<28)|((524576^v3l)>>4)) & 0x3FFFFFFF   ✓  → V4（见下注）


```

*   `v3i` 的拼法也是这么反解的（上面 V3/V4 里把它当已知用了，这里补它怎么来）：`v3i` 由 `dec(V3)<<2`（低 2 位从 V4 补）得到 `= 0x9de5de9d`；拆成高低 16 位，和 6.3 读出的 `h_ua/h_non`（速查表：`h_ua=614889485`、`h_non=207627517`）取模比对，一拍即合——
    
    ```
    v3i = 0x9de5de9d
    v3i >> 16          # = 0x9de5 = 40421 == h_ua % 65521   (614889485 % 65521 = 40421)  ✓
    v3i & 0xFFFF       # = 0xde9d = 56989 == h_non % 65521   (207627517 % 65521 = 56989)  ✓
    
    
    ```
    
    于是坐实`v3i = ((h_ua % 65521) << 16) | (h_non % 65521)`。（和 `low`、`h_url%65521`一样，`%65521` 是 acrawler 把 32 位哈希压进 16 位槽的惯用手法。）
    
*   V0/V1/V3/V5 一比即中；V2 的 `dec(V2)^v3l` 得到一个跨所有样本恒定的数，那就是折叠后的指纹常量 `FP_CONST`（第 8 章反解它）。
    
*   V4 有个坑：公式值 `((v3i&0xF)<<28)|…` 高位会溢出 30 位，而 `enc(·,30)` 只编码低 30 位，得 `& 0x3FFFFFFF` 才等于 `dec(V4)`。这也印证了字段就是「比特流切片」——大整数首尾拼成连续比特流再按 30/36 位切，边界不落在整值上（`524576=2049*256+32` 这个 V4 里的固定异或项，也是这么反解＋跨样本恒定确认的）。
    

这套「比特流切片」画成图就一目了然——为什么会冒出 `>>2`、`<<28` 这种怪移位：

![](https://mmbiz.qpic.cn/mmbiz_png/fJBlDTU8pLGnmVAmvgor0ibHaLddia9s0qibU6x15nn9tb9JibhbGs69j6ynTFC7YvibeSoOBb8ia9yGEdWzibbMzdJRYtrj1EcUxSx0RLia3DbUYdg/640?from=appmsg&watermark=1#imgIndex=4)比特流切片

到这里，输出结构（第 5 章）× 内部算术（第 6 章）拼完了，第 7 章直接组装成完整函数。

7. 完整算法
-------

> 至此，逆向分析已经全部结束了，希望大家能学习到新的知识。代码部分为付费内容，有需要的朋友可以付费查看。