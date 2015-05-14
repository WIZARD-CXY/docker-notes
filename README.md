####创建证书的基本流程

创建根证书 rootca.key

`$ openssl genrsa -out rootca.key 2048`

根据rootca.key生成认证文件rootca.crt（全程回车即可，不用输入额外内容）

`$ openssl req -x509 -new -nodes -key rootca.key -days 10000 -out rootca.crt`

生成registry要使用的 devregistry.key

`$ openssl genrsa -out devregistry.key 2048`

根据 devregistry.key生成证书请求 devregistry.csr 文件 注意填信息的时候 Common Name里面要填服务器的名称(比如填写devregistry)

`$ openssl req -new -key devregistry.key -out devregistry.csr`

根据证书请求文件以及根证书文件生成最终registry server端使用的crt文件

`$ openssl x509 -req -in devregistry.csr -CA rootca.crt -CAkey rootca.key -CAcreateserial -out devregistry.crt -days 10000`

实验用的测试的证书已经生成好，放在了hostnameCA文件夹中，生成证书的时候，Common Name 一项中填的是 devregistry 具体使用的时候，需要在客户端的/etc/hosts中把registry IP 和devregistry的映射关系加进来。

具体可以参考这个[文章](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04)


####registry配置

具体的nginx 以及 docker-registry 还有 docker-compose的配置文件可以参考这个[文章](http://dockone.io/article/338)

首先看一下nginx目录下的docker-registry.htpasswd文件，这个里面是用户认证信息，admin:加密密码，这里面一共两个可以用于认证的用户，一个是admin:admin 一个是 testuser:teseuser，具体添加新的用户的时候可以参考[这个文章](http://segmentfault.com/a/1190000000801162)

添加一个admin用户的具体操作就是：

`$ cd docker-notes && htpasswd -c nginx/docker-registry.htpasswd admin`

也许需要使用　`sudo apt-get install apache2-utils` 安装 `htpasswd`

之后配置nginx的配置文件，在配置文件nginx.conf中需要修改的地方有：
server中的server_name 改成实际使用的server hostname，比如这里就写成devregistry。
在docker-compose.yml文件中 volumn的地方,使用新生成的devregistry.key和devregistry.crt文件，文件路径根据实际情况修改。

特别注意的地方：
在写配置文件的时候，还是不要直接采用ip的形式，最好是修改 registry server 以及 client server的/etc/hosts文件 在里面加上对应的映射关系 具体可以参考这个[issue](https://github.com/docker/docker/issues/8943) 可能导致curl的时候用--cacert参数加证书可以但是docker login的时候不可以（貌似会默认按照hostname的方式来）

之后相关信息配置好之后，直接sudo docker-compose up这样就能启动对应的容器了。

####client端的配置
启动registry server容器之后，需要在客户端进行配置，首先把devregistry加入到/etc/hosts中，重启网络，由于证书是自己签发的，在docker login的时候可能会有x509: certificate signed by unknown authority 的错误，具体解决方案可以参考[这个issue](https://github.com/docker/docker/issues/8849) 验证一下，这个方式是有效的：
 
	Copy CA cert to /usr/local/share/ca-certificates.
 	sudo update-ca-certificates
 	sudo service docker restart
 	There is still a bug here, though. The docs say to install the CA cert in /etc/docker/certs.d/<registry>, and clearly that isn't sufficient. In fact, after installing the certificate globally, I removed the one in /etc/docker/certs.d, restarted Docker, and it still worked.

还有一个操作，就是按照docker login时候的提示信息，需要把根证书放在client端的/etc/docker/certs.d/registryname:9443/文件夹之下，这个也还是要放进去，不然docker login的时候还是会找不到根证书。

之后可以 curl --cacert rootca.crt https://admin:admin@devregistry:9443/v1/search 使用rest api查看，或者docker login devregistry:9443 登录成功之后，就可以使用了，把image的tag加上devregistry:9443/的前缀之后，就能push, pull对应的镜像了。






