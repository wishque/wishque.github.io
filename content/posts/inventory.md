---
title: "ansible(二) Inventory"
date: 2023-05-23T15:54:19+08:00
categories: [ansible]
---
ansible能够在多个节点上自动化执行tasks，这些节点通常被叫做inventory。

1. inventory可以直接在命令行书写, 也可以保存在一个文件中

```bash
#直接书写hosts
$ ansible -i 127.0.0.1,127.0.0.2 --list-hosts all
  hosts (2):
    127.0.0.1
    127.0.0.2

#通过文件读取inventory
$ cat <<eof > inventory
127.0.0.1
127.0.0.2
eof
$ ansible -i inventory --list-hosts all
  hosts (2):
    127.0.0.1
    127.0.0.2
```

2. inventory中不仅可以定义节点，还可以定义节点所属于的组。方便后续通过pattern选择要执行task的节点
3. inventory默认存放在/etc/ansible/hosts（可以通过配置文件修改）中，可以通过-i <path>指定inventory存放的位置
3. 为了更加灵活的管理inventory,除了书写单个的inventory文件，ansible还提供了三种方式：
    1. 在inventory目录下，存放多个inventory文件(yaml格式，ini格式，等等)
    2. 动态拉取（生成）inventory
    3. 结合上述两种方式

    可以在配置文件中配置plugin来支持更多的文件格式(toml等)，和inventory源（从云厂商拉取等）
4. 多个inventory文件加载顺序:
    1. 在inventory文件夹中,ansible根据inventory文件名的字典序加载inventory。
    2. 使用-i <path1> -i <path2> ... 来指定躲个inventory文件时,按指定顺序加载

## 基本格式

inventory最基础的格式是ini和yaml。

in INI:

```ini
#注释

#默认group为ungrouped
127.0.0.1

#组webserver中有一个节点foo.example.com
[webservers]
foo.example.com
```

in YAML:

```yaml
#所有的节点都属于all组
all:
    hosts:
        127.0.0.1
    children:
        webservers:
            hosts:
                foo.example.com
```

我们写了一个127.0.0.1,没有在任意的group下,他默认呢属于ungrouped.而foo.example.com属于webservers组

```bash
$ ansible-inventory -i inventory  --graph
@all:
  |--@ungrouped:
  |  |--127.0.0.1
  |--@webservers:
  |  |--foo.example.com
```
## hosts

可以通过书写hostname、ip来指定host节点,每个host节点可以属于多个不同的组。

可以通过[start:stop:step]来生成重复的hosts。

in INI:

```ini
foo[01:03:2].example.com
foo[a:b].example.com
```

in YAML:

``` yaml
all:
    hosts:
        foo[01:03:2].example.com
        foo[a:b].example.com
```

[01:03:2]表示从01开始到03每次加2,所以foo[01:03:2].example.com会展开为foo01.example.com,foo03.exmaple.com.同理[a:b]表示从a到b每次加1,foo[a:b].example.com展开为fooa.example.com,foob.example.com

```bash
$ ansible -i inventory --list-hosts all
  hosts (4):
    foo01.example.com
    foo03.example.com
    fooa.example.com
    foob.example.com
```

## groups

每个group下除了可以包含host节点，还可以包含子group。父group组包含所有子group组下的hosts。

in INI:

```ini
127.0.0.1

[webservers]
foo.example.com

[webservers:children]
nginx

[nginx]
bar.example.com
```

in YAML:

```yaml
all:
    children:
        webservers:
            hosts:
                foo.example.com
            children:
                nginx:
                    hosts:
                        bar.exmaple.com
```

127.0.0.1属于ungrouped组,foo.example.com属于webservers组,webservers组还有一个子group叫做nginx.nginx组下有一个host bar.example.com

```bash
$ ansible -i inventory --list-hosts webservers
  hosts (2):
    foo.example.com
    bar.exmaple.com
```

## variables

1. 同一个组的所有节点会继承group variables
2. 子group会继承来自父group的variables,父子group同名的variables,子group覆盖父group
3. 同名variables按加载顺序,后加载的覆盖先加载的

in INI:

```ini
127.0.0.1

[webservers]
127.0.0.2

[webservers:children]
nginx

[nginx]
127.0.0.4 num=4
127.0.0.3

[all:vars]
num=1

[webservers:vars]
num=2

[nginx:vars]
num=3
```

in YAML:

