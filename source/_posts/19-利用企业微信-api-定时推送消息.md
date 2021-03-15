---
title: 利用企业微信 api 定时推送消息
categories:
  - IT 业余票友
  - Python
tags:
  - Python
abbrlink: ee98fe1b
date: 2021-03-15 21:44:56
---

偶然发现了一些网络上的推送服务，包括 [Server酱](https://sct.ftqq.com/)、[sre24](https://sre24.com/)、[虾推啥](http://www.xtuis.cn/xtuisindex.html)、[WxPusher](https://wxpusher.dingliqc.com/docs/#/) 等，这些推送服务的共同之处在于它们都是以 http 协议为接入口，以微信（或其他方式）为输出，实现客制化的自定义消息推送。仔细了解后我注意到，它们中的一些使用了企业微信的 api 完成推送。那么有没有可能跳过这些第三方，直接自己来接入呢。

如果成功实现的话，以后自己写的小东西，类似于爬虫运行情况、抢购东西的结果、自动打卡的记录都可以直接推送到自己微信上了。

<!--more-->

#### 准备工作

根据企业微信官方提供的[文档](https://open.work.weixin.qq.com/api/doc/90000/90003/90487)，向用户端发送消息之前首先要获取 `access_token`， 接着依靠得到的 `access_token` 来调用发送消息的 api。由于 `access_token` 默认有 7200 s 的时效，我们需要实现将获取到的 `access_token` 缓存到本地，并在每次使用前检查是否过期。这么做的原因可以参考文档中的说明。

> 开发者需要缓存access_token，用于后续接口的调用**（注意：不能频繁调用gettoken接口，否则会受到频率拦截）**。当access_token失效或过期时，需要重新获取。
>
> 企业微信可能会出于运营需要，提前使access_token失效，开发者应实现access_token失效时重新获取的逻辑。

其实自己的代码也是只是针对过期预置了极其简单的重新获取逻辑（通过缓存获取时间来判断）；而针对于失效的逻辑嫌麻烦压根没做……只能说等真出了 bug 再补吧（懒癌发作）。获取到 `access_token` 后，就可以发送准备好的字符串了。这个脚本会被布置在我的云服务器上，将通过 linux 的 `crontab` 命令来实现 Python 脚本的定时运行。下面是一个简单的流程图。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210315172324.png" alt="逻辑流程图" style="zoom: 50%;" />



#### 注册企业微信

注册企业微信不需要任何要求，只需要准备一个微信号用来绑定成为管理员。通过[企业微信官方网站](https://work.weixin.qq.com/)注册成功后，在**应用管理**下方选择**创建应用**，之后填入相关信息，就可以得到应用的 `AgentId` 和 `Secret` 。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210315174024.png" alt="AgentId&Secret" style="zoom:50%;" />



此外我们还需要找到自己的**企业ID**，可以在**我的企业**页面的最下方找到。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210315174028.png" alt="企业id" style="zoom:50%;" />

得到这三个参数后，就可以着手开始写代码了。



#### 获取 `access_token` 

`access_token` 使用 HTTPS 协议的 get 请求获取，请求地址为 `https://qyapi.weixin.qq.com/cgi-bin/gettoken`， 需要附带两个参数。

|    参数    | 必须 |         说明          |
| :--------: | :--: | :-------------------: |
|   corpid   |  是  | 上一节得到的 `Secret` |
| corpsecret |  是  |  上一节得到的企业ID   |

这个功能很好实现，直接使用 python 的 requests 模块即可。

```python wxqiye_api.py
import requests

def get_access_token(access_token_url, corpid, secret):
    req = requests.get(access_token_url, params={
        'corpid': corpid,
        'corpsecret': secret,
    })
    access_token = req.json().get('access_token')
```

响应中会包含请求结果，我们可以提取相应的数据来进行打印或记录。

```json
{
   "errcode": 0,
   "errmsg": "ok",
   "access_token": "accesstoken000001",
   "expires_in": 7200
}
```

完整的代码会放在文末。



#### 发送消息

发送消息使用 post 方法，请求地址为 `https://qyapi.weixin.qq.com/cgi-bin/message/send`，需要将前面获取的 `access_token` 作为参数，发送的字符串作为内容。

```python wxqiye_api.py
import requests
import json

def send_msg(send_msg_url, msg):
    msg_data = {
            "touser": "@all",  # 默认情况发送给所有人即可
            "msgtype": "text",  # 类型为文本
            "agentid": 1000002,  # 企业应用的id，整型
            "text": {
                "content": msg
            },
        }
    req = requests.post(send_msg_url,
                        data=json.dumps(msg_data),
                        params={'access_token': access_token})
```

响应内容示例如下。

```json
 {
   "errcode" : 0,
   "errmsg" : "ok",
   "invaliduser" : "userid1|userid2",
   "invalidparty" : "partyid1|partyid2",
   "invalidtag": "tagid1|tagid2"
 }
```



#### 日志模块 `logging` 的使用

如果在 Python 脚本在运行中想查看某个变量，可以直接使用 `print()` 函数在控制台上显示打印信息。但如果需要看大量的 dubug 信息，或者需要将服务器上运行的脚本的调试信息保存到文件中以备后续不定期查阅，就需要使用 `logging` 模块来记录信息。下面是调用 `logging` 模块的示例。

```python
import logging

# 第一步，创建一个logger
logger = logging.getLogger()
logger.setLevel(logging.INFO)  # Log等级总开关
# 第二步，创建一个handler，用于写入日志文件
# mode="a" 表明日志使用追加形式，也就是说记录新的日志时不会覆盖旧的日志，而是将其续写到后面
# 日志的写入编码设置为 utf-8，否则会使用默认的 ACSII ，导致生成的带有中文的日志文件不能正确读取
fh = logging.FileHandler("auto_post.log", encoding="utf-8", mode="a")
# 输出到file的log等级的开关
fh.setLevel(logging.DEBUG)
# 第三步，定义handler的输出格式
formatter = logging.Formatter("%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s")
# 第四步，将logger添加到handler里面
fh.setFormatter(formatter)
logger.addHandler(fh)
```

下面是自己代码运行记录的日志示例。

```html auto_post.log
2021-03-15 01:36:37,347 - main.py[line:35] - INFO: 即将进行每日推送
2021-03-15 01:36:37,349 - wxqiye_api.py[line:48] - INFO: Token time limit <2729>.
2021-03-15 01:36:37,764 - wxqiye_api.py[line:71] - INFO: <0> Request status <200>
2021-03-15 01:36:37,764 - main.py[line:37] - INFO: 发送结束
```



#### 设置定时启动

Linux 中的 `crontab` 是用来定期执行程序的命令。它会每分钟定期检查是否有要执行的工作，如果有要执行的工作便会自动执行该工作。可以使用 `crontab -l` 来查看目前已经设置好的定时任务，用 `crontab -e` 来设定和修改定时任务。

首先将代码上传到服务器（可以使用 git），再输入 `crontab -e`，系统会自动打开一个文档。

```yml
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
5 8 * * * /usr/bin/python3 /home/ubuntu/weixin_post/main.py
```

最下面一行的 `5 8 * * * /usr/bin/python3 /home/ubuntu/weixin_post/main.py` 就是我设置的在每天的 8 点 5 分运行脚本的指令。要注意，**在 `crontab` 的配置中必须使用绝对路径**。具体的语法如下。

```
*    *    *    *    *
-    -    -    -    -
|    |    |    |    |
|    |    |    |    +----- 星期中星期几 (0 - 6) (星期天 为0)
|    |    |    +---------- 月份 (1 - 12) 
|    |    +--------------- 一个月中的第几天 (1 - 31)
|    +-------------------- 小时 (0 - 23)
+------------------------- 分钟 (0 - 59)
```

更详细的说明可以参照[这篇教程](https://www.runoob.com/linux/linux-comm-crontab.html)。

在调试过程中，我发现使用 `crontab` 来启动 Python 时，脚本默认的根目录并非脚本文件所在的目录，导致生成的缓存文件和日志文件路径混乱。要解决这个问题有两个方法。第一个方法是直接在代码中使用绝对路径，优点是稳定、一劳永逸，缺点是多平台调试会变得麻烦。第二个方法来自于这篇[文章](https://blog.csdn.net/The_Time_Runner/article/details/102664508)，可以在 crontab 的命令中添加一个打开指定文件夹的操作，即先转到目标路径再运行脚本，两个命令用 `;` 隔开。下面是示例。

```  
5 8 * * * cd /usr/bin/python3; /usr/bin/python3 /home/ubuntu/weixin_post/main.py
```

除了这两个方法以外，还可以先将用来打开文件目录并启动 Python 脚本的命令写入一个 `.sh` 文件，再使用 `crontab` 来运行这个 `.sh` 文件。这个方法我觉得比较麻烦，所以没有采用，具体可以参考这篇[文章](https://www.cnblogs.com/jacksonkwong/p/11631078.html)。



#### 运行效果

要实现消息的发送，只需生成一个 `WeixinCorporation` 类的实例并导入需要发送的字符串，实例即可自动完成 `access_token` 的获取和缓存并发送信息 。

```python
from wxqiye_api import WeixinCorporation

send_msg = WeixinCorporation('TEST\n推送测试\nPost by Python.')

# 运行结果
# Can't find a taken cache.
# New access token <JZmImdbCl...>.
# <0> Request status <200>
```

手机上收到的推送如下：

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210315191901.jpg" alt="手机推送示例" style="zoom:50%;" />



#### 项目文件

以下是获取 `access_token` 和发送消息的完整代码。

```python
import requests
import json
import os.path
import time
import logging


class WeixinCorporation(object):
    secret = "YOUR_SECRET"
    corpid = 'YOUR_CORPID'
    access_token_url = 'https://qyapi.weixin.qq.com/cgi-bin/gettoken'
    send_msg_url = 'https://qyapi.weixin.qq.com/cgi-bin/message/send'
    token_file = 'token_cache.txt'
    access_token = ''
    time_stamp = ''

    def __init__(self, msg):
        self.msg_data = {
            "touser": "@all",  # 默认情况发送给所有人即可
            "msgtype": "text",  # 类型为文本
            "agentid": 1000002,  # 企业应用的id，整型
            "text": {
                "content": msg
            },
        }
        self.check_cache()
        self.send_msg()

    def check_cache(self):
        if not os.path.isfile(self.token_file):  # 如果缓存文件不存在就创建并请求 token 并填入
            print("Can't find a taken cache.")
            logging.info("Can't find a taken cache.")
            self.get_access_token()
            self.write_to_cache(str(int(time.time())) + '\n' + self.access_token)
        else:  # 如果存在缓存文件，就读取其中的时间戳和 access_token，并作为属性赋给实例
            with open(self.token_file, 'r') as f:
                _cache = f.readlines()
                self.time_stamp = int(_cache[0].strip())
                self.access_token = _cache[1]
            used_time = int(time.time()) - self.time_stamp  # 计算此 token 已经使用的时间
            if used_time > 7000:  # 如果使用时间超过 7000 秒就重新请求新的 access_token
                print('Old taken is dying. Ready to get a new one.')
                logging.info('Old taken is dying. Ready to get a new one.')
                self.get_access_token()
                self.write_to_cache(str(int(time.time())) + '\n' + self.access_token)
            else:
                print('Token time limit <{}>.'.format(7200-used_time))
                logging.info('Token time limit <{}>.'.format(7200-used_time))

    def write_to_cache(self, content):
        with open(self.token_file, 'w') as f:
            f.write(content)

    def get_access_token(self):
        req = requests.get(self.access_token_url, params={
            'corpid': self.corpid,
            'corpsecret': self.secret,
        })
        self.access_token = req.json().get('access_token')
        print(f'New access token <{self.access_token[:9]}...>.')
        logging.info(f'New access token <{self.access_token[:9]}...>.')

    def send_msg(self):
        req = requests.post(self.send_msg_url,
                            data=json.dumps(self.msg_data),
                            params={'access_token': self.access_token})
        print(f'<{req.json().get("errcode")}> Request status <{req.status_code}>')
        logging.info(f'<{req.json().get("errcode")}> Request status <{req.status_code}>')

```




