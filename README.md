# secqa 安全知识问答

本文档主要总结日常工作中，关于安全漏洞或知识，开发人员的常见问题

## #架构评审常见问题

Q:系统初始阶段，要注意哪些问题？



Q:我想要学习安全测试，如何入门？



## #安全漏洞常见问题

### 账号体系相关问题

**1、为什么需要验证码？**

​		以登录接口为例，验证码的作用了为了防止攻击者通过自动化工具或脚本进行大量重复请求，实现账号密码组合的不断尝试，从而获取正确账号和密码的可能。

**2、我已经接入了验证码？为什么还是不行？**

​		验证码的正确使用姿势：仍以登录接口为例，验证码和账号密码等参数一起传递给服务端。服务端首选校验验证码：如果验证码正确，立即使当前验证码失效，然后进行后续接口操作（验证账号密码）；如果不正确，不执行任何操作；如果验证参数为空，也不应执行后续操作。

常见错误姿势：

- 验证码在单独的接口中校验，实际完成动作的接口于验证码不在同一接口
- 验证码未能做到一旦验证理解失效，可多次重复使用。
- 当接口未附带验证码参数，服务端逻辑也能进入到后续正常逻辑。

![验证码](img/验证码.gif)



**3、密码强度要求**

1. 密码长度最小值：10位。

2. 密码复杂度要求：3种(大写字母、小写字母、数字、特殊字符中任选3种)。

3. 密码最长使用期限：90天。 

4. 注意必须在服务端接口进行校验，js校验无效

5. 应当在注册、修改密码、重置密码等所有密码相关接口使用该策略

   

**新建用户密码生成方案**

1、生成随机密码，通过短信或邮箱方式通知用户
2、如果使用相同初始密码，必须强制要求第一登录时修改密码，不修改密码不允许访问系统。且密码强度必须符合要求

**权限相关漏洞问题**

越权测试方法：

1. 通过管理员账号登录并发起操作请求，通过代理拦截请求数据包
2. 替换其中的cookie为普通用户、低权限账号甚至无任何权限账号的cookie（测试前根据需要自行创建）
3. 取消拦截继续请求，结果发现操作成功。证明存在越权漏洞。

越权修复思路：

每个重要接口进行具体的操作前（比如数据的增、删、改、查）都应该校验当前用户的权限（或者所在角色的权限），如果有权限则继续完成操作，否则拒绝。

Novatar框架的越权修复方案：

1. 以pub 结尾的接口是任何人都可以访问的，未登录用户也可以访问。除了登录注销等接口，一般不需要用到。
2. 以pvt 结尾的接口是只要登录了就可访问，不会校验用户权限。比如获取公司组织架构信息的接口。
3. 以arz结尾的接口，在完成具体的操作前，会检查用户的权限，会根据用户管理和角色管理中的设置来运行或拒绝访问。大部分涉及到数据的增删查改操作都应该使用arz结尾的接口。

Sfopen框架的越权修复方案（包含两个步骤）：

1. spring-mvc.xml配置文件配置全局的权限检查拦截器。
2. 在模块管理部分配置需要区分权限的接口URL。

系统接口存在CORS跨域访问漏洞
应当校验origin或者referer，只允许来自本域的请求
建议全局配置filter实现



XSS解决方案：

1. 对用户输入的数据进行严格过滤，包括但不限于以下字符及字符串 Javascript script src img onerror { } ( ) < > = , . ; :  " ' # ! & / * \
2. 根据页面的输出背景环境，对输出进行编码
3. 使用一个统一的规则和库做输出编码
4. 对于富文本框，使用白名单控制输入，而不是黑名单
5. 在Cookie 上设置HTTPOnly 标志，从而禁止客户端脚本访问Cookie


系统存在fastjson反序列化漏洞，可构造命令执行.
修复方案：升级fastjson >= 1.2.63 ;或者使用gson替换fastjson（fastjson常出漏洞，强烈建议该方法）


Excel XXE修复方案
如果系统使用了低版本POI库，导致处理Excel文件时存在XXE和SSRF漏洞，升级该库，必须大于3.14版本
如果系统使用了Excel Streaming Reader <= 2.0.0 XXE 中，升级该库，必须大于2.0.0版本

XXE修复方案
https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html

系统存在JWT弱密钥等问题

如果你使用了SFOPEN框架，框架升级指南：



**未授权访问**

什么是未授权访问？

攻击者不需要登录系统，不需要提供有效的cookie等标识用户身份的信息就可成功访问系统，或者其中的接口（成功是只能够完成增删查改等操作）

未授权访问修复：

每个接口都需要判断当前请求提供的身份信息是否有效，可以通过全局filter实现。



**数据库账号密码加密**

https://stackoverflow.com/questions/10306673/securing-a-password-in-a-properties-file
https://blog.csdn.net/u011676300/article/details/80740530

**密码加密传输方案**：使用https或者自行加密传输，二者选其一即可

RSA加密传输参考https://www.cnblogs.com/hjtdlx/p/3944746.html

**密码加密存储方案**：至少采用【带salt】的SHA512。

注意不要弄混【加密传输】和【加密存储】，加密传输需要使用可解密的算法，服务器必须解密出原始密码后，进行密码强度判断。最后在数据库中存储的，必须是使用带salt的HASH算法计算后（不可逆）的结果。不能直接存储加密传输的密文！！！



**客户信息加密：**
系统涉及客户敏感信息，需要做如下处理：

1. 数据库中必须加密存储客户敏感信息（身份证，手机号，地址等），一般建议先做一次base64，然后AES256。
2. 系统中展示客户信息时应适当脱敏，接口返回的数据就需要是脱敏的。
3. 对客户信息的导出行为进行规范的日志记录，做到数据泄露可追溯！
4. 另外，顺丰机房、丰云上（走DCN区）的系统是可以调用硬件加密机的，不用自己实现。



**XSS解决方案：**

1. 对用户输入的数据进行严格过滤，包括但不限于以下字符及字符串 Javascript script src img onerror { } ( ) < > = , . ; :  " ' # ! & / * \

2. 根据页面的输出背景环境，对输出进行编码
3. 使用一个统一的规则和库做输出编码
4. 对于富文本框，使用白名单控制输入，而不是黑名单
5. 在Cookie 上设置HTTPOnly 标志，从而禁止客户端脚本访问Cookie



系统存在fastjson反序列化漏洞，可构造命令执行.
修复方案：升级到最新版本或使用GSON替换（建议使用该方案）



**Excel XXE修复方案**
如果系统使用了低版本POI库，导致处理Excel文件时存在XXE和SSRF漏洞，升级该库，必须大于3.14版本
如果系统使用了Excel Streaming Reader <= 2.0.0 XXE 中，升级该库，必须大于2.0.0版本



为什么每次安全测试都有新的问题？

1、自动化只能发现简单的问题，并不能代替人工测试。
2、每个安全测试人员的思路和方法不一样，能发现的问题也可能不同。
3、系统在更新、代码在变、配置在变化、数据在变化，这些都可能导致出现新的问题。
4、我们的测试技术也在进步。以前没测试到的功能，现在测试到了。以前发现不了的问题，现在能发现了。
5、没有人能保证一次测试发现所有问题，就像你们不能保证没有bug是一个道理。
说句不好听的话，有没有问题，根本上还是取决于你们的代码，遇到某种类型的问题，希望可以多学习了解一下，后续自然就会越来越好。



