## 10054错误
git clone 出现错误 10054，原因是服务端使用的是未经过第三方签署的临时SSL证书，Git因为安全考虑阻止访问。
解决方案： git config --global http.sslVerify “false”