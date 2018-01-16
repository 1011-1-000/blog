---
layout: post
title: "谈谈我对session, cookies和jwt的理解"
date: 2018-01-16 22:20:15 +0800
comments: true
categories: architecture
tags: [session,cookies,jwt]
---
最近在做项目重构，因为核心功能仅以restful风格接口提供，因此对于会话管理这一部分，目前考虑使用jwt(Json Web Token)。本文是我在项目开发过程中对这几种会话管理技术理解的一些总结。不对之处，请指正。

#### 为什么我们需要会话管理

众所周知，HTTP协议是一个无状态的协议，也就是说每个请求都是一个独立的请求，请求与请求之间并无关系。但在实际的应用场景，这种方式并不能满足我们的需求。举个大家都喜欢用的例子，把商品加入购物车，单独考虑这个请求，服务端并不知道这个商品是谁的，应该加入谁的购物车？因此这个请求的上下文环境实际上应该包含用户的相关信息，在每次用户发出请求时把这一小部分额外信息，也做为请求的一部分，这样服务端就可以根据上下文中的信息，针对具体的用户进行操作。所以这几种技术的出现都是对HTTP协议的一个补充。使得我们可以用HTTP协议+状态管理构建一个的面向用户的WEB应用。
<!--more-->

#### Session与Cookies的区别

这里我想先谈谈session与cookies，因为这两个技术是做为开发最为常见的。那么session与cookies的区别是什么？个人认为session与cookies最核心区别在于**额外信息由谁来维护**。利用cookies来实现会话管理时，用户的相关信息或者其他我们想要保持在每个请求中的信息，都是放在cookies中，而cookies是由客户端来保存，每当客户端发出新请求时，就会稍带上cookies，服务端会根据其中的信息进行操作。当利用session来进行会话管理时，客户端实际上只存了一个由服务端发送的session_id，而由这个session_id，可以在服务端还原出所需要的所有状态信息，从这里可以看出这部分信息是由服务端来维护的。

除此以外，session与cookies都有一些自己的缺点：

- cookies的安全性不好，攻击者可以通过获取本地cookies进行欺骗或者利用cookies进行CSRF攻击。
- 使用cookies时，在多个域名下，会存在跨域问题。
- session在一定的时间里，需要存放在服务端，因此当拥有大量用户时，也会大幅度降低服务端的性能。
- 当有多台机器时，如何共享session也会是一个问题，也就是说，用户第一个访问的时候是服务器A，而第二个请求被转发给了服务器B，那服务器B如何得知其状态。

实际上，session与cookies是有联系的，通常来说，session_id就是存放在cookies中的。

#### JWT认证

##### 什么是JWT

JWT是Json Web Token的全称，它是由三部分组成：

- header
- payload
- signature

header中通常来说由token的生成算法和类型组成。如：

```python
{
    "alg":"HS256",
    "typ":"JWT"
}
```

payload中则用来保存相关的状态信息。如用户id，role，name等。

```python
{
    "id": 10111000,
    "role": "admin",
    "name": "Leo"
}
```

signature部分由header，payload，secret_key三部分生成，其生成公式为：

```python
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret_key)
```

再将这三个部分组合成**header.payload.signature**的形式。

##### JWT如何工作

首先用户发出登录请求，服务端根据用户的登录请求进行匹配，如果匹配成功，将相关的信息放入payload中，利用上述算法，加上服务端的密钥生成token，这里需要注意的是secret_key很重要，如果这个泄露的话，客户端就可以随意篡改发送的额外信息，它是信息完整性的保证。生成token后服务端将其返回给客户端，客户端可以在下次请求时，将token一起交给服务端，一般来说我们可以将其放在Authorization首部中，这样也就可以避免跨域问题。接下来，服务端根据token进行信息解析，再根据用户信息作出相应的操作。

##### JWT优点与缺点及对应的解决方案

考虑JWT的实现，上面所述的关于session，cookies的缺点都不复存在了，不易被攻击者利用，安全性提高。利用Authorization首部传输token，无跨域问题。额外信息存储在客户端，服务端占用资源不多，也不存在session共享问题。感觉JWT优势很明显，但其仍然有一些缺点：

- 登录状态信息续签问题。比如设置token的有效期为一个小时，那么一个小时后，如果用户仍然在这个web应用上，这个时候当然不能指望用户再登录一次。目前可用的解决办法是在每次用户发出请求都返回一个新的token，前端再用这个新的token来替代旧的，这样每一次请求都会刷新token的有效期。但是这样，需要频繁的生成token。另外一种方案是判断还有多久这个token会过期，在token快要过期时，返回一个新的token。下面是我在项目里的一个实现。

  ```python
  @staticmethod
  def verify_auth_token(token):
      s = Serializer(current_app.config['SECRET_KEY'])
      try:
          data = s.loads(token)
          refresh_token_or_not = True if now() + 600 >= data.get('expire_time') else False
      except:
          return None
      return User.query.get(data['id']), refresh_token_or_not

  def auth(func):
  	"""
  	According to the token in the cookies(to compatible with the previous api, will abort) or authorization header to determine the login status. if user has logined,
  	the user object will be in global varaibles, so we can access it easily and return normally. however, the cases below
  	will not be allowed to finish the request:
  		1. no token or authorization content
  		2. token is in black list
  		3. token is expired
  	In two cases, the token will be added to the black list.
  		1. logout by the user
  		2. the outdate token, which means the token will be expired in ten minutes
  	"""
  	
  	def wrapper(*args, **kwargs):
  		
  		if not getattr(func, 'auth', True):
  			return func(*args, **kwargs)

  		token = request.headers.get('Authorization') or request.cookies.get('session_id')
  		# to process the authorization correctly
  		if not token:
  			return unauthorized('Please login first!')

  		token = token.split(' ')[-1]
  		# logout for user
  		if token in redis_db.get('token_black_list'):
  			return unauthorized("Invalid token")

  		user, refresh = User.verify_auth_token(token)
  		if user:
  			g.user = user
  			try:
  				res = func(*args, **kwargs)
  			except Exception as e:
  				current_app.logger.exception(e)
  				res = internal_error
  			if refresh:
  				res.setdefault('token', user.generate_auth_token())
  				redis_db.get('token_black_list').append(token)
  			return res
  		else:
  			return unauthorized("Invalid token or token has expired")

  	return wrapper
  ```

- 用户主动注销。JWT并不支持用户主动退出登录，当然，可以在客户端删除这个token，但在别处使用的token仍然可以正常访问。为了支持注销，我的解决方案是在注销时将该token加入黑名单。当用户发出请求后，如果该token在黑名单中，则阻止用户的后续操作，返回Invalid token错误。

#### 总结

无论session还是cookies或是jwt。目前情况是jwt仍然无法代替session，cookies也会有人用。它们各自有自己的优势和缺点，不能因为有一些缺点就否认技术的存在，缺点仍然可以采用一些技术手段来弥补，比如通过添加csrf token来阻止来自CSRF的攻击，比如利用redis集群来做session的存储和共享。技术只是工具，选择最适合你的才是最重要的。



