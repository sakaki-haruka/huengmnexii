---
title: |-
    メディアネットワーク演習II
    個別課題5 (Query String)
---

[戻る](./)

## 目標 ##

-   Query String を理解して少し本格的につくってみる
-   JavaScript 以外のクライアントを作成する
-   「ぬるぽ」を「ガッ」する
-   これまでを振り返る

## 動かしてみよう ##

### サーバの起動 ###

[第4回](sec04.html)で作ったメッセージボードのアプリを第5回向けに手直ししたので，班内で動作の確認をせよ．

```
./
|-- api/
|   |-- routers/
|   |   `-- message.py
|   |-- schemas/
|   |   |-- message.py
|   |   `-- system.py
|   |-- db.py
|   `-- main.py
`-- client.html
```

上記のように以下のファイルを配置し，`uvicorn api.main:app --host 0.0.0.0` 等により `api/main.py` で定義したAPIを有するサーバを起動する．

[sec05.zip](sec05.zip)をダウンロードし，`sec05.zip`があるディレクトリにて `unzip sec05.zip` を実行すれば展開される(今いるディレクトリに展開されるので注意すること; GitHub から取り出したものを利用してもよい; ファイル数が増えているので個別にダウンロードするのはお勧めしない)．

-   仮想環境を有効化すること([第1回](sec01.html)参照)
-   IPアドレスの調べ方等は[第2回](sec02.html)参照

[`api/main.py`](../sec05/api/main.py)


```python
import json

from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import HTMLResponse
from pydantic import ValidationError

from api.schemas.system import System
from api.routers import message


def load(app):
    try:
        with open("data.json", "rt", encoding="utf-8") as f:
            data_dict = json.load(f)
            app.state.system = System.model_validate(data_dict)
    except (FileNotFoundError, ValidationError):
        # ファイルが存在しない or ファイルがうまく読めない
        # →Default の System を作成する
        app.state.system = System()


async def save(app):
    with open("data.json", "wt", encoding="utf-8") as f:
        f.write(app.state.system.model_dump_json(indent=4))


@asynccontextmanager
async def lifespan(app: FastAPI):
    load(app)
    yield
    await save(app)


app = FastAPI(lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=['null'],
    allow_methods=['*'],
)


@app.get("/", response_class=HTMLResponse)
async def get_client():
    """Return client HTML"""
    data = ''
    with open('client.html', 'rt', encoding='utf-8') as f:
        data = f.read()
    return data


app.include_router(message.router)
```


[`api/db.py`](../sec05/api/db.py)


```python
from fastapi import Request


def get_system(request: Request):
    return request.app.state.system
```


[`api/routers/message.py`](../sec05/api/routers/message.py)


```python
from datetime import datetime
from fastapi import APIRouter
from fastapi import HTTPException
from fastapi import Depends

from api.db import get_system
from api.schemas.system import System, Response
from api.schemas.message import Message, MessageBase

router = APIRouter()


@router.get("/messages", response_model=Response)
async def get_messages(system: System = Depends(get_system),
                       from_id: int | None = 1, to_id: int | None = None,
                       from_time: datetime | None = None,
                       important: bool | None = None,
                       ids_only: bool = False):
    """全ての message を返す"""
    if from_id is None or from_id < 1:
        from_id = 1
    if to_id is None:
        # to_id が指定されない場合は，現在の最大IDまで取得する．
        to_id = system.current_id
    l: list = []
    for i in range(from_id, to_id + 1):
        if i in system.messages:
            if from_time is None or \
               from_time <= system.messages[i].update_time:
                if important is None:
                    l.append(i)
                elif system.messages[i].important == important:
                    l.append(i)

    # ID のリストのみ返す
    if ids_only:
        return Response(
            current_id=system.current_id,
            current_time=datetime.now(),
            ids=l)

    return Response(
        current_id=system.current_id,
        current_time=datetime.now(),
        messages={i: system.messages[i] for i in l})


@router.get("/messages/current_id")
async def get_messages_current_id(system: System = Depends(get_system)):
    return {"current_id": system.current_id}


@router.post("/messages", response_model=Message)
async def post_message(message: MessageBase,
                       system: System = Depends(get_system)):
    """message のPOST"""
    next_id = system.current_id + 1
    now = datetime.now()
    m = Message(
        id=next_id,
        time=now,
        update_time=now,
        **message.model_dump(),
    )
    system.messages[next_id] = m
    system.current_id = next_id
    return m


@router.get("/messages/{message_id}", response_model=Message)
async def get_message(message_id: int,
                      system: System = Depends(get_system), ):
    """個別 message のGET"""
    # 該当 ID の message が存在しない場合は 404 を返す(他の関数でも同様)
    if message_id not in system.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")

    return system.messages[message_id]


@router.put("/messages/{message_id}", response_model=Message)
async def put_message(message_id: int,
                      message: MessageBase,
                      system: System = Depends(get_system)):
    """message のPUT"""
    if message_id not in system.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")

    m = system.messages[message_id]
    m.message = message.message
    m.important = message.important
    m.update_time = datetime.now()
    system.messages[message_id] = m
    return m


@router.delete("/messages/{message_id}")
async def delete_message(message_id: int,
                         system: System = Depends(get_system)):
    """message のDELETE"""
    if message_id not in system.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")

    del system.messages[message_id]
    return {"success": True}


@router.get("/messages/{message_id}/important")
async def get_message_important(message_id: int,
                                system: System = Depends(get_system)):
    """message important flag の GET """
    if message_id not in system.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")

    return {"important": system.messages[message_id].important}


@router.put("/messages/{message_id}/important")
async def put_message_important(message_id: int,
                                system: System = Depends(get_system)):
    """message important flag の PUT (important = True)"""
    if message_id not in system.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")

    m = system.messages[message_id]
    m.update_time = datetime.now()
    m.important = True
    return {"success": True}


@router.delete("/messages/{message_id}/important")
async def delete_message_important(message_id: int,
                                   system: System = Depends(get_system)):
    """message important flag の DELETE (important = False)"""
    if message_id not in system.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")

    m = system.messages[message_id]
    m.update_time = datetime.now()
    m.important = False
    return {"success": True}
```


