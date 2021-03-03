---
title: 用 Python 实现深澜校园网登录认证
categories:
  - IT 业余票友
  - Python
tags:
  - Python
abbrlink: b7af6a98
date: 2021-03-03 22:10:36
---

一直觉得学校使用的校园网登录认证很~~烂~~不实用。研究了校园网认证的 http 抓包数据后，觉得是可以通过 Python 的 Requests 库来模拟的。若真的实现了这个功能，以后命令行页面的 Linux 系统便也可以方便地使用校园网了。

<!--more-->

### 准备工作

首先来观察从打开登录页面到登录完成浏览器进行的所有 http 请求。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210303222718.png" alt="image-20210303222622724" style="zoom: 33%;" />

使用 Fiddler 仔细甄别抓包数据，可以看到在登录过程中，浏览器向校园网的认证服务器发送的有效请求共有 9 个，请求方法为 get 。

![image-20210303223533388](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210303223535.png)

其中，编号为 3 、7、8、11 的为最核心的 4 个请求，分别对应了不同的步骤。

| 请求编号 |        请求 url        |                功能                |
| :------: | :--------------------: | :--------------------------------: |
|    3#    |    /srun_portal_pc     |          获取本机 ip 地址          |
|    7#    | /cgi-bin/get_challenge |    获取 challenge (taken) 参数     |
|    8#    |  /cgi-bin/srun_portal  |              登录验证              |
|   11#    | /cgi-bin/rad_user_info | 获取登录信息（剩余流量、账户余额） |



### 请求模拟

理论上，只要我们模拟了上面的四个请求，就可以完成登录的全过程。实践证明，所有的请求都可以使用只包含 UA 信息和 cookies 的请求头，如下所示：

```python headers
headers = {
	'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.26 Safari/537.36 ',
	'Cookie': 'lang=zh-CN'
    }
```

现在我们开始逐个分析每一个请求。

#### 获取本机 ip 地址

第一个请求的请求参数是固定值，所以可以直接请求带有参数的 url 进行 get 。分配给自己的 ip 地址可以在响应内容中找到。

```python
import requests
host_login_page_url = 'http://10.186.255.33/srun_portal_pc?ac_id=1&theme=basic'
get_ip_request = requests.get(host_login_page_url, headers=default_headers)
ip_address = re.findall('id="user_ip" value="(.*?)">', get_ip_request.text, re.S)[0]
```

#### 获取 challenge (taken) 参数

第二个请求用来获取 challenge ，也就是 taken ，这是一个由服务器返回的随机字符串，用来在本地给密码等个人信息加密。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210303233430.png" alt="image-20210303233428557" style="zoom:50%;" />

可以看到请求参数共有四个：

|   参数   |                             含义                             |
| :------: | :----------------------------------------------------------: |
| callback | js 内部使用的回调参数，Python 代码中可以为任意值，但不能为空 |
| username |                             学号                             |
|    _     |                   时间戳，可以直接自行生成                   |
|    ip    |                 上一个请求获得的 ip 本机地址                 |

如果请求成功，响应信息为 json 格式的字符串，其中包含了要使用的 challenge 。使用 Requests 进行传参请求的方法如下：

```python
import requests
get_challenge_url = 'http://10.186.255.33/cgi-bin/get_challenge'
get_challenge_params = {
	"callback": 'jQuery11277455' ,
	"username": username,
	"ip": ip,
	'_': str(int(time.time() * 1000))
        }
get_challenge_request = requests.get(get_challenge_url, headers=default_headers, params=get_challenge_params)
challenge = re.search('"challenge":"(.*?)"', get_challenge_request.text).group(1)
```

#### 登录验证

第三个请求是验证的关键步骤，也是最麻烦的步骤。我们需要在本地将一系列已知的参数进行加密编码，再发送给服务器。如果服务器验证通过，返回的 json 字符串中便会包含登录成功的相关信息。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210304014714.png" alt="image-20210303234435000" style="zoom:50%;" />

这次请求的参数很多，具体如下：

