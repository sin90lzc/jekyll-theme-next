简单记录下openssl的使用示例，以备忘。


## CA证书生成

### 生成CA的私有密钥

	# openssl genrsa --out /etc/pki/ca/private/cakey.pem 2048

### 生成CA自身的证书

	# openssl req -new -x509 -key /etc/pki/ca/private/cakey.pem -out /etc/pki/ca/cacert.pem
	
参数说明：
-x509：生成x.509格式的证书
-new：生成一个新的证书
-key：新的证书所对应的私有密钥
-out：输出证书

至此为止是CA的建设。只需要在其他系统中导入这个自建的CA证书，那些由该CA签名的证书将被信任。

下面部分则为应用创建证书并签名

--------

## 为应用或用户创建证书并签名

### 生成应用的私有密钥

	# openssl genrsa -out /tmp/nginx_key.pem 2048

### 为应用生成证书签名请求

	# openssl req -new -key /tmp/nginx_key.pem -out /tmp/nginx_cert.crt

生成的nginx_cert.crt即为证书签名请求，需要把这个文件交给CA进行签名

------

## 在CA中为证书签名

	# openssl x509 -req -in /tmp/nginx_cert.crt -out /etc/pki/ca/newcert/nginx_cert.crt -CAkey /etc/pki/ca/private/cakey.pem -CA /etc/pki/ca/cacert.pem -CAcreateserial


> 参考[基于OpenSSL自建CA和颁发SSL证书](http://seanlook.com/2015/01/18/openssl-self-sign-ca/)