[`api/schemas/message.py`](../sec05/api/schemas/message.py)


```python
from datetime import datetime
from pydantic import BaseModel, Field


class MessagePost(BaseModel):
    message: str | None = Field(None,
                                examples=["Default Message"],
                                description="Message body")
    important: bool = Field(False, description="Important or not")


class MessageBase(MessagePost):
    name: str | None = Field(None,
                             examples=["System"],
                             description="Message from")


class Message(MessageBase):
    id: int | None = Field(None, description="Message ID")
    time: datetime | None = Field(None, description="Message post time")
    update_time: datetime | None = Field(None,
                                         description="Message update time")
```


[`api/schemas/system.py`](../sec05/api/schemas/system.py)


```python
from datetime import datetime
from pydantic import BaseModel, Field
from .message import Message


class System(BaseModel):
    current_id: int = Field(0, description="Current (latest) ID")
    messages: dict[int, Message] = Field(default_factory=dict)


class Response(System):
    current_time: datetime | None = Field(None,
                                          description="Current server time")
    ids: list[int] = Field(default_factory=list)
```


[`client.html`](../sec05/client.html)


```html
<!doctype html>
<html>
<head>
<title>個別課題5</title>
<meta charset="utf-8" />
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.7.1/jquery.min.js"></script>
<script>

// サーバを指定する場合は server = 'http://192.168.11.9/' などとする
const server = ''

// 更新間隔(msec)
const interval = 5000

// interval 管理用の変数
let interval_id = null

// 編集中の message id
let edit_id = null

// 取得したサーバの時刻
let server_time = null

// filter mode (none, important, not_important)
const filter_mode = {
    none: 'none',
    important: 'important',
    not_important: 'not_important',
}
let filter = filter_mode.none

// 時刻フォーマット
const dateoption = new Intl.DateTimeFormat('ja-JP', {
    hour: 'numeric',
    minute: 'numeric',
    second: 'numeric',
})

// GET
async function api_request_get(url, method = 'GET') {
    try {
        const response = await fetch(url, {
            method: method
        })
        const json = await response.json()
        return json
    } catch (err) {
        console.error("GET失敗:", err)
        return { message: "エラーが発生しました" }
    }
}

// POST
async function api_request_post(url, data, method = 'POST') {
    try {
        const response = await fetch(url, {
            method: method,
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(data),
        })
        const json = await response.json()
        return json
    } catch (err) {
        console.error("POST失敗:", err)
        return { message: "エラーが発生しました" }
    }
}

// PUT (POSTを流用)
async function api_request_put(url, data) {
    return api_request_post(url, data, 'PUT')
}

// DELETE (GETを流用)
async function api_request_delete(url) {
    return api_request_get(url, 'DELETE')
}

// Message の POST (edit_id がある場合は PUT)
async function message_post(data) {
    let url = server + '/messages'
    if (edit_id === null) {
        return api_request_post(url, data)
    } else {
        url = url + '/' + edit_id
        return api_request_put(url, data)
    }
}

// Message の GET (個別 Message)
async function message_get(id) {
    let url = server + '/messages/' + id
    api_request_get(url).then(function(value) {
        display_replace_message(value)
    })
}

// Message の GET (全て)
async function message_get_all() {
    let url = server + '/messages'
    if (server_time !== null) {
        const params = { from_time: server_time }
        const search_params = new URLSearchParams(params)
        url += '?' + search_params.toString()
    }
    api_request_get(url).then(function(value) {
        if (Object.keys(value['messages']).length == 0) {
            if (!$('#messages ul li.message').length) {
                $('#messages ul').prepend(`<li class="message" id="dummy">ここにはなにもない</li>`)
            }
        } else {
            $('#messages ul .message#dummy').remove()
            for(let i in value['messages']) {
                display_add_message(value['messages'][i])
            }
        }
        server_time = value['current_time']
    })
}

// 各 Message のボタンに callback を登録
function register_callback(message_id) {
    // DELETEボタンが押された時
    $('button#' + message_id + '.delete').off('click').on(
        'click', function() {
            let url = server + '/messages/' + this.id
            api_request_delete(url)
            $(this).parent().remove()
        }
    )
    // EDITボタンが押された時
    $('button#' + message_id + '.edit').off('click').on(
        'click', function() {
            edit_id = this.id
            let url = server + '/messages/' + edit_id
            api_request_get(url).then(function(value) {
                $('#post_message_name').val(value['name'])
                $('#post_message_message').val(value['message'])
                $('button#post_message').html('PUT')
                $('h3#post').html('PUT ' + edit_id)
                if (value['important']) {
                    $('#post_message_important').attr('checked', true)
                } else {
                    $('#post_message_important').attr('checked', false)
                }
            })
        }
    )
    // ☆/★ボタンが押された時
    $('button#' + message_id + '.toggle_important').off('click').on(
        'click', function() {
            let url = server + '/messages/' + this.id + '/important'
            api_request_get(url).then(function(value) {
                if (value['important']) { // ★の時
                    return api_request_delete(url)
                } else { // ☆の時
                    return api_request_put(url)
                }
            }).then(function() {
                message_get_all()
            })
        }
    )
}

// message 表示用の HTML 生成(li の中身)
function display_item(val) {
    let i = '☆'
    if(val['important']) {
        i = '★'
    }
    let s = dateoption.format(Date.parse(val['time']))
    let s_update = dateoption.format(Date.parse(val['update_time']))
    return `${val['id']} ${val['name']}:<br />${val['message']}<br />Time: ${s}<br />Update: ${s_update}<br /><button class='delete' id="${val['id']}">Delete</button><button class="edit" id="${val['id']}">Edit</button><button class="toggle_important" id="${val['id']}">${i}</button>`
}

// message を li として ul に追加
function display_add_message(val) {
    if ($('#messages ul li.message#' + val['id']).length == 0) {
        // まだ存在しない場合は li を作成
        $('#messages ul').prepend(`<li class="message" id="${val['id']}"></li>`)
        $('#messages ul li.message#' + val['id']).hide()
    }
    display_replace_message(val)
}

// message の置き換え(li の中身を val で書き換える)
// filter の条件に応じて消したり表示したり
function display_replace_message(val) {
    $('#messages ul li.message#' + val['id']).empty()
    $('#messages ul li.message#' + val['id']).append(display_item(val))
    if (val['important']) {
        $('#messages ul li.message#' + val['id']).append('<div class="important"></div>')
        if (filter == filter_mode.not_important) {
            $('#messages ul li.message#' + val['id']).fadeOut()
        } else {
            $('#messages ul li.message#' + val['id']).hide()
            $('#messages ul li.message#' + val['id']).fadeIn()
        }
    } else {
        $('#messages ul li.message#' + val['id']).append('<div class="not_important"></div>')
        if (filter == filter_mode.important) {
            $('#messages ul li.message#' + val['id']).fadeOut()
        } else {
            $('#messages ul li.message#' + val['id']).hide()
            $('#messages ul li.message#' + val['id']).fadeIn()
        }
    }
    register_callback(val['id'])
}

// HTMLロード後，基本オブジェクトに callback を追加
$(function () {
    $('button#get_message').on(
        'click', function () {
            message_get_all()
        })
    $('button#post_message').on(
        'click', function () {
            let data = {
                name: $('#post_message_name').val().trim(),
                message: $('#post_message_message').val().trim(),
                important: $('#post_message_important').prop('checked')
            }
            if (data['name'].length == 0 || data['message'].length == 0) {
                return
            }
            message_post(data).then(function() {
                message_get_all()
            })
        })
    $('button#get_message_clear').on(
        'click', function () {
            $('#messages ul').empty()
            server_time = null
        })
    $('button#post_message_clear').on(
        'click', function () {
            edit_id = null
            $('button#post_message').html('POST')
            $('h3#post').html('POST')
            $('#post_message_result').empty()
            $('#post_message_message').val('')
        })
    $('button#filter_all').on(
        'click', function () {
            $('#messages ul li.message').show()
            filter = filter_mode.none
        })
    $('button#filter_not_important').on(
        'click', function () {
            $('#messages ul li.message').hide()
            $('#messages ul li.message .not_important').parent().show()
            filter = filter_mode.not_important
        })
    $('button#filter_important').on(
        'click', function () {
            $('#messages ul li.message').hide()
            $('#messages ul li.message .important').parent().show()
            filter = filter_mode.important
        })
    // 一旦 Messages を取得，あとは interval 毎取得
    message_get_all()
    if (interval_id !== null) {
        clearInterval(interval_id)
    }
    interval_id = setInterval(message_get_all, interval)
})
</script>
<style>
/* スタイルシート お好みで．．． */
input[type=text] {
    width: 95%;
}
ul {
    /* https://developer.mozilla.org/ja/docs/Web/CSS/CSS_flexible_box_layout */
    display: flex;

    /* https://developer.mozilla.org/ja/docs/Web/CSS/flex-wrap */
    flex-wrap: wrap;

    list-style: none; /* ・を消す */
}
/* Message の表示スタイル */
li.message {
    width: 120pt;
    background: #fee;
    border-radius: 12% 3%;
    list-style-position: inside;
    border: 1px solid #caa;
    padding: 5px;
}
/* Button の表示スタイル */
button {
    border: 1px solid #a88;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    margin: 1px 1px;
    transition-duration: 0.4s;
    cursor: pointer;
}
button:hover {
    background-color: #644;
    color: #fee;
}
button.toggle_important {
    border: none;
    background: none;
    font-size: 16px;
}
button.toggle_important:hover {
    color: black;
}
</style>
</head>
<body>

<h3 id="post">POST</h3>

<ul>
<li><label for="post_message_name">Name:</label>
<input type="text" id="post_message_name" name="name"></li>
<li><label for="post_message_message">Message:</label>
<input type="text" id="post_message_message" name="message"></li>
</ul>
<ul>
<li><label for="post_message_important">Important:</label>
<input id="post_message_important" type="checkbox"></li>
</ul>

<ul>
<li><button id="post_message" type='button'>POST</button></li>
<li><button id="post_message_clear" type='button'>Clear</button></li>
</ul>

<h3>GET</h3>

<ul>
<li><button id="get_message">GET</button></li>
<li><button id="get_message_clear">Clear</button></li>
</ul>

<h3>Filter</h3>

<ul>
<li><button id="filter_all">ALL</button></li>
<li><button id="filter_important">★</button></li>
<li><button id="filter_not_important">☆</button></li>
</ul>

<div id="messages"><ul></ul></div>

</body>
</html>
```