|     参数     |                             含义                             |
| :----------: | :----------------------------------------------------------: |
|   username   |                             学号                             |
|     type     |                        固定值，取 '1'                        |
|   password   |                      md5 加密得到字符串                      |
|      os      |                   固定值，取 'Windows 10'                    |
|     name     |                     固定值，取 'Windows'                     |
|      n       |                       固定值，取 '200'                       |
|      ip      |                         本机 ip 地址                         |
|     info     |                   base64 编码得到的字符串                    |
| double_stack |                        固定值，取 '0'                        |
|    chksum    |                    sha1 加密得到的字符串                     |
|   callback   | js 内部使用的回调参数，Python 代码中可以为任意值，但不能为空 |
|    action    |                       验证的类型 login                       |
|    ac_id     |                     重定向标识符，取 "1"                     |
|      _       |                   时间戳，可以直接自行生成                   |

观察请求发送的数据，需要我们自己手动生成的参数为 `password` ，`chksum` ，`info`，生成的方法比较繁琐，在下一章会详细介绍。传参请求的方法和上一个请求相同，不再赘述。

#### 获取登录信息

其实进行到上一步，登录便已经成功，网络也可以正常访问了。所以这一步的实现更像锦上添花，用来获取个人的登录信息，包括账户余额，流量用量等，也就是下图显示的信息。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210303222723.png" alt="image-20210303222639926" style="zoom: 33%;" />

第四个请求很简单，只需要传入一个任意的 `callback` 和 `_` 参数，请求 `'http://10.186.255.33/cgi-bin/rad_user_info'` 即可。参数的含义和上面的几次请求相同。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210304014729.png" alt="image-20210304000440365" style="zoom:50%;" />

我们可以在返回的 json 数据中看到各种登录信息，可以使用正则表达式提取。具体的请求和提取方法不再赘述。



### 几个参数的编码和加密方法