```yaml
all:
    hosts:
        127.0.0.1
    vars:
        num: 1
    children:
        webservers:
            hosts:
                127.0.0.2
            vars:
                num: 2
            children:
                nginx:
                    hosts:
                        127.0.0.3:
                        127.0.0.4:
                            num: 4
                    vars:
                        num: 3
```

127.0.0.1属于组all,所以他继承了all的变量 num=1
127.0.0.2他属于webservers,all组,由于webservers是all的子组,所以127.0.0.1先继承all的num=1,然后继承webservers的num=2,所以左后num=2
同理127.0.0.3 num=3
127.0.0.4由于定义了一个属于127.0.0.4的变量,他最后被加载.所以num=4

```bash
$ ansible-inventory -i inventory --vars --graph
@all:
  |--@ungrouped:
  |  |--127.0.0.1
  |  |  |--{num = 1}
  |--@webservers:
  |  |--@nginx:
  |  |  |--127.0.0.4
  |  |  |  |--{num = 4}
  |  |  |--127.0.0.3
  |  |  |  |--{num = 3}
  |  |  |--{num = 3}
  |  |--127.0.0.2
  |  |  |--{num = 2}
  |  |--{num = 2}
  |--{num = 1}
```
### 加载顺序

1. all group (all group 是所有group的父group)
2. 父group
3. 子group
4. host

默认情况下,同一级的group变量加载顺序是按照字典序的.所以后加载的group variables会覆盖先加载的.可以通过设置`ansible_group_priority`来改变同一级group的加载顺序,数字越大,越迟加载,也就是说,变量优先级越高

### INI格式variables类型问题

在使用INI格式定义variables时,variables值的类型可能出现问题:

- 当定义host时,在同一行定义variables时,值的类型按照python代码解析,可以同时定义多个key=val(以空格分隔),如果值中含有空格,需要以单引号或双引号包裹
- 当在:vars段定义variables时,值按照字符串解析,例如var=FALSE,variables值为字符串"FALSE",每行一个key=val

```ini
localhost var1=FALSE var2="a b" var3=[1,2]

[all:vars]
var=FALSE
```

```yaml
- hosts: all
  gather_facts: false
  connection: local
  tasks:
      - name: check var  type
        debug:
            msg: "{{var|type_debug}}"
      - name: check var1  type
        debug:
            msg: "{{var1|type_debug}}"
      - name: check var3  type
        debug:
            msg: "{{var3|type_debug}}"
```

我们写了一个playbook这个playbook会对所有的hosts执行tasks.一共有三个task,第一个输出var的类型,第二个输出var1的类型,第三个输出var3的类型.
我们在inventory文件中定义var=FALSE,var1=FALSE,写的都是FALSE但是最后的类型一个是bool一个是str.而var3我们写了一个list,是正常解析的.

```bash
$ ansible-playbook -i inventory p.yaml

PLAY [all] ***********************************************************************************************

TASK [check var  type] ***********************************************************************************
ok: [localhost] => {
    "msg": "bool"
}

TASK [check var1  type] **********************************************************************************
ok: [localhost] => {
    "msg": "str"
}

TASK [check var3  type] **********************************************************************************
ok: [localhost] => {
    "msg": "list"
}

PLAY RECAP ***********************************************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## pattern

### 常用写法

| Description | Pattern(s)               | Targets                                      |
| ----------- | -------------------------|----------------------------------------------|
| 所有hosts   | all(*)                  ||
| 单个host    | host1                    ||
| 多个hosts   | host1:host2(host1,host2)||
| 单个group   | webservers               ||
| group的并集 | webservers:dbservers     |webserver组的所有hosts和dbservers组的所有hosts|
| group的差集 | webservers:!atlanta      |webserver组的所有hosts除去atlanta组的所有hosts|
| group的交集 | webservers:&staging      |同时在webservers组和staging组的hosts          |

写host,group时可以使用通配符

### 运算符优先级

从上往下计算,最上面的优先级最高

1. : 和 ,
2. &
3. !

### 进阶写法

[ansible doc](https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html#id6)

- 正则
- 模板解析pattern
- 将group作为python列表

## ansible-inevntory命令

`ansible-inventory [...] <ACTIONS> [host|group]`

actions:
* \-\-graph [GROUP] 生成inventory图表
* \-\-host <HOST> 输出指定host的信息,可以作为inventory script(输出json)
* \-\-list 输出所有hosts的信息,可以作为inventory script(输出json)

常用的选项:
* \-\-vars 在输出图表时,输出variables信息
* \-i <INVENTORY> 指定inventory

## 源码
### main
ansible-inventory的main函数存放在bin目录下

```python
class InventoryCLI(CLI):
    ...