ブラウザからサーバにアクセス[^access]すると以下のようなクライアントのページ(画面)が表示される．

[^access]: サーバのIPアドレスが 192.168.11.13 であれば `http://192.168.11.13:8000/`をブラウザで開く．

![](images/sec05_client.svg)

`Name` および `Message` になにか文字列を入力し，`POST` ボタンを押すと，API を用いてメッセージがサーバに送られ，かつ自動でクライアント側も更新される．

一通りの機能が実装されているので，班内で動作の確認をせよ．

![](images/sec05_client2.svg)

なお，`client.html` に以下の記述があるとおり，5秒毎にAPIを用いて更新をチェックしている(ただしメッセージの削除に関する更新には対応できていない)．

```javascript
// 更新間隔(msec)
const interval = 5000
// 中略
setInterval(message_get_all, interval);
```

### ブラウザ以外のクライアントでアクセスしてみる(1) ###

`curl` と `jq` コマンドを利用するので，インストールされていない場合はインストールする．以下のコマンドでそれぞれインストールされる(Mac の場合は `Homebrew` でインストールする)．なお，`curl` の使い方については[第1回](sec01.html)の解説で少し触れている．

```
% sudo apt install curl
% sudo apt install jq
```

サーバを立ち上げた状態で，以下のコマンドにより API に `GET` アクセスしてみる[^hit]．`http://127.0.0.1:8000/` の箇所は適宜変更すること．

