# HTTP  
---


   
## 请求方法  


### 1. GET  
主要用于获取资源。  

### 2. POST
用来传输实体的主体，虽然用GET方法也可以传输实体的主体，但是一般不用GET方法进行传输，而是用POST方法。虽然说POST功能和GET很相似，但是POST的主要目的不是是获取主体内容。   

### 3. PUT  
PUT方法用来传输文件。   

### 4. HEAD
和GET方法一样，但是不会返回主体，可以用于确认主体的更新时间等。   

### 5. DELETE   
DELETE方法用来删除文件，和PUT方法相反。   

### 6. OPTION  
OPTION用来确认服务器支持的方法。   

### 7. TRACE  
TRACE用来追踪经过了多少服务器。   

### 8. CONNECT  
CONNECT方法要求与代理服务器通信时建立隧道。  


## 常见状态码


### 1. 2XX  
请求被服务器正常处理。   

### 2. 3XX
重定向错误，需要某些特殊处理以正确处理请求。   

### 3. 4XX  
客户端请求错误。  

### 4. 5XX  
服务器错误。  


## 通用首部字段  


### 1. Cache-Control  
- public指令：表明其他用户也可以使用该缓存。  
- private: 特定用户可以使用该缓存。   
- no-cache: 如果是客户端表示客户端不接受缓存，如果是服务端表示缓存服务器可以缓存，但是每次使用缓存必须向服务器确认。  
- no-store: 表示请求或者对应的响应中包含机密信息，不允许缓存。（注意和no-cache的区别）。  
- max-age：如果是客户端表示如果缓存没有超过该时间范围，直接把缓存返回，如果是服务器表示如果没有超过该时间就不必向服务器确认，如果超过了该时间就必须向服务器确认。（单位：秒）  

### 2. Connection  
- 控制不再转发给代理的首部字段，例如：Connection：Upgrade。  
- 管理持久连接。例如：Connection：close。  

### 3. Date  
表示http报文创建的时间，例如：Date：Tue，03 Jul 2012 04:40:59 GMT。  


## 请求首部字段  


### Accept  
表示客户端可以处理的媒体类型及媒体类型的优先级。

### Accept-Charset  
表示客户端支持的字符集以及优先级。  

### Accept-Encoding  
表示客户端支持的内容编码以及优先级。  

### Authorization  
表示用户的认证信息，如果认证失败，一般会返回401错误。  

### If-XXXXX  
只有当满足某种条件时，客户端才会执行某种请求。例如：If-Match，If-Modified-Since，If-None-Match，If-Range，If-Unmodified-Since。  

### Range  
只是获取部分资源的范围。  

### Referer  
表示该请求是从哪个页面发起的。  

### User-Agent  
描述客户端信息。  


## 响应首部字段  


### Age  
告知客户端该响应创建了多久了。（单位：秒）  

### ETag  
实体的标实。  

### Location 
一般通过3XX错误配合使用，将请求重定向某个资源。  

### Server  
告知客户端服务器上的软件应用名称。  

### WWW-Authenticate  
告知客户端该资源需要认证。 


## 实体首部字段  


### Allow  
该实体支持的请求方法。  

### Content-Encoding  
实体的编码方式。  

### Content-Length  
资源大小。（单位：字节） 

### Content-Range  
实体范围。（只是一部分实体） 

### Content-Type  
实体的类型，以及编码等。  

### Expires  
实体失效时间。  

### Last-Modified  
资源上次修改的时间。  


## Cookie相关头部  


### Set-Cookie  
当服务器开始管理客户端状态时，会事先告知各种信息。  

- expires:设置cookie的有效期，如果没有设置该属性，则有效期就是本次会话（session）这段时间内。  
- domain：指定可以使用该cookie的域名。  
- secure：只有建立https安全连接时才可以发送cookie。  
- HttpOnly:限制JavaScript无法获取到cookie内容。  

### Cookie  
将从服务端收到的cookie再发送给服务端，自己也可以在添加cookie。  