def main(args=None):
    InventoryCLI.cli_executor(args)


if __name__ == '__main__':
    main()
```

InventoryCli.cli_executor函数来自继承的CLI类

```
class CLI(ABC):
    @classmethod
    def cli_executor(cls,args=None):
        #创建C.ANSIBLE_HOME文件夹,这是ansible配置文件的根目录
        ...
        #将args按utf8格式转化为文本
        ...
        cli=cls(args)
        exit_code=cli.run()
        #错误处理
        ...
```

可以看到真正做事的函数是run函数,我们看InventoryCLI.run函数

``` python
class InventoryCLI(CLI):
    def run(self):
        #运行父类的run函数,进行了一些命令行参数的解析"-v"
        super(InventoryCLI,self).run()

        # 初始化inventory
        self.loader, self.inventory, self.vm = self._play_prereqs()

        #按照命令行参数输出格式化结果,主要调用的是vm.get_vars,inventory.get_hosts
            ...
            #调用了vm.get_vars
            myvars = self._get_host_variables(host=hosts[0])
            ...

            ...
            #调用了inventory.get_hosts
            self.inventory.subset(context.CLIARGS['subset'])
                ...
                results = self.inventory_graph()
                ..
                top = self._get_group('all')
                ...
                    results = self.yaml_inventory(top)
                ...
                    results = self.toml_inventory(top)
                ...
                    results = self.json_inventory(top)
                results = self.dump(results)
        ...

class CLI(ABC):
    @staticmethod
    def _play_prereqs():
        ...

        # all needs loader
        loader = DataLoader()

        #loader的一些配置,包含plugins,default_collection,vault
        ...
        # create the inventory, and filter it based on the subset specified (if any)
        inventory = InventoryManager(loader=loader, sources=options['inventory'], cache=(not options.get('flush_cache')))

        # create the variable manager, which will be shared throughout
        # the code, ensuring a consistent view of global variables
        variable_manager = VariableManager(loader=loader, inventory=inventory, version_info=CLI.version_info(gitinfo=False))

        return loader, inventory, variable_manager
