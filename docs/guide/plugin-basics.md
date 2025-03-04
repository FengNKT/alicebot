# 插件基础

## 加载插件

在 [快速上手](./getting-started.md) 一章中我们创建了一个 `plugins` 目录，并在配置文件中设置其为插件目录。所以，放入 `plugins`
目录的任何不以 `_` 开头的 Python
模块都会被自动加载并测试是否是插件。你也可以不配置 `plugin_dirs` 并通过以下方式“程序式”地加载插件。

但通常情况下，不需要使用下面的方法，推荐通过配置 `plugin_dirs` 或 `plugins` 配置项来加载插件。

### 加载插件目录

在 `main.py` 文件中添加以下行：

```python {4}
from alicebot import Bot

bot = Bot()
bot.load_plugins_from_dirs(["plugins", "/home/xxx/alicebot/plugins"])

if __name__ == "__main__":
    bot.run()

```

这实际上和配置 `plugin_dirs` 为 `["plugins", "/home/xxx/alicebot/plugins"]`
是相同的，目录可以是相对路径或者绝对路径。以 `_` 开头的插件不会被加载。

### 加载单个插件

在 `main.py` 文件中添加以下行：

```python {7}
from alicebot import Bot

class TestPlugin(Plugin):
    pass

bot = Bot()
bot.load_plugins("plugins.hello", TestPlugin)

if __name__ == "__main__":
    bot.run()

```

`load_plugins` 方法可以传入插件类、字符串或者 `pathlib.Path` 对象，如果为后两者会分别视为插件模块的名称（格式和 Python `import` 语句相同）和插件模块文件的路径进行加载。

## 编写插件

每个插件即是一个插件类。

虽然并非强制，但建议将一个插件类存放在一个单独的 Python 模块中。

```text
.
├── plugins
│   ├── a.py (单个 Python 文件)
│   └── b (Python 包)
│      └── __init__.py
├── config.toml
└── main.py
```

插件类都必须是 `Plugin` 类的子类，并必须实现 `rule()` 和 `handle()` 方法。

```python {5-6}
from alicebot import Plugin


class TestPlugin(Plugin):
    priority: int = 0
    block: bool = False

    async def handle(self) -> None:
        pass

    async def rule(self) -> bool:
        return True

```

其中类名 `TestPlugin` 建议不要重复。

`priority` 属性表示插件的优先级，数字越小表示优先级越高。

`block` 属性表示当前插件执行结束后是否阻止事件的传播，如果设置为 `True` ，则当前插件执行完毕后会停止当前事件传播，比当前插件优先级低的插件将不会被执行。

以上两个属性都是可选的，默认为 `0` 和 `False` 。

