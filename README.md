# 1. Zabbix-Template-Memcache

>环境描述: 由于Memcached在大型应用中, 通常存在多个实例, 在本环境中, 一共存在两个实例(11211,11212), 如果采用固定端口进行监控, 监控模版会很不灵活, 那么就需要lld自动发现来监控.

## 2. 自动发现多端口

### 2.1. 模板自动发现列表



### 2.2 自动发现端口脚本
```sh
shell> cat memcache_low_discovery.sh
#!/bin/bash
#Fucation:zabbix low-level discovery
memcache() {
            port=($(sudo netstat -tpln | awk -F "[ :]+" '/[m]emcached/ && /0.0.0.0/ {print $5}'))
            printf '{\n'
            printf '\t"data":[\n'
               for key in ${!port[@]}
                   do
                       if [[ "${#port[@]}" -gt 1 && "${key}" -ne "$((${#port[@]}-1))" ]];then
                          printf '\t {\n'
                          printf "\t\t\t\"{#MEMPORT}\":\"${port[${key}]}\"},\n"
                     else [[ "${key}" -eq "((${#port[@]}-1))" ]]
                          printf '\t {\n'
                          printf "\t\t\t\"{#MEMPORT}\":\"${port[${key}]}\"}\n"
                       fi
               done
                          printf '\t ]\n'
                          printf '}\n'
}
$
```

---
### 2.3. 设置脚本权限

设置自动发现脚本的权限为755, 属主(组)为zabbix, 允许zabbix无密码运行netstat, 禁用Disable requiretty.
```sh
shell> pwd
/Zabbix/bin
shell> chmod 755 memcache_low_discovery.sh
shell> chown zabbix:zabbix memcache_low_discovery.sh
shell> ll
-rwxr-xr-x 1 zabbix zabbix    813 Apr  5 09:34 memcache_low_discovery.sh
shell> echo "zabbix ALL=(root) NOPASSWD:/bin/netstat">>/etc/sudoers 
shell> sed -i 's/^Defaults.*.requiretty/#Defaults    requiretty/'/etc/sudoer
```

---
### 2.4. 设置自定义Key

修改zabbix_agentd.conf配置文件, 增加以下内容:
```sh
shell> vi /usr/local/etc/zabbix_agent.com
UserParameter=memcached_stats[*],(echo stats; sleep 0.1) | telnet 127.0.0.1 $1 2>&1 | awk '/STAT $2 / {print $NF}'
UserParameter=zabbix_low_discovery[*],/bin/bash /usr/local/zabbix/bin/memcache_low_discovery.sh $1
```
重启Zabbix Agent进行, 使配置得以生效
```sh
shell> /etc/init.d/zabbix_agentd restart
```

---
## 3. 导入模板
导入模板Template_App_Memcached_Service_Chinese.xml, 并配置正则表达式: 


> 名称: Memcache regex 
表达式类型: 结果为真 
表达式: ^(11211|11212)$ 
解释: 获取的值如若是匹配11211|11212, 则匹配关系成立, 结果为真, 如果还有其他实例11213,11214, 则表达式为:^(11211|11212|11213|11214)$, 根据个人环境, 依次类推!

---
## 4. 测试数据获取

主机关联模版, 默认数据为3600s后自动更新, 并在Zabbix服务器端测试获取数据
```sh
shell> zabbix_get -s 10.66.1.52 -k zabbix_low_discovery[memcache]
{
"data":[
 {
"{#MEMPORT}":"11211"}
 ]
}
shell> zabbix_get -s 10.66.1.52 -k memcached_stats[11211,pid]
301
```

---
## 5. 模板下载
> Template_App_Memcached_Service_Chinese.xml
