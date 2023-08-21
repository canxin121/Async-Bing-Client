Here is the English translation of the markdown file:

# Usage Guide

## Table of Contents

[Step 1: Create bing client](#step-1-create-bing-client)

[Step 2: Use client](#step-2-use-client)

- [1]. [Create new conversation](#1-create-new-conversation)

- [2]. [Get all cached chat lists](#2-get-all-cached-chat-lists)

- [3]. [Reload cached chat lists](#3-reload-cached-chat-lists)

- [4]. [Delete specified number of chats (up to 200)](#4-delete-specified-number-of-chats-up-to-200)

- [5]. [Delete specified chat](#5-delete-specified-chat)

- [5]. [Chat with bing](#5-chat-with-bing)

- [6]. [AI drawing (provided by openai's dall-e)](#6-ai-drawing-provided-by-openais-dall-e)

You can click the table of contents to jump to the corresponding content.

## Step 1: Create bing client

1. Bing_Client:
   Parameters:
    - (1).`cookie`: bing.com cookie, need to use browser extension `cookie-editor`
      Export cookie to json in bing chat interface, then pass file path of str or Path, can also directly pass
      list[dict] formatted cookie
    - (2).`proxy`: LAN proxy, need to fill in when there is no global proxy or split tunneling on local machine, support
      http, httpx, socks5

```python
import asyncio

from async_bing_client import Bing_Client

client = Bing_Client(cookie="cookie.json", proxy="socks5://127.0.0.1:7890")

if __name__ == '__main__':
    async def main():
        await client.init()


    asyncio.run(main())
```

## Step 2: Use client

### [1]. Create new conversation

```python 
# The new conversation created here can be directly used for ask_stream
chat = await client.create_chat()

```

### [2]. Get all cached chat lists

Contains chat history of all latest 200 conversations

```python
chat_list = client.chat_list
```

### [3]. Reload cached chat lists

```python
# Just reinitialize 
await client.init()
```

### [4]. Delete specified number of chats (up to 200)

Parameters:

- (1). `count: int = 20`: Number to delete
- (2). `del_all: bool = False`: Whether to delete all (up to 200, because bing can only load latest 200 conversations)

```python
await client.delete_conversation_by_count()
```

### [5]. Delete specified chat

Parameters:

- (1). `conversation_id:str`: conversation_id of chat to delete (chat key is conversation_id)

```python
await client.delete_conversation('conversation_id')
```

### [5]. Chat with bing

Can pass pictures for image recognition, can also return Image type or md image format including new ai drawings
generated by bing

1. Return different generator functions: ask_stream_raw

Parameters:

- (1). `question: str`: Your question
- (2). `image: str | Path | bytes = None`: Image you want to recognize (str here can be link of web image, can also be
  str of path)
- (3). `chat: dict = None`: Chat window to send to, will create new one if not passed, and yield a NewChat class
  containing the new chat
- (4). `conversation_style: ConversationStyle = ConversationStyle.Creative`: Conversation style to use
- (5). `personality=None`: Preset personality to load (just needs to be normal text, will auto obfuscate), content must
  be within bing's moral and legal limits, otherwise will keep returning Apology
- (6). `locale:str =guess_locale()`: Locale to use (usually don't need to specify yourself)

Return types:
`AsyncGenerator[NewChat | Apology | Notice | SearchResult | Text | SourceAttribution | SuggestRely | Limit | Response]`

- `Text`: bing's text reply. Can access content attribute or use str method to get content. Addition of Text will still
  return Text
- `Image`: bing's image reply. url attribute is image link, base64 (Optional) attribute is image base64 content, name is
  image name
- `NewChat`: Newly generated chat, can access chat attribute of NewChat to obtain.
- `Apology`: bing's refusal or error message. Can access content attribute or use str method to obtain content
- `Notice`: bing's notification message. Can access content attribute or use str method to obtain content
- `SearchResult`: bing's search results. content attribute stores list[dict] or str. Can use str method to convert to
  suitable str format
- `SourceAttribution`: bing's resource link reply, display_name attribute is displayed link name, see_more_url attribute
  is link, image (optional) attribute is resource preview image
- `SuggestRely`: bing's suggested reply. Can access content attribute or use str method to obtain content
- `Limit`: bing's conversation limit content, will return Apology after exceeding limit. num_user_messages attribute is
  number of messages user has sent, max_num_user_messages is max number of consecutive messages user can send
- `Response`: Summary of bing's messages in this conversation, content stores dict of all data generated in this
  conversation.
  ask_stream function wraps ask_stream_raw example

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

2. Generator function outputting pure str ask_stream:

```python
async for text in client.ask_stream("hello", chat=chat,
                                    yield_search=False,
                                    conversation_style=ConversationStyle.Balanced):
    print(text, end="")
```

### [6]. AI drawing (provided by openai's dall-e)

(This function will be called automatically in ask_stream, can directly use human language to let bing generate images)
Return value: `List[Image]`

`Image`: bing's image reply. url attribute is image link, name is image name

```python
images = await client.draw("drawing prompt")
```