如 [它是如何工作的？](./#它是如何工作的？) 一节中所说，当协议适配器产生一个事件（比如机器人接收到了一条消息）后，会按照优先级分发事件给各个插件，AliceBot
会依次执行每个插件的 `rule()` 方法，根据返回值判断是否要执行 `handle()`
方法。

`Plugin` 类内置了一些属性和方法：

- `self.event` ：当前正在被此插件处理的事件。
- `self.name` ：插件类名称。
- `self.bot` ：机器人对象。
- `self.config` ：插件配置。
- `self.state` ：插件状态。
- `self.stop()` ：停止当前事件传播。
- `self.skip()` ：跳过自身继续当前事件传播。

除了 `self.event` 之外的属性和方法将在 [插件进阶](./plugin-advanced) 一节中详细介绍。

不同适配器产生的事件是不同的，下文以 CQHTTP 适配器为例编写一个 Hello 插件。

### 编写 `rule()` 方法

```python {9-13}
from alicebot import Plugin


class HalloAlice(Plugin):
    async def handle(self) -> None:
        pass

    async def rule(self) -> bool:
        return (
            self.event.adapter.name == "cqhttp"
            and self.event.type == "message"
            and str(self.event.message).lower() == "hello"
        )

```

如果你觉得把所有条件都写在一行不够好看的话也可以这样写：

```python {9-13}
from alicebot import Plugin


class HalloAlice(Plugin):
    async def handle(self) -> None:
        pass

    async def rule(self) -> bool:
        if self.event.adapter.name != "cqhttp":
            return False
        if self.event.type != "message":
            return False
        return str(self.event.message).lower() == "hello"

```

由于不同的适配器产生的事件是不同的，所以应该先判断产生当前事件的适配器名称。

然后应该判断当前事件的类型，CQHTTP 适配器产生的事件的类型有：`message` 、 `notice` 和 `request` ，只有 `message`
类型的适配器才有 `message` 属性，这个插件只对消息事件进行响应。

CQHTTP 适配器消息事件的 `message` 属性表示当前接收到的消息，类型是 `CQHTTPMessage` ，是 AliceBot 内置 `Message`
类的子类。

AliceBot 内置的 `Message` 类实现了许多实用的方法，建议所有适配器开发者尽可能使用，具体使用在 [插件进阶](./plugin-advanced.md)
中有提到。在这里可以直接使用 `str()`
函数将 `Message` 类型的 `self.event.message` 转换为字符串。

除此之外，在 `rule()` 方法中常用的还有 `self.event.message.startswith('xxx')`
和  `self.event.message.endswith('xxx')`
。相当于字符串的 `startswith()` 和 `endswith()` 方法。

### 编写 `handle()` 方法

```python {6}
from alicebot import Plugin


class HalloAlice(Plugin):
    async def handle(self) -> None:
        await self.event.reply("Hello, Alice!")

    async def rule(self) -> bool:
        if self.event.adapter.name != "cqhttp":
            return False
        if self.event.type != "message":
            return False
        return str(self.event.message).lower() == "hello"

```

正如上面所说的，当 `rule()` 方法返回 `True` 时， `handle()` 方法将会被调用。这里，我们使用了 `message`
类型的事件的一个方法，`reply()`
用于快捷回复当前消息，而不需要额外指定发送消息的接收者。

`reply()` 方法是一个异步方法，所以在调用它时必须加上 `await` ，表示等待它直到返回结果。

下面，让我们再看一个例子来学习更多用法。

## 示例：天气插件

```python
from alicebot import Plugin
from alicebot.exceptions import GetEventTimeout


class Weather(Plugin):
    async def handle(self) -> None:
        args = self.event.get_plain_text().split(" ")
        if len(args) >= 2:
            await self.event.reply(await self.get_weather(args[1]))
        else:
            await self.event.reply("请输入想要查询天气的城市：")
            try:
                city_event = await self.event.adapter.get(
                    lambda x: x.type == "message", timeout=10
                )
            except GetEventTimeout:
                return
            else:
                await self.event.reply(
                    await self.get_weather(city_event.get_plain_text())
                )

    async def rule(self) -> bool:
        if self.event.adapter.name != "cqhttp":
            return False
        if self.event.type != "message":
            return False
        return self.event.message.startswith("天气")

    @staticmethod
    async def get_weather(city):
        if city not in ["北京", "上海"]:
            return "你想查询的城市暂不支持！"
        return f"{city}的天气是..."

```

你可以通过向机器人发送 `天气 北京` 格式的消息来获取天气信息，同时，也可以只发送 `天气` ，这时机器人会向你询问城市，再次发送要查询的城市名称即可查询天气。

在这个例子中，插件需要去进一步获取接收到的下一条消息，这里，我们使用了 `get()`
方法，它的使用方法可以参考 [API 文档](/api/adapter/#Adapter.get) 。

简而言之，它就是用于获取符合条件的事件的，同样的，它也是一个异步方法，请使用 `await` 等待。

需要注意的是， `get()`
方法可以指定超时时间，以避免插件一直等待获取事件，当超时发生时，它会引发 `alicebot.exception.GetEventTimeout`
异常，请留心对这种情况的处理。

此外，这里的 `get_weather()` 方法并没有接入真正的天气数据，你可以使用任意的天气 API 代替，但请注意，在进行网络请求时需要使用基于协程异步的网络库，如
aiohttp 或 httpx ，而不能使用如 requests
等同步的网络请求库，这会导致程序被阻塞。

相信阅读到这里，你应该已经可以写出一个 AliceBot 插件了。接下来建议你继续阅读 [插件进阶](./plugin-advanced.md) 和你将要使用的适配器的教程。
