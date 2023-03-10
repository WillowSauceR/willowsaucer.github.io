# mc.py

## 简介

该文件模块位于 ``plugins/py/mc.py``，提供多种功能，包括监听器名字兼容，API修改，日志输出函数，配置文件操作函数等功能

### 内容

#### 1.监听器名字兼容

我们通过修改 ``mc.setListener``和 ``mc.removeListener``传入的监听器名字来实现兼容新旧监听器名字，例如

```python
def setListener(event: str, function: Callable[[object], Optional[bool]]) -> None:
    notContainer = True
    if event == "onJoin": # py插件内使用的监听器
        event = "onPlayerJoin" # BDSpyrunnerW 提供的监听器
    if event == "onOpenContainer": # 将打开容器解析为同时设置多个监听器以简化代码
        notContainer = False
        mco.setListener("onOpenChest", function) # 打开箱子
        mco.setListener("onOpenBarrel", function) # 打开木桶
    if notContainer:
        return mco.setListener(event, function)
```

#### 2.API修改

我们通过包装 ``BDSpyrunnerW``提供的 ``API``来修改想要传入的值并提供更为清晰的函数原型，例如

```python
def setCommandDescription(cmd:str, description:str, function: Callable[[object], Optional[bool]] = None) -> None:
    if function:
        return mco.setCommandDescription(cmd, description, function)
    else:
        return mco.setCommandDescription(cmd, description)
```

#### 3.日志输出函数

模块内提供了统一的日志输出接口来帮助开发者规范插件的控制台输出。

初始化类，这里使用 ``__name__``以使用插件的文件名作为输出日志名。假设我们当前编写的插件名为 ``myplugin.py``。

```python
logger = mc.Logger(__name__)
```

要产生第一条输出，我们编写了如下代码

```python
def testonServerStarted(e):
    logger.info("Listener onServerStarted")
mc.setListener("onServerStarted", testonServerStarted)
```

它将会在服务器完成启动时输出以下内容

```plaintext
14:23:07 INFO [myplugin] Listener onServerStarted
```

为了使输出更加多样化，我们修改了代码，如下

```python
def testonServerStarted(e):
    logger.warn("Listener onServerStarted", info="LOG")
mc.setListener("onServerStarted", testonServerStarted)
```

这将会产生如下输出

```plaintext
14:37:05 WARN [myplugin][LOG] Listener onServerStarted
```

#### 4.配置文件操作

我们使用``ConfigManager``类来简化对``json``配置文件的管理

可用的成员函数：

```python
def __init__(self, filename:str, folder:str = "", encoding="utf-8")
def read(self)
def save(self, config={})
def make(self, config={})
```

初始化参数

```plaintext
filename: 文件名，位于 plugins/py/<folder>/ 目录下，不需要添加文件后缀名".json"
```

```plaintext
folder: 保存配置文件的文件夹，位于 plugins/py/ 目录下，默认为filename
```

```plaintext
encoding: 文件编码，默认为utf-8
```

以下是示例代码，假设您的插件文件名为 ``myplugin.py``

```python
import mc
conf_mgr = mc.ConfigManager(__name__)
# 定义默认配置文件
def_conf = {}
def_conf['obj_1'] = 3
def_conf['obj_2'] = True
def_conf['i_am_3'] = "I am 3"
def_conf['the_4_obj'] = [{'name': 'this is 1'}, {'name': 'another_obj', 'type': 'str'}]
def_conf['i_am_none'] = None

# 创建配置文件，已有则不做任何操作
conf_mgr.make(def_conf)

# 读取配置文件
config = conf_mgr.read()

# 输出配置内容
logout(config['i_am_3'])
logout(config['the_4_obj'][1]['name'])
```

这将会在 ``plugins/py/myplugin/myplugin.json``中产生如下内容

```json
{
    "obj_1": 3,
    "obj_2": true,
    "i_am_3": "I am 3",
    "the_4_obj": [
        {
            "name": "this is 1"
        },
        {
            "name": "another_obj",
            "type": "str"
        }
    ],
    "i_am_none": null
}
```

并且在控制台输出以下内容

```plaintext
14:58:38 INFO [myplugin] I am 3
14:58:38 INFO [myplugin] another_obj
```

修改配置文件中的内容，在控制台上的输出也会相应的发生改变

#### 5.指针操作

为了简化使用函数接口修改值的操作，我们选用指针作为 ``C++``和 ``Python``之间的数据交换桥梁。
``Python``中的 ``BDSpyrunnerW``提供的指针通常是由一串数字表示的 ``int``类型，在文档中我们称其为 ``pointer``类型，需要使用 ``ctypes``库对其进行取值和修改操作。
当需要修改指针指向的值时，可以使用 ``mc``文件模块中提供的 ``Pointer``类，该类提供了简便的访问指针指向的内存数据并对其进行修改的方法。

以下为在玩家攻击时修改产生的伤害的示例：

```python
import mc
import ctypes

logger = mc.logger("ATK")

def onPlayerAttack(event):
    pointer = mc.Pointer(event['damage'], ctypes.c_float)
    damage = pointer.get()
    pointer.set(100)
    logger.info(
        f"{event['player'].name}: ",
        f"{damage} -> {pointer.get()} ({event['damage']})",
        info="onPlayerAttack"
    )

mc.setListener("onPlayerAttack", onPlayerAttack)
```

在初始化``mc.Pointer``时，需要传入指针和指针类型，``BDS``中的伤害值为``float``类型，因此传入``ctypes.c_float``。内存修改立即生效，因此在调用``set``成员函数后，不论是在``C++``还是``Python``中访问该内存，获得的值都是修改后的。这是该方法的基本原理。

在服务器内使用空手攻击生物，一般生物会被直接击杀，控制台会打印类似于如下内容

```plaintext
11:45:14 INFO [ATK][onPlayerAttack] SenpaiHomo: 1.0 -> 100.0 (114514001919810)
```

在指针类型为``c_char_p``时，获取和处理数据的方式稍有不同，使用``get``方法获取到的是``bytes``类型的数据，使用``set``方法设置值时也同样需要传入``bytes``类型的数据，考虑使用``.encode()``和``.decode()``完成``str``和``bytes``之间的转换，例如以下代码修改聊天消息：

```python
import mc
import ctypes

def on_player_chat(event):
    player = event['player']
    name_ptr = mc.Pointer(event['name_ptr'], ctypes.c_char_p)
    msg_ptr = mc.Pointer(event['msg_ptr'], ctypes.c_char_p)

    player_dimension = player.did
    player_pos = ", ".join([f"{int(num)}" for num in player.pos])

    msg = msg_ptr.get().decode()
    name_mod = ""
    msg_mod = f"[{player_dimension}][{player_pos}] <{player.name}> {msg}"

    name_ptr.set(name_mod.encode())
    msg_ptr.set(msg_mod.encode())

mc.setListener('onChatPkt', on_player_chat)
```

当玩家发送``test``消息后，则会在聊天框显示类似于如下内容：
```plaintext
[0][11, 45, 14] <SenpaiHomo> test
```

#### 6.文件监控

有时，我们需要监控文件内容的变化并在那之后执行相应的操作，``FileMonitor``类就提供这个功能，可以用于插件或配置文件的热重载等等。

可使用的成员函数：

```python
# 初始化，不传入任何参数则代表自动热重载所有插件
def __init__(self, path="plugins/py/", callback=reload, args=("",), interval=1)
# 启动监控
def start(self)
#停止监控
def stop(self)
```
