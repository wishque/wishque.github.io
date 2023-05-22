---
title: "Ansible"
date: 2023-05-22 19:33:55
summary: 
draft: true
---
> （第一版ansible README）Ansible is a extra-simple Python API for doing 'remote things' over SSH.
## 基本流程

[第一版ansible](https://github.com/ansible/ansible/commit/f31421576b00f0b167cdbe61217c31c21a41ac02)
<details>
<summary>代码简化后如下:</summary>

```python
if __name__=="__main__":
    # 解析命令行参数
    ...
    host_list=file(host_list).read().split("\n")
    # 匹配正则表达式的host
    hosts=[host for host in host_list if matches(host)]
    for host in hosts:
        #多线程运行executor(host),将返回的结果收集为列表
        pooler.parmap(executor,host)

def executor(host):
    #通过python的ssh库paramiko连接到host
    conn=connect(host)
    #通过sftp将文件copy到host的某个目录中，返回文件完整路径
    outpath=copy_module(conn)
    #添加可执行权限
    exec_command("chmod +x %s",outpath)
    #执行./outpath module_args
    result=exec_command("%s %s",outpath,module_args)
    #将命令的执行结果（状态码，标准输出，标准错误）转化为json字符串
    result=json.loads(result)
    return [host,result]
```
</details>

虽说在第一次commit之后十年，ansible已经被修改了几十万次,但是ansible做的事情没有变.
1. 连接host（通常通过ssh）
2. 复制module文件到host的临时文件夹（通常通过sftp,scp）
3. 执行该module（一般是python文件）
4. 删除该module

通过添加`ANSIBLE_KEEP_REMOTE_FILES=1`环境变量，可以让ansible保留module文件（默认位于~/.ansible/tmp/文件夹下)。例如执行下面这个命令后，可以在该文件夹下找到AnsiballZ\_ping.py文件。
```
ANSIBLE_KEEP_REMOTE_FILES=1 ansible -m ping  localhost
```
由于执行模块有可能是异步的，这会导致删除模块和执行模块同时进行。所以ANsiballZ\_ping.py文件会执行位于系统临时文件夹的模块文件，临时文件由系统在ansible执行后自动删除，这是为了修复该问题。 可以在执行下面命令后在debug_dir目录下找到真正的模块文件。
```
python3 AnsibllZ_ping.py explode
```

<details>
<summary>内容如下:</summary>

```python
from ansible.module_utils.basic import AnsibleModule


def main():
    #申明模块，该模块仅一个参数data,str类型,默认值为pong
    module = AnsibleModule(
        argument_spec=dict(
            data=dict(type='str', default='pong'),
        ),
        supports_check_mode=True
    )
    #如果data=="crash",返回错误boom
    if module.params['data'] == 'crash':
        raise Exception("boom")

    #正常返回pong
    result = dict(
        ping=module.params['data'],
    )

    module.exit_json(**result)


if __name__ == '__main__':
    main()
```
</details>

由于ping是官方模块，该文件也可以在[ansible仓库](https://github.com/ansible/ansible/blob/devel/lib/ansible/modules/ping.py)下找到。
此外，为了更好的管理hosts和执行module，ansible还提供了inventory，playbook。
- inventory: 管理hosts
- playbook: 管理module的在host上的执行顺序

## modules

在modules目录下，我们还可以看到所有的官方模块，我们在另一篇文章会记录下学习过程。包括
<details> 
<summary>ansible运行时管理</summary>

    - add\_host:在当前playbook内存中inventory中添加host（或者group）
    - import\_playbook:引用playbook
    - import\_role:引用role
    - import\_tasks:引用tasks
    - include\_role:加载并执行role
    - include\_tasks:加载tasks,与import\_tasks在loop,with\_items,conditions等关键词一起使用时有区别
    - include\_vars:在运行task时，动态的从文件中加载variables
    - group\_by:基于fact创建群组
    - meta:执行ansible actions
</details>

<details>
<summary>linux包管理</summary>

    - apt:管理apt包
    - apt\_key:添加或删除apt key
    - apt\_repository:添加或删除apt仓库
    - deb822_repository:添加或删除deb882格式的仓库
    - debconf:配置.deb包
    - dpkg\_selections::dpkg包选项

    - dnf:通过dnf包管理器管理包
    - dnf5:通过dnf5包管理器管理包(开发中)

    - rpm\_key:在rpm数据库中添加或删除gpg key
    - yum:通过yum包管理器管理包
    - yum\_repository:添加或删除yum仓库

    - package:任意linux操作系统的包管理器
</details>

<details>
<summary>文件</summary>

    - blockinfile:添加、修改、删除标记行包围的字符块
    - lineinfile:管理字符文件中的行
    - assemble:用碎片组合配置文件
    - template:解析模板文件到目标主机
    - replace:替换一个文件中所有匹配的字符串(back-referenced)

    - file:管理文件和文件属性
    - find:基于某些原则返回文件列表
    - stat:获取文件或文件系统状态
    - tempfile:创建临时文件和目录
    - unarchive:解压压缩文件（可选从本地复制后）

    - copy:复制文件到远程位置
    - slurp:从远程主机上获取文件 （base64加密）
    - fetch:从远程节点取得文件
    - get\_url:从HTTP,HTTPS或者FTP下载文件到节点
</details>

<details>
<summary>系统管理</summary>

    - group:添加或删除用户组
    - user:管理用户账号

    - systemd|systemd\_service: 管理systemd units
    - sysvinit:管理Sysv服务
    - service:管理services服务

    - cron:管理cron.d和crontab任务
    - hostname:管理hostname
    - iptables:修改iptables规则
    - pip:管理python包依赖
    - getent:unix getent命令的包装
    - known\_hosts:从kwnon\_hosts文件添加或删除host
    - reboot:重启机器
</details>

<details>
<summary>facts</summary>

    - setup:收集远程主机的facts
    - set\_fact:设置host variable和fact
    - gather\_facts:收集关于远程hosts的fact信息
    - package\_facts:包的信息作为fact
    - service\_facts:返回service状态信息作为fact数据
</details>

<details>
<summary>执行控制</summary>

    - assert:判断表达式是否正确
    - async\_status:获取异步task的状态
    - pause:暂停playbook执行
    - debug:在执行时打印语句
    - fail:失败并且打印信息
    - set\_stats:为本次ansible执行设置并展示stats
    - wait\_for:等待条件为真后执行
    - wait\_for\_connection:等待远程系统可达(可用)
    - validate\_argument\_spec:
</details>

<details>
<summary>版本管理</summary>

    - git:从git checkout部署软件（或文件）
    - subversion:部署子版本仓库
</details>

<details>
<summary>执行命令</summary>

    - raw:执行脏的命令
    - command:在目标机器执行命令
    - script:移动本地scripts到远程节点并执行
    - shell:在主机上执行shell命令
    - expect:执行命令并且回应promot提示符
</details>

<details>
<summary>web</summary>

    - uri:和web服务沟通
    - ping:尝试连接host,检查python是否可用。成功返回pong
</details>

## inventory



