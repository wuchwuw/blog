生成ssh key

ssh-keygen -t rsa

添加到授信列表

cd .ssh/
cat id_rsa.pub >> authorized_keys
cat authorized_keys

测试登录

在.ssh下添加config文件

vim config

保存以下代码

Host test
HostName 99.99.99.99(你的服务器ip)
#登陆的用户名
User travis
IdentitiesOnly yes
#登陆使用的密钥
IdentityFile ~/.ssh/id_rsa

执行 ssh test 测试连接