なお，最初の行の `%` は[プロンプト](https://en.wikipedia.org/wiki/Command-line_interface#Command_prompt)を表しており，それ以降の `curl` からの部分を端末に入力する．

[^hit]: APIを叩く，などという．

```
% curl http://127.0.0.1:8000/messages
{"current_id":4,"messages":{"1":{"name":"筒井","message":"テスト","important":false,"id":1,"time":"2024-05-18T19:54:10.749061","update_time":"2024-05-18T19:54:10.749061"},"2":{"name":"筒井","message":"テスト","important":false,"id":2,"time":"2024-05-18T19:54:11.623206","update_time":"2024-05-18T20:01:09.113812"},"3":{"name":"筒井","message":"テスト","important":true,"id":3,"time":"2024-05-18T19:54:12.544985","update_time":"2024-05-18T20:01:06.254776"},"4":{"name":"筒井","message":"テスト","important":true,"id":4,"time":"2024-05-18T19:54:13.435663","update_time":"2024-05-18T20:01:04.911829"}},"current_time":"2024-05-18T20:13:05.821966","ids":[]}
```

見にくいので `jq` コマンドに結果を整形してもらう．

```json
% curl http://127.0.0.1:8000/messages | jq
{
  "current_id": 4,
  "messages": {
    "1": {
      "name": "筒井",
      "message": "テスト",
      "important": false,
      "id": 1,
      "time": "2024-05-18T19:54:10.749061",
      "update_time": "2024-05-18T19:54:10.749061"
    },
    "2": {
      "name": "筒井",
      "message": "テスト",
      "important": false,
      "id": 2,
      "time": "2024-05-18T19:54:11.623206",
      "update_time": "2024-05-18T20:01:09.113812"
    },
    "3": {
      "name": "筒井",
      "message": "テスト",
      "important": true,
      "id": 3,
      "time": "2024-05-18T19:54:12.544985",
      "update_time": "2024-05-18T20:01:06.254776"
    },
    "4": {
      "name": "筒井",
      "message": "テスト",
      "important": false,
      "id": 4,
      "time": "2024-05-18T19:54:13.435663",
      "update_time": "2024-05-18T20:01:04.911829"
    }
  },
  "current_time": "2024-05-18T20:14:20.485851",
  "ids": []
}
```

同じエンドポイントに対する `GET` によるデータ取得で，少し異なったデータ取得を行いたい場合は Query String を用いる(解説 [Query String と URL (もしくはURI)] 参照)．そのような実装を行っている．

**IDのみ取得**:

```json
% curl 'http://127.0.0.1:8000/messages?ids_only=true' | jq
{
  "current_id": 4,
  "messages": {},
  "current_time": "2024-05-18T20:16:34.402135",
  "ids": [
    1,
    2,
    3,
    4
  ]
}
```

**IDが2から3のもののみ取得**:

```json
% curl 'http://127.0.0.1:8000/messages?from_id=2&to_id=3' | jq | less
{
  "current_id": 4,
  "messages": {
    "2": {
      "name": "筒井",
      "message": "テスト",
      "important": false,
      "id": 2,
      "time": "2024-05-18T19:54:11.623206",
      "update_time": "2024-05-18T20:01:09.113812"
    },
    "3": {
      "name": "筒井",
      "message": "テスト",
      "important": true,
      "id": 3,
      "time": "2024-05-18T19:54:12.544985",
      "update_time": "2024-05-18T20:16:04.573594"
    }
  },
  "current_time": "2024-05-18T20:17:12.629130",
  "ids": []
}
```

**importantフラグが立っているもののみ取得**:

```json
% curl 'http://127.0.0.1:8000/messages?important=true' | jq
{
  "current_id": 4,
  "messages": {
    "3": {
      "name": "筒井",
      "message": "テスト",
      "important": true,
      "id": 3,
      "time": "2024-05-18T19:54:12.544985",
      "update_time": "2024-05-18T20:17:52.258022"
    }
  },
  "current_time": "2024-05-18T20:18:02.562740",
  "ids": []
}
```

**ある時刻以降に更新があったもののみ取得**:

```json
% curl 'http://127.0.0.1:8000/messages?from_time=2024-05-18T20:16' | jq
{
  "current_id": 4,
  "messages": {
    "3": {
      "name": "筒井",
      "message": "テスト",
      "important": true,
      "id": 3,
      "time": "2024-05-18T19:54:12.544985",
      "update_time": "2024-05-18T20:17:52.258022"
    },
    "4": {
      "name": "筒井",
      "message": "テスト",
      "important": false,
      "id": 4,
      "time": "2024-05-18T19:54:13.435663",
      "update_time": "2024-05-18T20:16:05.581446"
    }
  },
  "current_time": "2024-05-18T20:19:50.274739",
  "ids": []
}
```

最新のメッセージのIDを返すエンドポイントを作成してあり，これによりメッセージの投稿があったかどうかを確認することができる．

```json
% curl http://127.0.0.1:8000/messages/current_id | jq
{
  "current_id": 4
}
```

以上は `GET` によるアクセスであるが，以下のように `POST` することもできる．`http://127.0.0.1:8000/messages` に対して `-d` オプションで指定した JSON データを `POST` している．

```
% curl -X POST http://127.0.0.1:8000/messages -d '{ "name": "Bot", "message": "Test" }' -H "Content-Type: application/json"
```

### ブラウザ以外のクライアントでアクセスしてみる(2) ###

上記は `curl` コマンドにより API にアクセスしたが，以下では Python プログラムおよび Shell Script を用いる．

#### Python ####

以下は投稿があった際にその内容を表示する Python プログラムである．前述の `sec05.zip` に含まれている．

[`bot-simple.py`](../sec05/bot-simple.py)

```python
#!/usr/bin/env python3

import json
import time

import requests
import api.schemas.message

BASE_URL = 'http://127.0.0.1:8000'


def post_message(name, message):
    url = f"{BASE_URL}/messages"
    m = api.schemas.message.MessageBase(name=name, message=message)
    requests.post(url, json=m.model_dump())


def get_message(message_id):
    url = f"{BASE_URL}/messages/{message_id}"
    res = requests.get(url)
    res_dict = json.loads(res.text)
    response = api.schemas.message.Message.model_validate(res_dict)
    return response


def print_message(message):
    star = "★" if message.important else ""
    print(f"{message.update_time.strftime('%H:%M:%S')} "
          f"{message.id} "
          f"{message.name}: {message.message}{star}")


def check(server_current_id):
    url = BASE_URL
    url = f"{url}/messages/current_id"
    res = requests.get(url)
    res_dict = json.loads(res.text)
    if server_current_id is not None and \
       res_dict['current_id'] != server_current_id:
        for i in range(server_current_id + 1, res_dict['current_id'] + 1):
            message = get_message(i)
            print_message(message)

    return res_dict['current_id']


def main():
    server_current_id = None
    while True:
        server_current_id = check(server_current_id)
        time.sleep(1)


if __name__ == "__main__":
    main()
```

このプログラムは [`requests`](https://requests.readthedocs.io/) に依存するので `pip` でインストールする．

```bash
% pip install requests
```

`api/` が**ある**ディレクトリに [`bot-simple.py`](../sec05/bot-simple.py) を置いて(**重要**: `api/bot-simple.py` に置くのではない)，以下のように実行する(クラスを用いた実装である[bot.py](../sec05/bot.py)を用いてもよい)．

```
% ./bot-simple.py
20:22:01 筒井: ほげほげ
20:22:04 筒井: ほげほげ
20:22:07 筒井: ほげほげ
```

メッセージが `POST` されると，それを表示する．前述の `http://127.0.0.1:8000/messages/current_id` で取得した値を保存しておき，再度取得した際に値が大きくなっているかで投稿があったかどうかを検知している．なお `client.html` では `http://127.0.0.1:8000/messages?from_time=2024-05-18T20:16` といった `GET` を定期的に行い，更新があったメッセージの表示の更新を行っている(解説 [Query String の処理] 参照)．

<a id="file_mode"></a>

以下のように `Permission denied` でファイルが実行できない場合はファイ
ルのモードをチェックする．

```
% ./bot-simple.py
bash: ./bot-simple.py: Permission denied
```

ファイルのモードのチェックは以下のようにして行う(「ネットワーク構成論」の第8回の[「アクセス制御リスト」のスライド](https://csw.ist.hokudai.ac.jp/network/2024/2024-08.pdf#page=18)参照)．

```
% ls -l bot-simple.py
-rw-r--r-- 1 tsutsui users 1334 May 19 20:05 bot-simple.py
```

`rw-r--r--` の意味は文字列左から

-   `rw-`: ファイルのオーナー(上記の場合 `tsutsui`)の権限
-   `r--`: ファイルのグループ(上記の場合 `users`)の権限
-   `r--`: それ以外のユーザの権限

となっており，`rw-r--r--` は8進数にて `644` などと読む．`r` や `w` などの意味は
以下の通り．つまり executable になっていない．

-   `r`: readable
-   `w`: writable
-   `x`: executable

たとえば以下のようにしてモードを変更する．

```
% chmod 755 bot-simple.py
% ls -l bot-simple.py
-rwxr-xr-x 1 tsutsui users 1334 May 19 20:05 bot-simple.py
```

#### Shell Script ####

上記は投稿があった際にその内容を表示するプログラムであったが，自動で定期的に POST を行う Shell Script を実行してみる．

[`bot.sh`](../sec05/bot.sh)

```bash
#!/bin/sh

base_url=http://127.0.0.1:8000
name=Bot
message=Test

if [ ! "$1" = "" ] ; then
    name=$1
fi
if [ ! "$2" = "" ] ; then
    message=$2
fi
if [ ! "$3" = "" ] ; then
    base_url=$3
fi

while true ; do
    json="{ \"name\": \"$name\",
            \"message\": \"$message\",
            \"important\": false }"
    curl -X POST "$base_url/messages" \
         -H "accept: application/json" \
         -H "Content-Type: application/json" \
         -d "$json"
    echo
    sleep 3
done
```

なお，前述の `sec05.zip` に含まれている．

```
% ./bot.sh
```

あるいは

```
% ./bot.sh 名無しさん ぬるぽ http://192.168.11.13:8000
```

とすれば Name と Message，サーバを指定することができる(もちろんファイルを直接書き換えてもよい)．単に `curl` の実行を定期的に行っているだけである．コマンドを用いて簡単に処理可能なものは Shell Script を用いて繰り返したり**自動化**することが多い．

`Permission denied` となり実行できない場合は，ファイルのモードを確認すること．

なお，最初の行の `#!/bin/sh` や `#!/usr/bin/env python3` は[シバン](https://ja.wikipedia.org/wiki/%E3%82%B7%E3%83%90%E3%83%B3_(Unix))と呼ばれ，重要なので，消さないこと．

## 課題 ##

`bot-simple.py` を改造して，特定の文字列(ぬるぽ)のメッセージが投稿された場合に，特定の文字列(ガッ)を投稿する Bot クライアントを作成しなさい．

メッセージの `POST` を行うための関数は `post_message` として用意されている．`requests.post()` の `json=` 引数を利用することで，JSON 形式のデータを送信している．

## 追加課題 ##

投稿されたメッセージの内容を確認して，それに応じてなにかを投稿する**クライアント**を作成しなさい．たとえば以下など．

-   `1 + 1` などの計算可能な式が投稿された場合に，その答えを返す
-   数値が投稿された場合にそれが素数か否かを返す

演習IIのネットワーク構成論の回で習った正規表現を使うなどが考えられる．

複数人が同時にこのような Bot を動作させると投稿が際限なく増える可能性があるので注意すること．Botによる投稿か否かを判断して(それとわかるように先頭の文字列を工夫するなど)，Bot に対する応答は行わないなどの対処をしたほうがよい．が，しなくてもよい．

なお，今回の課題では，FastAPI の外部(クライアント)で各種処理を実現することを考えているが，FastAPI 側でアクセスがあった際になにがしか処理するような方式も考えられる(こちらが一般的)．

## 解説 ##

### Query String と URL (もしくはURI) ###

`POST` や `PUT` では HTTP リクエストにてヘッダに続くボディにデータを含めることができる．しかし `GET` や `DELETE` では一般的にはボディにデータを含めない．とはいえ `GET` や `DELETE` の際にもなにがしかのデータ(文字列)をサーバに送りたいという要求がある．その際に Query String が利用される．つまり URL に含めてしまうというものである(他にヘッダにデータを含めるやり方もあり，[Cookie](https://ja.wikipedia.org/wiki/HTTP_cookie) を受け取ったりサーバに送ったりなどはこちらが用いられる)．

ELMS の Google アカウントを利用する際に大学のSSOのページが表示されて，そこでログインすると Google のサービスのページが表示されたりするが，その際の認証情報は Query String や HTTP リクエストのヘッダ情報が利用されて，うまく引き渡される[^token]．

(実際の運用では，セキュリティ上の理由から認証情報は主に HTTP ヘッダや Cookie で扱われ，Query String に含める方法は一般的ではない)

[^token]: [RFC 6750 The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750)など参照．

さて，URLは [RFC 3986 Uniform Resource Identifier (URI): Generic Syntax](https://datatracker.ietf.org/doc/html/rfc3986) にて定義されている．以下参照．

-   [3. Syntax Components](https://datatracker.ietf.org/doc/html/rfc3986#section-3)
-   [3.2.  Authority](https://datatracker.ietf.org/doc/html/rfc3986#section-3.2)

以下そこからの抜粋である．

```
   The following are two example URIs and their component parts:

         foo://example.com:8042/over/there?name=ferret#nose
         \_/   \______________/\_________/ \_________/ \__/
          |           |            |            |        |
       scheme     authority       path        query   fragment
          |   _____________________|__
         / \ /                        \
         urn:example:animal:ferret:nose
```

`http://127.0.0.1:8000/messages?from_time=2024-05-18T20:16` の場合，

-   scheme: `http`
-   authority: `127.0.0.1:8000`
-   path: `/messages`
-   query: `from_time=2024-05-18T20:16`

となっている．query は path とともにサーバに送られ，サーバにて解釈される．fragment はサーバに送られず，取得したデータの位置を示すために利用される．

query は `key=value` を複数含めることができ，一般的には
`key1=value1&key2=value2&key3=value3` のように `&` で連結する[^separator]．

[^separator]: `;` で連結する方式も存在するが，現在は `&` での連結(分割)が推奨されている様子．

### Query String の処理 ###

Query String を処理するため，かつ現実的な実装[^practical]になるように `get_messages` 関数を大幅に修正した．以下に該当箇所を示す．

[^practical]: 毎回全てのデータを `GET` しない．

`GET` 用のパスオペレーション関数で，以下のように引数を定義すると，それが Query String として利用できるようになる．たとえば Query String `from_id=10` を付けて API にアクセスすると，`10` が `from_id` に代入される．

なお，`:` から `=` の手前までは，変数の型のヒントを与えており，どのような値が想定されるかが API として定義される．<http://localhost:8000/docs> などで確認するとよい．

```python
@router.get("/messages", response_model=Response)
async def get_messages(system: System = Depends(get_system),
                       from_id: int | None = 1, to_id: int | None = None,
                       from_time: datetime | None = None,
                       important: bool | None = None,
                       ids_only: bool = False):
    """全ての message を返す"""
    if from_id is None or from_id < 1:
        from_id = 1
    if to_id is None:
        # to_id が指定されない場合は，現在の最大IDまで取得する．
        to_id = system.current_id
    l: list = []
    for i in range(from_id, to_id + 1):
        if i in system.messages:
            if from_time is None or \
               from_time <= system.messages[i].update_time:
                if important is None:
                    l.append(i)
                elif system.messages[i].important == important:
                    l.append(i)

    # ID のリストのみ返す
    if ids_only:
        return Response(
            current_id=system.current_id,
            current_time=datetime.now(),
            ids=l)

    return Response(
        current_id=system.current_id,
        current_time=datetime.now(),
        messages={i: system.messages[i] for i in l})
```

### Schema の変更 ###

上述のとおり，様々な `/messages` への `GET` の仕方を許容することにしたので，これまで `Messages` 型を返していたところを `Response` 型を定義してそれを返すことにした．`ids_only == True` であれば `ids` で ID のリストを返し，`ids_only == False` であればこれまでどおり `messages` でメッセージのリストを返す．加えてサーバにアクセスした時刻を返すために `current_time` というフィールドを用意している(これにより，ある時刻以降の更新箇所のみ取得できるような実装が可能となる)．

また，`messages.messages` とか書くのがややこしいので，これまでの `Messages` は `System` という型に名前を変えた．これらに関連して，`Response` 型および `System` 型を `api/schemas/system.py` に分離している．

[`api/schemas/system.py`](../sec05/api/schemas/system.py)


```python
from datetime import datetime
from pydantic import BaseModel, Field
from .message import Message


class System(BaseModel):
    current_id: int = Field(0, description="Current (latest) ID")
    messages: dict[int, Message] = Field(default_factory=dict)


class Response(System):
    current_time: datetime | None = Field(None,
                                          description="Current server time")
    ids: list[int] = Field(default_factory=list)
```


### Depends ###

前回までは以下のように，各パスオペレーション関数に `request` という引数を定義し，この `request.app` を用いて `state` にアクセスしていた．

```python
@router.get("/messages", response_model=message_schema.Messages)
async def get_messages(request: Request):
    """全ての message を返す"""
    return request.app.state.messages
```

`request` から `request.app.state.messages` (この回の場合は `request.app.state.system` になる)を取り出す処理を何度も書くことになるので，この回のコードでは `Depends` を用いて簡潔に記述している．たとえば以下など．

```python
@router.get("/messages/current_id")
async def get_messages_current_id(system: System = Depends(get_system)):
    return {"current_id": system.current_id}
```

FastAPI は強力かつ直感的な依存性注入(DI: Dependency Injection)システム[^di]を持っており，その機能を利用し変数 `system` を取得している．詳しくは <https://fastapi.tiangolo.com/ja/tutorial/dependencies/> を参照．

`get_system` は `Request` から `system` を取り出す Helper function であり，リレーショナル・データベース(RDB)を利用する際にこの箇所を変更すれば良いように `api/db.py` というファイルに分離している．

[^di]: 依存性注入(DI: Dependency Injection): デザインパターンの1つ．

## これまでの内容 ##

これまでの内容は以下の通り．

-   第0回: 環境構築(WSLなど)．`FastAPI`動作確認．`GET`まで．
-   第1回: `POST`メソッドでサーバにデータを送る．JavaScriptを用いたクライアント作成．
-   第2回: ネットワークの設定(WSLとポートフォワーディング)．型あるいは構造，スキーマについて考えた．
-   第3回: データの保存(シリアライズ)と排他制御(ロック機構)．
-   第4回: `GET`，`POST`，`PUT`，`DELETE` 勢揃い(REST API設計実装)．FastAPI の [APIRouter](https://fastapi.tiangolo.com/reference/apirouter/)利用．
-   第5回: Query String．更新された箇所のみ取得する方法を学んだ．

なお，この演習では以下を取り扱っていない．このことを念頭に自由製作に取り組むこと．

### 認証/認可 ###

-   ユーザの区別がつきません．
-   セキュリティもなにもありません．
-   対応しようとするとかなりの追加知識が必要になります．

認証/認可版のサンプルを [sec06.zip](sec06.zip) に用意している(あるいは GitHub から取り出すこと)．

-   <https://fastapi.tiangolo.com/ja/tutorial/security/> に基づくコードとしている．
-   このサンプルではログイン済みユーザかどうかのみを確認しており，各メッセージの作成者かどうかは確認していない．認証は行っているが，厳密な意味での認可制御は行っていない．
-   追加で `pip` により install すべきモジュールがある．`pip install -r requirements.txt` などで install する．
-   セキュリティの観点から以下に注意すること．
    -   `SECRET_KEY` をソースコード中に直接記述しているが，実運用では環境変数などから読み込むべき．
    -   取得したアクセストークンをブラウザの `localStorage` に保存している．この方法では JavaScript からトークンを参照できるため，XSS (Cross-Site Scripting)などにより不正なスクリプトが実行された場合，トークンが読み取られる可能性がある．実運用では，HTTPS の利用，XSS 対策，トークンの有効期限管理，Cookie の `HttpOnly` 属性の利用など，より慎重な設計が必要となる．
-   現状ではユーザ管理のエンドポイントが存在しないので，ユーザを削除する場合はサーバを停止のうえ直接 `data.json` を編集すること．

sec05 と sec06 の差分は以下のように diff コマンドを利用すると確認できる．

```bash
% diff -uwr sec05/ sec06/
```

### リレーショナル・データベース(RDB) ###

-   規模が大きくなると耐えられません．
-   FastAPI は [Object-relational mapping (ORM)](https://ja.wikipedia.org/wiki/%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E9%96%A2%E4%BF%82%E3%83%9E%E3%83%83%E3%83%94%E3%83%B3%E3%82%B0)に対応可能ですが，理解には追加知識がかなり必要になります．
    -   [SQLite](https://www.sqlite.org/)を使う程度ならまだ可能性あります．
    -   Docker を利用すると RDB 環境を容易に構築可能です．
-   なお，データベースはそれ単体で授業科目になったり，大学院入試の科目(北海道大学にはたぶんない)になったりする内容です．[資格試験](https://www.ipa.go.jp/shiken/kubun/db.html)もある．

リレーショナル・データベース(RDB)対応のサンプルを [sec07.zip](sec07.zip) に用意している(あるいは GitHub から取り出すこと)．Docker の利用を想定している．

sec06 と sec07 の差分は以下のように diff コマンドを利用すると確認できる．

```bash
% diff -uwr sec06/ sec07/
```

### フロントエンド ###

-   JavaScriptのサンプルは渡してありますが難解です．
-   (素の) JavaScript は**なんとなく**動くので逆にたちが悪くて，デバッグでかなりの時間を浪費する可能性があります．`console.log("debug")` などを JavaScript に含めると JavaScript Console (ブラウザで Control+Shift+J で開く)に表示されるので，それを使ってデバッグできます．


[戻る](./)
