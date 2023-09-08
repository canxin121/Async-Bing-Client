Async Poe Client
功能优势  
- 像使用 Newbing 网页一样使用 API
- 支持传入图片进行图像识别
- 支持人类语言绘图
- 方便持久化存储会话数据
- 划分 WebSocket 通信（WSS）信息为自定义类,支持输出分类结果,便于后续开发者封装功能
- 高并发处理能力
- 快速获取聊天数据
- 批量删除会话功能
- 无需 Tun 模式代理
- 支持HTTP, SOCKS 代理

# 使用说明

## 目录

[步骤一:创建 bing client](#步骤一创建-bing-client)

[步骤二: 使用client](#步骤二-使用client)

- [1].[创建新的对话](#1创建新的对话)

- [2]. [获取所有缓存的chat列表](#2-获取所有缓存的chat列表)

- [3]. [重新加载缓存的chat列表](#3-重新加载缓存的chat列表)

- [4]. [删除指定数量的chat(最多两百个)](#4-删除指定数量的chat最多两百个)

- [5]. [删除指定的chat](#5-删除指定的chat)

- [5]. [与bing对话](#5-与bing对话)

- [6].[ ai画图 (由openai的 dall-e提供的画图)](#6-ai画图-由openai的-dall-e提供的画图)

可以点击目录跳转到相应的内容位置。

## 步骤一:创建 bing client

1. Bing_Client:  
   参数:
    - (1).`cookie`: bing.com的cookie,需要使用浏览器扩展`cookie-editor`
      在bing聊天界面到处cookie成json,然后保存到文件中,然后传入文件路径的str或Path,也可以直接将list[dict]格式的cookie直接传入
    - (2).`proxy`:局域网代理,在本机没有全局的代理或分流时,需要填写,支持http,httpx,socks5

```python
import asyncio

from async_bing_client import Bing_Client

client = Bing_Client(cookie="cookie.json", proxy="socks5://127.0.0.1:7890")

if __name__ == '__main__':
    async def main():
        await client.init()


    asyncio.run(main())
```

## 步骤二: 使用client

### [1].创建新的对话

```python
# 这里创建的新的对话可以直接用于ask_stream
chat = await client.create_chat()

```

### [2]. 获取所有缓存的chat列表

这里面已经包含了所有的最近两百条对话的聊天记录

```python
chat_list = client.chat_list
```

### [3]. 重新加载缓存的chat列表

```python
# 直接重新初始化即可
await client.init()
```

### [4]. 删除指定数量的chat(最多两百个)

参数:

- (1). `count: int = 20`: 要删除的数量
- (2). `del_all: bool = False`: 是否全部删除(最多两百个,因为bing只能加载最近两百个会话)

```python
await client.delete_conversation_by_count()
```

### [5]. 删除指定的chat

参数:

- (1). `conversation_id:str`: 要删除的chat的conversation(chat的key即是conversation_id)

```python
await client.delete_conversation('conversation_id')
```

### [5]. 与bing对话

可以传图图片进行识图,也可以返回Image类型或者md的image格式 包含new bing生成的ai画图

1. 返回不同类的生成器函数:ask_stream_raw

参数:

- (1). `question: str`: 你要询问的内容
- (2). `image: str | Path | bytes = None`: 你要识别的图片(这里的str可以是网络图片的链接,也可以是路径的str)
- (3). `chat: dict = None`: 要发送到的chat 会话窗口,如果没有则会新建一个,并yield一个 NewChat类, 包含这个新的chat
- (4). `conversation_style: ConversationStyle = ConversationStyle.Creative`: 要使用的聊天模式
- (5). `personality=None`: 要加载的预设人格(只需要为正常文本形式即可,会自动混淆), 内容必须在bing的道德允许和法律允许之内,否则会一直返回Apology类型
- (6). `locale:str =guess_locale()`: 要使用的地区(一般不需要自己去指定)

返回的类型:
`AsyncGenerator[NewChat | Apology | Notice | SearchResult | Text | SourceAttribution | SuggestRely | Limit | Response]`

- `Text`: bing的文本回答.可以访问content属性或者使用str方法来获得其中的内容.可以实现加法,将Text相加得到的还是Text
- `Image`: bing的图片回答.其中url属性为图片的链接,base64(Optional)属性为图片的base64内容,name为图片的名称
- `NewChat`: 新生成的chat,可以访问NewChat的chat属性来获得.
- `Apology`: bing的拒绝信息或者错误信息.可以访问content属性或者使用str方法来获得其中的内容
- `Notice`: bing的通知信息.可以访问content属性或者使用str方法来获得其中的内容
- `SearchResult`: bing的搜索结果.content属性存储其list[dict]或者str. 可以使用str方法转化成合适的str格式
- `SourceAttribution`: bing的资源链接回复,display_name属性表示链接展示名称,see_more_url属性为链接,image(optional)
  属性为资源预览图
- `SuggestRely`: bing的建议回复.可以访问content属性或者使用str方法来获得其中的内容
- `Limit`: bing的对话次数上限内容,超过上限后再询问返回的将是Apology.num_user_messages属性为用户当前发送了的消息的数量,max_num_user_messages为用户可以发送的连续消息的上限数量
- `Response`: bing的本次对话的消息汇总,其content储存本次对话产生的所有数据的dict.
  ask_stream函数对ask_stream_raw封装的示例

```python
chat = client.create_chat()
sources = []
suggest_reply = []
images = []
limit = None
async for data in client.ask_stream_raw('question', 'image', chat=chat):
    if isinstance(data, Text):
        yield data.content
    elif isinstance(data, SuggestRely):
        suggest_reply.append(data)
    elif isinstance(data, SourceAttribution):
        sources.append(data)
    elif isinstance(data, Apology):
        yield '\n' + data.content
    elif isinstance(data, Image):
        images.append(data)
    elif isinstance(data, Limit):
        limit = data
    elif isinstance(data, SearchResult) and yield_search:
        yield str(data)
for index, source in enumerate(sources):
    if index == 0:
        yield "\nSee more:  \n"
    yield f"({index + 1}):{source}  \n"
for index, image in enumerate(images):
    if index == 0:
        yield "\nDrew images:  \n"
    yield f"{index + 1}:{image}  \n"
for index, reply in enumerate(suggest_reply):
    if index == 0:
        yield "\nSuggest Replys:  \n"
    yield f"{index + 1}:{reply.content}  \n"
if limit:
    yield f"\n\nLimit:{limit.num_user_messages} of {limit.max_num_user_messages}  "
```

2. 输出纯str的生成器函数 ask_stream:

```python
async for text in client.ask_stream("在吗", chat=chat,
                                    yield_search=False,
                                    conversation_style=ConversationStyle.Balanced):
    print(text, end="")

```

### [6]. ai画图 (由openai的 dall-e提供的画图)

(这个功能在ask_stream中会被自动调用,可以直接用人类语言让bing生成图片)
返回值: `List[Image]`

其中的`Image`: bing的图片回答.其中url属性为图片的链接,name为图片的名称

```python
images = await client.draw("画图的提示词")
```
