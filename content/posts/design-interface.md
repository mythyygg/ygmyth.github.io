---
title: "如何设计一个良好的 API"
date: 2020-10-04T01:52:10+08:00
draft: false
---
本文是自己工作中的一点总结.一个平台的前后端数据传输接口一般都会在内网环境下通信，所以安全性可以得到很好的保护。本文重点讨论一下提供公网域名访问接口如何设计？我们应该考虑哪些问题？

## 安全性
安全性问题是一个接口必须要保证的规范。如果接口保证不了安全性，那么你的接口相当于直接暴露在公网环境中任人蹂躏。
安全措施主要在两个方面，一方面就是如何保证数据在传输过程中的安全性，另一个方面是数据已经到达服务器端，服务器端如何识别数据，如何不被攻击。

1. 数据加密
数据在传输过程中是很容易被抓包的，如果传输通过 http 协议，那么用户传输的数据可以被任何人获取；所以必须对数据加密，使用 https 协议，在 http 和 tcp 之间添加一层加密层(SSL层)，这一层负责数据的加密和解密。

2. 数据加签
数据加签就是由发送者产生一段无法伪造的一段数字串，来保证数据在传输过程中不被篡改，服务端通过验证这个签名来判断数据是否被篡改。
通常的做法是使用业务参数加上一个签名key拼接起来进行md5加密(当然使用其他方式进行不可逆加密也没问题).

3. 时间戳机制
有不法者直接拿到抓取的数据包进行恶意请求；这时候可以使用时间戳机制，在每次请求中加入当前的时间，服务器端会拿到当前时间和消息中的时间验证，看看是否在一个固定的时间范围内，这样恶意请求的数据包是无法更改里面时间的，所以超过这个时间范围就视为非法请求。

4. 限流机制
限流是为了更好的维护系统稳定性。使用redis进行接口调用次数统计，ip+接口地址作为key，访问次数作为value，每次请求value+1，设置过期时长来限制接口的调用频率。

5. 敏感数据脱敏
在接口调用过程中，可能会涉及到订单号等敏感数据，这类数据通常需要脱敏处理，最常用的方式就是加密。加密方式使用安全性比较高的RSA非对称加密。非对称加密算法有两个密钥，这两个密钥完全不同但又完全匹配。只有使用匹配的一对公钥和私钥，才能完成对明文的加密和解密过程。

6. 使用POST作为接口请求方式
一般调用接口最常用的两种方式就是GET和POST。两者的区别也很明显，GET请求会将参数暴露在浏览器URL中，而且对长度也有限制。为了更高的安全性，所有接口都采用POST方式请求。

## 标准化
RESTful API 通过 GET POST PUT DELETE 等方式对服务器的资源进行操作，因此在定义 API 的时候需要明确定义出：请求方式、版本、资源名称和资源ID，格式如： GET http://{host}/{version}/{resources}/{resource_id}，如查看用户编码为1 的用户信息 GET /v1/users/1。

标准化主要从 URL 设计、状态码的使用、服务器响应码使用，具体可参考阮一峰老师的RESTful API 最佳实践
![](https://www.ruanyifeng.com/blog/2018/10/restful-api-best-practices.html)

## 幂等性
幂等性是指任意多次请求的执行结果和一次请求的执行结果所产生的影响相同。说的直白一点就是查询操作无论查询多少次都不会影响数据本身，因此查询操作本身就是幂等的。但是新增操作，每执行一次数据库就会发生变化，所以它是非幂等的。

幂等问题的解决有很多思路。利用redis实现分布式锁,并防止重复提交。提供一个生成随机数的接口，随机数全局唯一。调用接口的时候带入随机数。第一次调用，业务处理成功后，将随机数作为key，操作结果作为value，存入redis，同时设置过期时长。第二次调用，查询redis，如果key存在，则证明是重复提交，直接返回错误。

## 数据规范
1. 版本控制
随着业务的发展，设计一下有兼容性的 API 是非常重要的，如果接口不能够向下兼容，业务就会受到很大影响，产品涵盖 Android iOS PC 等客户端，用户需要产品升级到最新的版本，才能更好地使用，这种情况下引入版本的概念，实现 API 的兼容性。可以参考一些开放 API 的设计，通过版本号或一些开关参数来支持一些新功能。

2. 统一响应数据格式
为了方便给客户端响应，响应数据会包含三个属性，状态码（code）,信息描述（message）,响应数据（data）。客户端根据状态码及信息描述可快速知道接口，如果状态码返回成功，再开始处理数据。

响应结果定义及常用方法：
```java
public class R implements Serializable {

    private static final long serialVersionUID = 793034041048451317L;

    private int code;
    private String message;
    private Object data = null;

    public int getCode() {
        return code;
    }
    public void setCode(int code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }
    public void setMessage(String message) {
        this.message = message;
    }

    public Object getData() {
        return data;
    }

    /**
     * 放入响应枚举
     */
    public R fillCode(CodeEnum codeEnum){
        this.setCode(codeEnum.getCode());
        this.setMessage(codeEnum.getMessage());
        return this;
    }

    /**
     * 放入响应码及信息
     */
    public R fillCode(int code, String message){
        this.setCode(code);
        this.setMessage(message);
        return this;
    }

    /**
     * 处理成功，放入自定义业务数据集合
     */
    public R fillData(Object data) {
        this.setCode(CodeEnum.SUCCESS.getCode());
        this.setMessage(CodeEnum.SUCCESS.getMessage());
        this.data = data;
        return this;
    }
}
```

# 总结
本篇文章从安全性、幂等性、数据规范等方面讨论了API设计规范。当然，一个好的API还少不了一个优秀的接口文档。虽然很多程序员不喜欢写文档，而且不喜欢别人不写文档。像swagger或其他接口管理工具是必不可少的配置，通过简单配置，就可以在开发中测试接口的连通性，上线后也可以生成离线文档用于管理API。