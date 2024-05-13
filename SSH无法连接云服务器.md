问题：
    无法使用华为云CloudShell连接服务器
登录提示：
    无法连接到远程主机,请检查主机网络与端口是否正确
解决：
    Ubuntu安装防火墙(Uncomplicated Firewall)后，未开启22端口
        查看已开启端口
        sudo ufw status
        添加22端口
        sudo ufw allow 22
        或
        sudo ufw allow ssh