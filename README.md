# 西南交大教务网成绩通知系统
这是一整套完整的西南交大教务网自动查询成绩和自动通过邮件来通知新成绩的系统，可以和小伙伴们一起用。
**本项目使用了大量Promise风格的代码，如果不熟悉Promise风格，可以先了解一下Promise哦。**
## 功能
在网页端录入你的教务账号、密码，自动查询成绩，在成绩发生更新时，1分钟之内通过邮件将分数和科目告知你。

如果你想使用我的系统，请访问 http://vin94.cn/grades
### 如何运行
```javascript
cnpm install
node app.js
```
需要在module/目录下创建一个`sensitiveConfig.js`文件, 用于保存敏感的配置信息, 内容如下
```javascript
/**
 * 敏感信息
 */
const nodemailer = require('nodemailer');

// mysql配置
exports.mysql_config = {
    host: '',
    user: '',
    password: '',
    database: ''
};

//email配置
exports.transporter = nodemailer.createTransport({
    // host: 'smtp.ethereal.email',
    service: 'qq', // 使用了内置传输发送邮件 查看支持列表：https://nodemailer.com/smtp/well-known/
    port: 465, // SMTP 端口
    secureConnection: true, // 使用了 SSL
    auth: {
        user: '@xx.com', //你的邮箱
        // 这里密码不是qq密码，是你设置的smtp授权码
        pass: '',
    }
});

//在用户多了以后，单个邮箱不够用了，可能会出现发信限制，于是改用两个邮箱账号（优先使用ne126那个）
exports.transporter = {
    txEnter: nodemailer.createTransport({
        host: 'smtp.exmail.qq.com',
        // service: 'mxhichina', // 使用了内置传输发送邮件 查看支持列表：https://nodemailer.com/smtp/well-known/
        port: 465, // SMTP 端口
        secureConnection: true, // 使用了 SSL
        auth: {
            user: '', //你的邮箱
            // 这里密码不是qq密码，是你设置的smtp授权码
            pass: ''
        }
    }),
    txEnterEmail: 'example@vin94.cn',
    ne126: nodemailer.createTransport({
        // host: 'smtp.ethereal.email',
        service: '126', // 使用了内置传输发送邮件 查看支持列表：https://nodemailer.com/smtp/well-known/
        port: 465, // SMTP 端口
        secureConnection: true, // 使用了 SSL
        auth: {
            user: 'example@126.com', //你的邮箱
            // 这里密码不是qq密码，是你设置的smtp授权码
            pass: ''
        }
    }),
    ne126Email: 'example@126.com'
}

// 百度API的token，每隔一段时间（几天）就要重新获取一次
exports.baiduApi = {
    'access_token': ''
};

// 在使用百度识别文字的API时用到这个，改成你的公网IP或者域名即可。
// 如果你的不是公网IP，就不能采用URL上传图片的方式，得换成base64编码来发送图片，详情参考
// https://ai.baidu.com/docs#/OCR-API-GeneralBasic/top
exports.domain = 'vin94.cn';
```
## 文件

```
├─ module // 自定义的模块
┃├─ email.js // 发送邮件
┃├─ getImg.js // 从教务网获取验证码图片及SESSIONID
┃├─ getLatestGrade.js // 查询成绩
┃├─ handleConnection.js // 处理mysql连接
┃├─ imgToString.js // 调用百度文字识别API，识别验证码
┃├─ keepChecking.js // 控制循环查询新成绩
┃├─ logger.js // 写日志
┃├─ loginToDean.js // 登录到教务网
┃├─ sensitiveConfig.js // 配置文件，包含敏感信息
┃├─ user.js // 用户模块
┃└─ util.js // 一些小工具
├─ public
┃├─ css
┃├─ data
┃┃├─ randImg // 验证码图片的目录
┃├─ js
├─ router
┃└─ router.js // 路由控制
├─ views
┃├─ grade-setting.html
┃├─ index.html
┃└─ quest.html
├─ .gitignore
├─ app.js // 程序入口
├─ package.json
└─ README.md
```

## 为什么要做这个
每到期末季，我校学子总是少不了不断地登录教务网，一遍又一遍输入枯燥无味的验证码，不断地刷新成绩查询页面，期盼着老师快点出分。
奈何我校没有别人学校的公众号成绩通知，只能自己想一个解决方案来实现最新成绩的通知了。
## 实现方法
### 模拟登录
首先要模拟登录，获取Cookie。
在此之前，我的亲朋好友们会在网页端录入他们的教务账号、密码以及我提供的兑换码到数据库中（服务器资源有限，若用户过多会导致请求过于频繁，会封锁IP，但是这个功能我还没做，因为不是那么重要）。
稍微看了下教务网登录页的代码，再用Chrome的开发者工具查看网络请求，再借助Cookie插件，研究了一下发现是在请求验证码的时候，响应头带上了‘Set-Cookie’字段，因此第一步应该是发送获取验证码的请求以获取一个SESSIONID，再想办法把这个SESSIONID认证。
可以通过百度提供的API识别验证码（幸好教务网的验证码图案简单，不然就难办了），再加上账号和密码发送`POST`请求，获取`SESSIONID`。

[百度AI开发者平台](https://ai.baidu.com/tech/imagerecognition)

[通用文字识别](https://ai.baidu.com/docs#/OCR-API-GeneralBasic/top)

教务网的SESSION有效期很短，但是只要不断地发新请求，它就能保持有效。

### 获取成绩查询页
经过研究后发现，只要访问
`http://jwc.swjtu.edu.cn/vatuu/StudentScoreInfoAction?setAction=studentScoreQuery&viewType=studentScore&orderType=submitDate&orderValue=desc`
就可以直接进入成绩查询页，而不需要经过点击再发起新的请求，因此只要在Cookie中带上SESSIONID发一个GET请求就可以获得包含你成绩信息的页面。
```javascript
axios.get(url, {
        headers: {//设置请求头
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3',
            'Accept-Encoding': 'gzip, deflate',
            'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
            'Cache-Control': 'max-age=0',
            'Connection': 'keep-alive',
            'Cookie': `JSESSIONID=${Sid}; username=${username}`,//SESSIONID和学号
            'Host': 'jwc.swjtu.edu.cn',
            'Upgrade-Insecure-Requests': '1',
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36'
        }
    }).then((result) => {
        let data = result.data;
        console.log(data);
        getDataFromHTML(data);
    }).catch((err) => {
        console.log(err);
    });
```
### 解析页面，抓取最新成绩
现在思路就非常清晰了，接下来只要在获取到的页面中，将其中的成绩信息提取出来即可。
下面用到一个叫`cheerio`的模块，用法参照[官方文档](https://www.npmjs.com/package/cheerio)
```javascript
    function getDataFromHTML(data) {
        const $ = cheerio.load(data);
        $('table#table3 tbody tr:nth-child(2) td:nth-child(3)').each((i, elem) => {
            let className = elem.children[0].data;//课程名
            console.log(`yes`);
            console.log(className);
        });
    }
```