```

可以看到inventory的解析包含了三个类,DataLoader,InventoryManager,VariableManager.

### InventoryManager

``` python
class InventoryManager(object):
    def __init__(self, loader, sources=None, parse=True, cache=True):

    # base objects
    #实际加载数据类
    self._loader = loader
    #inventory数据类
    self._inventory = InventoryData()
        def __init__(self, loader, sources=None, parse=True, cache=True):

        # base objects
        self._loader = loader
        self._inventory = InventoryData()

        # 解析inventory sources文件
        if parse:
            self.parse_sources(cache=cache)
        ...

    def parse_sources(self, cache=False):

        parsed = False
        #按照命令行参数传递顺序解析inventory文件
        for source in self._sources:

            if source:
                #如果不是传递inventory字符串的话 -i 127.0.0.1,127.0.0.2
                if ',' not in source:
                    source = unfrackpath(source, follow=False)
                #解析单个inventory文件
                parse = self.parse_source(source, cache=cache)
                if parse and not parsed:
                    parsed = True

        #整合group,host,vars关系
        if parsed:
            # do post processing
            self._inventory.reconcile_inventory()
        else:
            if C.INVENTORY_UNPARSED_IS_FAILED:
                raise AnsibleError("No inventory was parsed, please check your configuration and options.")
            elif C.INVENTORY_UNPARSED_WARNING:
                display.warning("No inventory was parsed, only implicit localhost is available")

        for group in self.groups.values():
            group.vars = combine_vars(group.vars, get_vars_from_inventory_sources(self._loader, self._sources, [group], 'inventory'))
        for host in self.hosts.values():
            host.vars = combine_vars(host.vars, get_vars_from_inventory_sources(self._loader, self._sources, [host], 'inventory'))

    def parse_source(self, source, cache=False):
        ''' Generate or update inventory for the source provided '''

        parsed = False
        failures = []
        display.debug(u'Examining possible inventory source: %s' % source)

        # use binary for path functions
        b_source = to_bytes(source)

        #如果我们使用的是inventory目录的话
        if os.path.isdir(b_source):
            display.debug(u'Searching for inventory files in directory: %s' % source)
            #按字典序加载目录中的文件
            for i in sorted(os.listdir(b_source)):

                display.debug(u'Considering %s' % i)
                # Skip hidden files and stuff we explicitly ignore
                if IGNORED.search(i):
                    continue

                # recursively deal with directory entries
                fullpath = to_text(os.path.join(b_source, i), errors='surrogate_or_strict')
                #进入else分支,解析单个文件
                parsed_this_one = self.parse_source(fullpath, cache=cache)
                display.debug(u'parsed %s as %s' % (fullpath, parsed_this_one))
                if not parsed:
                    parsed = parsed_this_one
        else:
            # left with strings or files, let plugins figure it out

            # set so new hosts can use for inventory_file/dir vars
            self._inventory.current_source = source

            # try source with each plugin
            #寻找适合的plugin,C.INVENTORY_ENABLED
            for plugin in self._fetch_inventory_plugins():

                plugin_name = to_text(getattr(plugin, '_load_name', getattr(plugin, '_original_path', '')))
                display.debug(u'Attempting to use plugin %s (%s)' % (plugin_name, plugin._original_path))

                # initialize and figure out if plugin wants to attempt parsing this file
                try:
                    #正确的plugin
                    plugin_wants = bool(plugin.verify_file(source))
                except Exception:
                    plugin_wants = False

                if plugin_wants:
                    try:
                        #使用该plgin进行解析
                        plugin.parse(self._inventory, self._loader, source, cache=cache)
                        try:
                            plugin.update_cache_if_changed()
                        except AttributeError:
                            # some plugins might not implement caching
                            pass
                        parsed = True
                        display.vvv('Parsed %s inventory source with %s plugin' % (source, plugin_name))
                        break
                    except AnsibleParserError as e:
                        display.debug('%s was not parsable by %s' % (source, plugin_name))
                        tb = ''.join(traceback.format_tb(sys.exc_info()[2]))
                        failures.append({'src': source, 'plugin': plugin_name, 'exc': e, 'tb': tb})
                    except Exception as e:
                        display.debug('%s failed while attempting to parse %s' % (plugin_name, source))
                        tb = ''.join(traceback.format_tb(sys.exc_info()[2]))
                        failures.append({'src': source, 'plugin': plugin_name, 'exc': AnsibleError(e), 'tb': tb})
                else:
                    display.vvv("%s declined parsing %s as it did not pass its verify_file() method" % (plugin_name, source))

        if parsed:
            self._inventory.processed_sources.append(self._inventory.current_source)
        else:
            # only warn/error if NOT using the default or using it and the file is present
            # TODO: handle 'non file' inventory and detect vs hardcode default
            if source != '/etc/ansible/hosts' or os.path.exists(source):

                if failures:
                    # only if no plugin processed files should we show errors.
                    for fail in failures:
                        display.warning(u'\n* Failed to parse %s with %s plugin: %s' % (to_text(fail['src']), fail['plugin'], to_text(fail['exc'])))
                        if 'tb' in fail:
                            display.vvv(to_text(fail['tb']))

                # final error/warning on inventory source failure
                if C.INVENTORY_ANY_UNPARSED_IS_FAILED:
                    raise AnsibleError(u'Completely failed to parse inventory source %s' % (source))
                else:
                    display.warning("Unable to parse %s as an inventory source" % source)

        # clear up, jic
        self._inventory.current_source = None

        return parsed