现在距离整个流程的跑通只剩下登陆验证请求中 `password`、`chksum` 和 `info` 三个参数的生成。已经有大神对网页中的 js 代码进行了详尽的分析，提供了响应的编码和加密方法，详见[文章](https://zhuanlan.zhihu.com/p/122556315)。现在我主要从流程上阐述一下各个参数在 js 代码中的生成原理和 python 的实现方法，也是自己在研究 js 代码过程中的笔记。

#### password

```js jquery.srun.portal.js
// password 值由 hmd5 加上一个字符串生成
if (data.otp) {
                data.password = "{OTP}" + data.password;
            } else {
                data.password = "{MD5}" + hmd5;
            }
// hmd5 由 pwd 方法由用户输入的 password 加盐生成，这里的 token 即 challenge
hmd5 = pwd(data.password, token);
// pwd 方法就是一个简单的 md5 加密
function pwd(d, k) {
        return md5(d, k);
    }
```

这里的 md5 加密我们可以直接调用 python 中的 `hashlib` 库完成。我在调试过程中发现，md5 函数无论是使用密码和 challenge 加盐生成，还是由空字符串和 challenge 加盐生成，都可以通过最后的验证。这可以说是深澜认证系统的一个 bug，在这篇[文章](https://zhuanlan.zhihu.com/p/122556315)中，作者也提到了类似的问题：

> 因为第一次GET请求的response并没有`password`参数，所以程序中`$data.password`只受token的影响，并不包含密码信息；而密码是通过`$data.info`传递的，这还是真的一个比较奇怪的点。

#### info

```js jquery.srun.portal.js
// info 即参数 i
info: i
// i 由方法 info 生成
i = info({
    username: username,
    password: data.password,
    ip: (data.ip || response.client_ip),
    acid: data.ac_id,
    enc_ver: enc
	}, token),
// info 方法是用字符串加上一系列窒息操作生成，参数 d 为 challenge，k 为一个字典
// 首先将 json 转成了字符串，接着进行了 xEncode 加密，最后用 base64 编码
function info(d, k) {
        return "{SRBX1}" + $.base64.encode(xEncode(json(d), k));
    }
// 参数 k
{
    username: username,  // 学号 "1800000214"
    password: data.password,  // 密码 "passward"
    ip: (data.ip || response.client_ip),  // 本机ip "10.184.00.291"
    acid: data.ac_id,  // 重定向标识符，取 "1"
    enc_ver: enc  // enc 是固定一个字符串，取 "srun_bx1"
}
// json 方法
function json(d) {
        return JSON.stringify(d);
    }
// xEncode 方法（部分）
function xEncode(str, key) {
        if (str == "") {
            return "";
        }
		[......]
```

这里使用的 base64 编码和正常的编码方法有所不同，其使用了自定义的字母表来进行编码，自定义字母表如下方代码所示。关于字母表的含义可以参考[这篇文章](https://blog.csdn.net/xxj13706568076/article/details/105969173)，

```python
_ALPHA = "LVoJPiCN2R8G90yg+hmFHuacZ1OWMnrsSTXkYpUq/3dlbfKwv6xztjI7DeBE45QA"
```

另外，xEncode 也是一个罕见的加密方法，但幸运地是现在已经有现成的改写好的 python 轮子，我们直接拿来使用即可。

#### chksum

```js jquery.srun.portal.js
// chksum 由 chksum() 方法生成
chksum: chksum(chkstr),
// chksum() 方法即进行一次 sha1 加密
function chksum(d) {
    return sha1(d);
}
// chkstr 由上面的一系列参数累加得来
var chkstr = token + username;
            chkstr += token + hmd5;
            chkstr += token + data.ac_id;
            chkstr += token + (data.ip || response.client_ip);
            chkstr += token + n;
            chkstr += token + type;
            chkstr += token + i;
```
参数 `chksum` 的生成方法和参数 `password` 类似，直接调用 `hashlib` 库对累加的数据进行 sha1 加密即可。这个参数应该是对所有的信息进行一个总体验证。



### 编写 Python 代码

跑通了整个流程，就可以启动 IDE 开写了。其他几位大神的代码基本都是面向过程，我个人觉得使用起来不够清爽。所以我把编码加密流程进行了重构，封装成了一个类 `ShenlanEncode` 。在生成实例时传入学号和密码，即可通过调用实例的属性来获取 `password`、`chksum` 和 `info` 三个参数，下面是一个例子:

```python
from xauat_login import ShenlanEncode
item = ShenlanEncode(username='1101101110', password='password')
password = item.md5
info = item.info
chksum = item.chksum
```

需要进行的各个请求的过程也被封装成了一个类 `XauatLogin`。它的实例会在生成时自动获取 ip 地址，之后调用实例的方法即可完成登录等操作。

```python
from xauat_login import XauatLogin
login = XauatLogin(username='1101101110', password='password')
a.log_in()  # 登录
a.log_out()  # 注销
a.get_login_info()  # 获取登录信息
```

最后套上一个简单的循环，运行效果如下图。虽然很简陋，各类异常也没有被很好地处理，但核心的功能还是磕磕绊绊地实现了。等完成了毕业答辩再来进一步完善吧 ^_^ 。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210304010754.png" alt="image-20210304010738122" style="zoom:50%;" />



### 项目文件

[vincentimba/shenlan_xauat](https://github.com/vincentimba/shenlan_xauat)



### 参考

1. [深澜校园网登录的分析与python实现-北京理工大学版](https://zhuanlan.zhihu.com/p/122556315)
2. [深澜认证协议分析,python模拟登录](https://blog.csdn.net/qq_41797946/article/details/89417722)
3. [python自定义解决base64编码](https://blog.csdn.net/xxj13706568076/article/details/105969173)



### 后记：关于 base64 编码的呓语

现在的时间是晚上 2 点 48 分，伴随着耳机里播放的告五人的《带我去找夜生活》，我困惑了许久的问题终于告一段落。

在写代码的过程中，不得不说找到的参考文章提供了非常多的~~轮子~~帮助。但阅读这些文章和查阅源码后，我发现了一个问题，那就是深澜登录验证中使用了一个自定的 base64 编码方法，准确地说是使用了自定义的字母表来进行编码。但对于这里具体的原理，文章作者只是粗略地带过并给出了代码和方法。于是我找到了自己的一组登录数据，按照代码尝试进行了加密，最终得到的 base64 加密的字符串，与之前抓包得到的一致。从技术上说，整个流程已经可以跑通了。

但患有被害妄想症的我不禁想，如果深澜在后续的编码中更换了字母表该怎么办呢？虽然这种概率是很小的（因为我模拟了正常的登录流程，且这本来就不是一个加密的环节）。经过学习后得知， base64 编码是一种可逆的、防君子不防小人的编码手段，是一种双向的编码方式。也就是说，**即使服务器端更换了其中代表映射关系的字母表，也可以在足够多的输入和输出样本下将对应的字母表遍历出来**。

虽然知道了结论，好奇的我还是决定验证一下这个过程。我调用了 python 中自带的 base64 库，并对之前的自己得到的字符串进行了编码。问题出现了，**通过 python 库得到的编码结果与大佬提供方法得到的编码结果不一样**，甚至还要多出一些字符。

我百思不得其解，开始了疯狂搜索，终于在[python自定义解决base64编码](https://blog.csdn.net/xxj13706568076/article/details/105969173)这篇文章中找到了答案（非常感谢这位作者，不然我晚上要睡不着了 ^_^）。这位作者除提供了一种更加简洁的 python 实现的自定义字母表编码方案外，还附带了一些带着调侃语气的阐述：

> 好了，到现在为止，英文我们实现了基本的编码解码，虽然代码粗糙，总归结果令人满意。现在我们来讨论一下中文，接着前文话题，中文的ASCII码怎么解决？python总是考虑的很多。ord方法 用在英文上 返回的是ascii码值，而用在中文上返回的就是 unicode值。没错，完美解决中文编码的时候编码。但是问题还是来了，当你解码的时候，chr（）方法只能返回ascii码，因此在解码中文编的码时就无法正常解码了。网查，讨论，均未解决。留着以后有大佬指点。

问题终于找到了。原来不是 python 内置的 base64 库和上面找到的 base64 编码方式不兼容的问题。现在再仔细看一看我用来调试的代码。

```python
import base64
str = '目标编码字符串'
# 下面将 unicode 编码的字符串编码成二进制数据
# 补充一句，UTF-8 是在互联网上使用最广的一种 Unicode 的实现方式
raw_str = str.encode("utf-8")
# 接下来将二进制数据进行 base64 编码
base64_result = base64.b64encode(raw_str)
# 输出 base64 编码结果
print(base64_result)
```

深澜的 js 语句中，相关的信息被 xEncode 方法编码成了新的字符串。在上述作者提供的代码中，会先将此字符串编码成二进制数据，再按部就班地进行 base64 编码，且这个方法只能识别 ASCII 字符，这一点可以从其他作者提供的代码中看出。

```python
def _getbyte(s, i):
    x = ord(s[i])
    if (x > 255):  # 这里要求字符的编码必须在 ASCII 的范围中
        print("INVALID_CHARACTER_ERR: DOM Exception 5")
        exit(0)
    return x
```

但对于 python 内置的函数，在对字符串进行 base64 编码之前，需要先将其转化为二进制数据，再导入 `base64.b64encode()` 方法。要将字符串转化为二进制数据，就要先使用 `str.encode("utf-8")` 方法，填入相应的编码格式。由于 python 内部统一使用 Unicode 格式，这里只能强行使用了 `utf-8` 来解码 xEncode 方法生成的字符串，而这个字符串的编码格式目前是未知的，所以解码出的二进制数据和自己解码出的二进制数据并不相同，这就是导致输出结果不一致的原因。

当然，上面只是我自己的推测，不一定就是真正的原因。但如果再继续深一步探究，无疑已经超出了我这个业余票友的能力和精力范围，只好暂时止步于此，以后有机会再仔细补一补编码知识。现在只需知道，如果字母表真的发生了改变，我们只需要通过多次登录抓取不同的输入输出结果，并通过上面自制的识别 ASCII 字符的编码方法遍历出新的字母表映射关系即可。