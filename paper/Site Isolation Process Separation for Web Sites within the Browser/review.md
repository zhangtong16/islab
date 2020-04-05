
## 摘要
浏览器将多个网站的渲染放在同一个进程中进行，无法防御微架构攻击。

由于网页程序日益复杂，为了防止攻击者通过浏览器安装恶意软件，浏览器通常将不受信的内容放到一个或多个低权限的沙盒中执行，
但这种方式将从多个站点获取的数据存放在同一个渲染进程内，**即使在沙盒内部也可能存在敏感数据**，无法防止攻击者对敏感数据的窃取。
因此浏览器需要更严格的划分和隔离。


## 威胁模型
攻击者能够利用iframe或popup，使存在恶意代码的网站被加载到与其他网站相同的渲染进程中，浏览器通过渲染进程中的安全检查进行站点的隔离。
本文中的威胁模型考虑两种不同的攻击方式：
1. **渲染器漏洞攻击**：攻击者利用渲染进程的漏洞，获取在渲染进程执行任意代码的权限。由此进行如伪造IPC请求隐私数据的攻击。此时，要求浏览器的其他进程完全信任渲染进程。
2. **内存泄露攻击**：攻击者无法执行任意代码，但能够获得渲染进程地址空间内的所有数据，如利用meltdown或spectre等攻击。这种攻击不需要依赖渲染器的漏洞。

本文考虑将敏感数据从与principals相关的执行环境上下文 (包括documents和worker等) 中分隔开。
保护html和获取的json和xml文件，以及保存的cookie等状态信息。
## 架构
本文提出的解决方案主要分为两点：
- Site-Dedicated Processes  
Site-Dedicated Processes的目的是为每个独特的站点分配一个渲染进程，以此来在沙盒中对多个站点进行隔离。其中，需要注意的问题包括：
1. 划分的依据。本文中以site而非origin为依据进行划分，`document.domain`改变时site不会变化。
2. 区分site中需要保护的数据，以及浏览器进程中需要保护的数据。
3. 站点切换时需要保存的执行环境，以及这些环境改变的时间节点。
4. 其他收到影响的因素。
- Cross-Origin Read Blocking (CORB)  
CORB的主要目的是区分能够被跨站点访问的数据，例如script，image等。本文认为这些资源文件中不应该存在敏感信息。而对于HTML，XML和JSON等资源，文中认为需要被保护。但在这些资源文件有时难以区分，例如js代码可能被标记为`text/html`而非`application/javascript`。对此，作者建议使用CORB标准，筛选出包括HTML，XML和JSON的3类资源，且使用*confirmation sniffing*搜索response的某个前缀，确保其类型准确无误。  

除了以上两点内容，针对第一类攻击者，本文考虑了低权限的沙盒与高权限的浏览器进程的通信过程，捕获恶意IPC信息防止攻击者伪造对敏感信息的访问。(?)

## 评估
针对上述评估内容，评估结果如下
1. 解决的问题
   - 通过Site Isolation，以下问题得到了解决：
     - Authentication
     - Cross-origin messaging
     - Anti-clickjacking
     - Keeping data confidential
     - Storage and permissions
   - 以下问题可以解决，但还没有实现
     - Anti-CSRF
     - Embedding untrusted documents
   - 本文方案实现之后还可能出现的问题
     - Bypassing Site Isolation
     - Targeting non-isolated data
     - Cross-process attacks
2. 缓解瞬时执行攻击
   - 作者认为，现有的例如禁言SharedArrayBuffers或高精度计时器的方法效果都不好，Site Isolation的方法则是*move data worth stealing outside of the attacker’s address space*。(?)
3. 性能，9-13% total memory overhead
4. 兼容性