```

我们可以看到 InventoryManager类对于从命令行参数传递过来的sources,按照顺序进行解析.如果该source是目录,则按照字典序依次解析inventory文件.解析单个文件时,依次搜索通过INVENTORY_ENABLED配置添加的plugin,找到正确的plugin后,通过plugin解析该文件.

#### plugins

我们可以在lib/ansible/plugins/inventory目录下找到官方的plugin,包括advanced_host_list,auto,constructed,generator,host_list,ini,script,toml,yaml.可以看一下ini文件是如何解析的.其实是和tag0.0.1相似的,十分的简单

``` python
class InventoryModule(BaseFileInventoryPlugin):
    def parse(self, inventory, loader, path, cache=True):
        ...
                (b_data, private) = self.loader._get_file_contents(path)
        ...
            self._parse(path, data)

    def _parse(self, path, lines):

        self._compile_patterns()


        pending_declarations = {}
        groupname = 'ungrouped'
        state = 'hosts'
        self.lineno = 0
        for line in lines:
            self.lineno += 1

            line = line.strip()
            # Skip empty lines and comments
            if not line or line[0] in self._COMMENT_MARKERS:
                continue
            #如果该行为[group_name:xxx] 
            m = self.patterns['section'].match(line)
            if m:
                #[group_name:children]
                (groupname, state) = m.groups()

                groupname = to_safe_group_name(groupname)

                state = state or 'hosts'
                if state not in ['hosts', 'children', 'vars']:
                    title = ":".join(m.groups())
                    self._raise_error("Section [%s] has unknown type: %s" % (title, state))

               #如果要为group添加hosts,children,vars但是group不存在,添加group
                if groupname not in self.inventory.groups:
                    if state == 'vars' and groupname not in pending_declarations:
                        pending_declarations[groupname] = dict(line=self.lineno, state=state, name=groupname)

                    self.inventory.add_group(groupname)
                if groupname in pending_declarations and state != 'vars':
                    if pending_declarations[groupname]['state'] == 'children':
                        self._add_pending_children(groupname, pending_declarations)
                    elif pending_declarations[groupname]['state'] == 'vars':
                        del pending_declarations[groupname]

                continue
            elif line.startswith('[') and line.endswith(']'):
                self._raise_error("Invalid section entry: '%s'. Please make sure that there are no spaces" % line + " " +
                                  "in the section entry, and that there are no other invalid characters")

            #add host
            if state == 'hosts':
                hosts, port, variables = self._parse_host_definition(line)
                self._populate_host_vars(hosts, variables, groupname, port)
            #add vars
            elif state == 'vars':
                (k, v) = self._parse_variable_definition(line)
                self.inventory.set_variable(groupname, k, v)
            #add children
            elif state == 'children':
                child = self._parse_group_name(line)
                if child not in self.inventory.groups:
                    if child not in pending_declarations:
                        pending_declarations[child] = dict(line=self.lineno, state=state, name=child, parents=[groupname])
                    else:
                        pending_declarations[child]['parents'].append(groupname)
                else:
                    self.inventory.add_child(groupname, child)
            else:
                # This can happen only if the state checker accepts a state that isn't handled above.
                self._raise_error("Entered unhandled state: %s" % (state))

        for g in pending_declarations:
            decl = pending_declarations[g]
            if decl['state'] == 'vars':
                raise AnsibleError("%s:%d: Section [%s:vars] not valid for undefined group: %s" % (path, decl['line'], decl['name'], decl['name']))
            elif decl['state'] == 'children':
                raise AnsibleError("%s:%d: Section [%s:children] includes undefined group: %s" % (path, decl['line'], decl['parents'].pop(), decl['name']))

class DataLoader:
    def _get_file_contents(self, file_name):
    ...
        try:
            with open(b_file_name, 'rb') as f:
                data = f.read()
                return self._decrypt_if_vault_data(data, b_file_name)
        except (IOError, OSError) as e:
            raise AnsibleParserError("an error occurred while trying to read the file '%s': %s" % (file_name, to_native(e)), orig_exc=e)


```

可以看到,加载ini文件的流程非常简单.先通过DataLoader类读取文件,需要时通过vault解密文件.然后按行读取文件.正则匹配到某一行为[group_name:state]时,改变state.其余行依照当前state为group_name添加hosts,children,vars.

#### InventoryData
    InventoryData类是内存中的inventory,这个类和tag0.0.1相比没有很大的区别.
``` python
class InventoryData(object):
    def __init__(self):

        #数据存放类型为字典,groups={group_name:Group} hosts={host_name:Host}
        self.groups = {}
        self.hosts = {}

        # provides 'groups' magic var, host object has group_names
        self._groups_dict_cache = {}

        # current localhost, implicit or explicit
        self.localhost = None

        self.current_source = None
        self.processed_sources = []

        #默认存在all,ungrouped两个group
        for group in ('all', 'ungrouped'):
            self.add_group(group)
        #所有的group都是all的子group,包括ungrouped
        self.add_child('all', 'ungrouped')

class Group:
        self.depth = 0
        self.name = to_safe_group_name(name)
        #group下的hosts存放为列表
        self.hosts = []
        self._hosts = None
        self.vars = {}
        self.child_groups = []
        self.parent_groups = []
        self._hosts_cache = None
        self.priority = 1

class Host:
    def __init__(self, name=None, port=None, gen_uuid=True):
        #vars
        self.vars = {}
        #所属的groups
        self.groups = []
        self._uuid = None

        self.name = name
        self.address = name

        if port:
            self.set_variable('ansible_port', int(port))

        if gen_uuid:
            self._uuid = get_unique_id()
        self.implicit = False
```


## 总结

非常容易

