---
title: |-
    メディアネットワーク演習II
    個別課題4 (REST API)
---

[戻る](./)

## 目標 ##

`GET`，`POST`，`PUT`，`DELETE` の4つのHTTPメソッドを用いてREST APIを設計実装する．実装を通じて挙動を理解する．加えて FastAPI の [APIRouter](https://fastapi.tiangolo.com/reference/apirouter/)を使ってみる．

## 動かしてみよう ##

### サーバの起動 ###

```
./
|-- api/
|   |-- routers/
|   |   `-- message.py
|   |-- schemas/
|   |   `-- message.py
|   `-- main.py
`-- client.html
```

上記のように以下のファイルを配置し，`uvicorn api.main:app --host 0.0.0.0` により `api/main.py` で定義したAPIを有するサーバを起動する(仮想環境を有効化することを忘れずに)．詳細は[第1回](sec01.html)参照．

[sec04.zip](sec04.zip)をダウンロードし，`sec04.zip`があるディレクトリにて `unzip sec04.zip` を実行すれば展開される(今いるディレクトリに展開されるので注意すること; 個別にダウンロードしてもよいし GitHub から取り出したものを利用してもよい)．

なお，GitHub から取り出した場合は既に `data.json` のサンプルが含まれるので，`data.json` を一旦削除するか，`data.json` のファイル名を別の名前に変更すること．

[`api/main.py`](../sec04/api/main.py)

```python
import json

from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import HTMLResponse
from pydantic import ValidationError

import api.schemas.message as message_schema
from api.routers import message


def load(app):
    try:
        with open("data.json", "rt", encoding="utf-8") as f:
            data_dict = json.load(f)
            app.state.messages = message_schema.Messages.model_validate(data_dict)
    except (FileNotFoundError, ValidationError):
        # ファイルが存在しない or ファイルがうまく読めない
        # →Default の Message を作成する
        app.state.messages = message_schema.Messages()

    app.state.counter = 0
    if len(app.state.messages.messages) != 0:
        # id の最大値 + 1 をカウンタにセット
        app.state.counter = max(app.state.messages.messages.keys()) + 1


async def save(app):
    with open("data.json", "wt", encoding="utf-8") as f:
        f.write(app.state.messages.model_dump_json(indent=4))


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

[`api/routers/message.py`](../sec04/api/routers/message.py)


```python
from datetime import datetime
from fastapi import APIRouter
from fastapi import HTTPException
from fastapi import Request

import api.schemas.message as message_schema

router = APIRouter()


@router.get("/messages", response_model=message_schema.Messages)
async def get_messages(request: Request):
    """全ての message を返す"""
    return request.app.state.messages


@router.post("/messages", response_model=message_schema.Message)
async def post_message(request: Request, message: message_schema.MessageBase):
    """message のPOST"""
    m = message_schema.Message(time=datetime.now(),
                               id=request.app.state.counter,
                               **message.model_dump())
    request.app.state.messages.messages[request.app.state.counter] = m
    request.app.state.counter += 1
    return m


@router.get("/messages/{message_id}", response_model=message_schema.Message)
async def get_message(request: Request, message_id: int):
    """個別 message のGET"""
    # 該当 ID の message が存在しない場合は 404 を返す(他の関数でも同様)
    if message_id not in request.app.state.messages.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")
    return request.app.state.messages.messages[message_id]


@router.put("/messages/{message_id}", response_model=message_schema.Message)
async def put_message(request: Request,
                      message_id: int, message: message_schema.MessageBase):
    """message のPUT"""
    if message_id not in request.app.state.messages.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")
    m = message_schema.Message(time=datetime.now(), id=message_id,
                               **message.model_dump())
    request.app.state.messages.messages[message_id] = m
    return m


@router.delete("/messages/{message_id}")
async def delete_message(request: Request, message_id: int):
    """message のDELETE"""
    if message_id not in request.app.state.messages.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")
    del request.app.state.messages.messages[message_id]
    return {"success": True}


@router.get("/messages/{message_id}/important")
async def get_message_important(request: Request, message_id: int):
    """message important flag の GET """
    if message_id not in request.app.state.messages.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")
    # get_message_important
    important = True
    return {"important": important}


@router.put("/messages/{message_id}/important")
async def put_message_important(request: Request, message_id: int):
    """message important flag の PUT (important = True)"""
    if message_id not in request.app.state.messages.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")
    # put_message_important
    return {"success": True}


@router.delete("/messages/{message_id}/important")
async def delete_message_important(request: Request, message_id: int):
    """message important flag の DELETE (important = False)"""
    if message_id not in request.app.state.messages.messages:
        raise HTTPException(status_code=404,
                            detail="Message cannot be found")
    # delete_message_important
    return {"success": True}
```


[`client.html`](../sec04/client.html)


```html
<!doctype html>
<html>
<head>
<title>個別課題4</title>
<meta charset="utf-8" />
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.7.1/jquery.min.js"></script>
<script>

// サーバを指定する場合は server = 'http://192.168.11.9/' などとする
const server = ''

// 編集中の message id
let edit_id = null

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
        console.log(data)
        return api_request_put(url, data).then(function(value) {
            display_replace_message(value)
        })
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
    api_request_get(url).then(function(value) {
        let a = Object.keys(value['messages']).reverse()
        $('#messages ul').empty()
        if (a.length == 0) {
            $('#messages ul').append(`<li class="message">ここにはなにもない</li>`)
        } else {
            for (let i = 0; i < a.length; i++) {
                display_add_message(value['messages'][a[i]])
            }
        }
    })
}

// 各 Message のボタンに callback を登録
function register_callback(message_id) {
    // DELETEボタンが押された時
    $('button#' + message_id + '.delete').off('click').on(
        'click', function() {
            let url = server + '/messages/' + this.id
            api_request_delete(url).then(function() {
                $(this).parent().remove()
            }.bind(this))
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
            })
        }
    )
    // ☆/★ボタンが押された時
    $('button#' + message_id + '.toggle_important').off('click').on(
        'click', function() {
            const toggle_id = this.id
            let url = server + '/messages/' + toggle_id + '/important'
            api_request_get(url).then(function(value) {
                console.log(value['important'])
                if (value['important']) { // ★の時
                    api_request_delete(url).then(function() {
                        message_get(toggle_id)
                    })
                } else { // ☆の時
                    api_request_put(url).then(function() {
                        message_get(toggle_id)
                    })
                }
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
    return `${val['id']} ${val['name']}:<br />${val['message']}<br />${s}<br /><button class='delete' id="${val['id']}">Delete</button><button class="edit" id="${val['id']}">Edit</button><button class="toggle_important" id="${val['id']}">${i}</button>`
}

// message を li として ul に追加
function display_add_message(val) {
    $('#messages ul').append(`<li class="message" id="${val['id']}">${display_item(val)}</li>`)
    register_callback(val['id'])
}

// message を置き換え(li の中身を val で書き換える)
function display_replace_message(val) {
    let s = dateoption.format(Date.parse(val['time']))
    $('#messages ul li.message#' + val['id']).empty()
    $('#messages ul li.message#' + val['id']).append(display_item(val))
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
            }
            if (data['name'].length == 0 || data['message'].length == 0) {
                return
            }
            message_post(data)
        })
    $('button#get_message_clear').on(
        'click', function () {
            $('#messages ul').empty()
        })
    $('button#post_message_clear').on(
        'click', function () {
            edit_id = null
            $('button#post_message').html('POST')
            $('h3#post').html('POST')
            $('#post_message_result').empty()
            $('#post_message_message').val('')
        })
    message_get_all()
})
</script>
<style>
/* スタイルシート お好みで．．． */
input {
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
<li><button id="post_message" type='button'>POST</button></li>
<li><button id="post_message_clear" type='button'>Clear</button></li>
</ul>

<h3>GET</h3>

<ul>
<li><button id="get_message">GET</button></li>
<li><button id="get_message_clear">Clear</button></li>
</ul>

<div id="messages"><ul></ul></div>

</body>
</html>
```


`api/schemas/message.py` については後述．

`api/main.py` が比較的シンプルになっていることが確認できる．ほとんどの機能は `api/routers/message.py` で実現しており，それを `api/main.py` にて `app.include_router(message.router)` により取り込んでいる．`api/routers/message.py` はメッセージを扱う関数のみ記述しており，このようにして機能をファイルに分割し，見通しを良くすることができる(ただし今回の場合はメッセージを扱う以外の機能はないので，効果は限定的である)．

補足(読み飛ばしてもよい):

なお，これまで内部状態を保持していた `app.state` に APIRouter (ここでは `api/routers/message.py`)からアクセスする必要があり，これには各パスオペレーション関数([第1回](sec01.html)参照)に `request` という引数を定義し，この `request.app` (これは `main.py` で生成している FastAPI のインスタンス `app` に対応する)を用いて `state` にアクセスしている．以下参照．

```python
@router.get("/messages", response_model=message_schema.Messages)
async def get_messages(request: Request):
    """全ての message を返す"""
    return request.app.state.messages
```

コードが長くなると(各行に一定確率でバグが混入すると仮定すれば)全体としてバグが含まれる可能性が高くなるため，`request` から所望の値を取り出す [Helper function](https://en.wikipedia.org/wiki/Wrapper_function#Helper_function) (特定の処理を簡単に繰り返し使えるようにまとめた小さな補助的関数)を定義してもよいかもしれない．

### 動作確認1 ###

<http://127.0.0.1:8000/> をブラウザで開くと以下のようなクライアントのページ(画面)が表示される．

![](images/sec04_client.svg)

`Name` および `Message` になにか文字列を入力し，`POST` ボタンを押すと，API を用いてメッセージがサーバに送られ，サーバに保存される．何回か `POST` ボタンによりメッセージを送った後に `GET` ボタンを押してみる．すると以下のような画面となる．

![](images/sec04_client2.svg)

第3回の[追加課題](sec03.html#追加課題)でサーバに複数メッセージを保存できるようにしたが，そのメッセージを全て表示するような挙動をしている．

### 仕様 ###

簡単なメッセージボードのようなものをここでは実現している．<http://127.0.0.1:8000/docs> を開いてどのようなAPIが用意されているかを確認する．

![](images/sec04_APIs.png)

それぞれの意味はここでは以下の通り．

-   `GET /`:\
    Client HTML の取得([第3回](sec03.html)参照)

-   `GET /messages`:\
    全メッセージの取得

-   `POST /messages`:\
    メッセージの追加．複数形のエンドポイント名であるが，メッセージを1つ追加する．複数形のリソース(`/messages`)に対して POST することで，新しいリソースを1つ追加するという設計になっている．

-   `GET /messages/{message_id}`:\
    `{message_id}`(整数を想定)で指定されるメッセージの取得

-   `PUT /messages/{message_id}`:\
    `{message_id}`で指定されるメッセージの更新

-   `DELETE /messages/{message_id}`:\
    `{message_id}`で指定されるメッセージの削除

-   `GET /messages/{message_id}/important`:\
    `{message_id}`で指定されるメッセージの important フラグ(True か False)の取得

    (サンプルではダミー実装としているので注意)

-   `PUT /messages/{message_id}/important`:\
    `{message_id}`で指定されるメッセージの important フラグのセット(Trueにする)

-   `DELETE /messages/{message_id}/important`:\
    `{message_id}`で指定されるメッセージの important フラグの解除(Falseにする)

各メッセージに付属する `DELETE` ボタンを押すとメッセージが削除される．また，各メッセージに付属する `EDIT` ボタンを押すと，上部のメッセージを書く箇所が更新モードになり，メッセージの更新が可能となる(`PUT` ボタンにより送信)．これらの機能は直感的に操作可能なように既に `client.html` にて JavaScript を用いて実装されている．

important フラグはメッセージに付属の星マークで True/False が示されており，以下の表示を想定する．表示については JavaScript にてすでに実装されている．

-   ★: True
-   ☆: False

### 動作確認2 ###

サーバを終了(`C-c`)すると，データがファイルに保存される．保存された内容を見てみよう．`data.json` の中身は以下の様になっている．`"messages"` をキーとした値が辞書となっており，そこには `message_id` をキーとして，各メッセージが保持されている．

```json
{
    "messages": {
        "0": {
            "name": "筒井",
            "message": "テスト",
            "important": false,
            "id": 0,
            "time": "2024-05-12T22:23:35.626917"
        },
        "1": {
            "name": "筒井",
            "message": "テスト1",
            "important": false,
            "id": 1,
            "time": "2024-05-12T22:23:38.503006"
        },
        "2": {
            "name": "筒井",
            "message": "テスト2",
            "important": false,
            "id": 2,
            "time": "2024-05-12T22:23:40.947508"
        },
        "3": {
            "name": "筒井",
            "message": "テスト3",
            "important": false,
            "id": 3,
            "time": "2024-05-12T22:23:43.906841"
        }
    }
}
```

これに対応するスキーマ(schema)は以下の通りとしている．第3回の[追加課題](sec03.html#追加課題)ではリスト型で `messages` を定義したが，今回は所望のメッセージの取得を容易とするため，辞書型としている(実際の実装においては，多くの場合リレーショナル・データベース(RDB)が用いられる)．なお `important` フラグは簡易的な実装となっており，実際の状態は保存されていない(課題で実装する)．

[`api/schemas/message.py`](../sec04/api/schemas/message.py)

```python
from datetime import datetime
from pydantic import BaseModel, Field


class MessageBase(BaseModel):
    name: str | None = Field(None,
                             examples=["System"],
                             description="Message from")
    message: str | None = Field(None,
                                examples=["Default Message"],
                                description="Message body")
    important: bool = Field(False, description="Important or not")


class Message(MessageBase):
    id: int | None = Field(None, description="Message ID")
    time: datetime | None = Field(None, description="Message post time")


class Messages(BaseModel):
    messages: dict[int, Message] = Field(default_factory=dict)
```

### 動作確認3 ###

班の中の1名のPCをサーバとし，班の中で皆でそのサーバにアクセスし，複数人で同時に書き込んだり，削除したり試してみること．

## 課題 ##

メッセージに付属する星マークをクリックする度に

-   ☆ の場合は ★ に，
-   ★ の場合は ☆ に，

変化するようにサーバを実装しなさい．クライアント側は既に実装されているが，サーバの実装が不完全なので，完成させてください．修正が必要なファイルは `api/routers/message.py` である．より具体的には仕様のうち以下の箇所の実装が不完全である．

-   `GET /messages/{message_id}/important`:\
    `{message_id}`で指定されるメッセージの important フラグ(True か False)の取得
    
-   `PUT /messages/{message_id}/important`:\
    `{message_id}`で指定されるメッセージの important フラグのセット(Trueにする)

-   `DELETE /messages/{message_id}/important`:\
    `{message_id}`で指定されるメッセージの important フラグの解除(Falseにする)

なお，`api/routers/message.py` にて important フラグには `request.app.state.messages.messages[message_id].important` でアクセスできる．

## 追加課題 ##

班で話し合ってなにがしか機能を追加しなさい．たとえば以下．

-   important フラグ同様に junk フラグを設ける
-   important フラグがついたもののみ表示するようにする
-   like (いいね)ができるようにする
-   見た目を変える
-   サーバの `data.json` を自分で用意し，たとえば天気予報をメッセージボードで表示するようにする([サンプル](../sec04/data.json)，[ソース](../sec04/tenki.zip))([参考](https://anko.education/webapi/jma))．
    -   サンプルとして提供される `data.json` は，気象庁ホームページの地域時系列予報[^jma]から得られた情報を加工して作成したものである．
-   セキュリティ対策をする([セキュリティについて](sec02.html#セキュリティについて)参照)
-   なにもメッセージが存在しない場合のメッセージ(`ここにはなにもない`)をおしゃれに変更する

Python のコード(バックエンド)の方は修正が簡単であるが，クライアント側の JavaScript (フロントエンド)がいろいろと厄介であると思われる．一般的に，フロントエンド開発についてもフレームワークが利用される([React](https://react.dev/)，[Vue.js](https://ja.vuejs.org/)，[Angular](https://angular.io/)など)．

[^jma]: 気象庁ホームページ, 各種データ・資料 > 気象庁情報カタログ > 気象(予報・予測) > 地域時系列予報, <https://www.data.jma.go.jp/suishin/cgi-bin/catalogue/make_product_page.cgi?id=Jikeiret>.

## Dockerを用いた実行 ##

(本節は興味のある人のみ参考とすること．利用するファイルは GitHub にある)

Docker はアプリケーションの実行環境をコンテナとしてパッケージ化し，どの計算機でも同一の環境で動作させるための仕組みである．OS全体を仮想化するのではなく，必要なライブラリや設定のみを含めた軽量な環境を構築できる．

本演習のようなWebアプリケーションにおいても，Docker を用いることで Python のバージョンやライブラリの違いに依存せずに実行できる．また，将来的にデータベース(RDB)などの他のサービスと組み合わせる際にも，それぞれを独立したコンテナとして起動することで，接続や構成管理を容易に行うことができる．

Docker Desktop をインストールし(2026-04-29現在，管理者権限でインストーラーを実行する必要があり，また，一度ログアウトさせられる)，WSL2 Integration を有効化する．これにより，WSL2 上の Ubuntu 環境から docker コマンドが利用可能となる(ここは各自で設定すること)．

GitHub 上の `sec04/compose.yaml` および `sec04/Dockerfile` がサンプルである．これを用いて Docker 上でコンテナを起動する．`sec04/` に移動して，以下を実行する．

```bash
docker compose up
```

Docker のコンテナ内部で `uvicorn api.main:app --host 0.0.0.0` が実行される．ポートは8000番を利用するように設定してあり，これまでの演習と同様に，WSLおよびWindowsのいずれからもポート8000でアクセス可能となる(対応する設定は `Dockerfile` および `compose.yaml` に記述されている)．なお，`C-c` で終了することができる．


上記をバックグラウンドで動作させたい場合は以下のように `-d` オプションをつける．

```bash
docker compose up -d
```

この場合，停止するには以下を実行する．

```bash
docker compose down
```

Docker Compose には `up` や `down` の他に様々なコマンドがあるので必要に応じて調べること．`Dockerfile` を更新して，コンテナイメージを再度ビルドするには以下を実行する．

```bash
docker compose build
```

ここでは最小限の動作を行うためのサンプルのみを提供している．実際には `compose.yaml` に複数のコンテナを記述し(`uvicorn` によるサーバの他に PostgreSQL などの RDB コンテナなど)，それらをネットワーク的に接続して動作するような環境を構築することになる．興味がある人は各自調べて試してみるとよい．

参考:

-   <https://docs.docker.com/desktop/setup/install/windows-install/>
-   <https://learn.microsoft.com/ja-jp/windows/wsl/tutorials/wsl-containers>

## 解説 ##

### REST ###

REST (Representational State Transfer) は，Web APIの設計に用いられるアーキテクチャスタイル．Roy Fielding 氏が2000年に発表した博士論文の中で提唱された概念[^REST]．本来はアーキテクチャにおける原則と制約の集合を指す(以下)．

-   **ステートレス**なクライアント/サーバプロトコル
    -   各リクエストやメッセージに必要な情報をすべて含み，クライアントやサーバがセッション状態を記憶しておく必要がない
-   リソース(情報)はURIにより**一意に識別**される
-   リソースに対して，HTTPメソッド(`GET`，`POST`，`PUT`，`DELETE`など)により**操作を行う**
    -   データ永続化に求められる[CRUD (Create，Read，Update，Delete)](https://ja.wikipedia.org/wiki/CRUD)と対応づけて理解されることが多い
-   リソースの表現には，関連する他のリソースへのリンクを含めることができる(**ハイパーメディア**)
    -   クライアントはリンクを辿ることで状態遷移を行う(HATEOAS; Hypermedia as the Engine of Application State)

実際に「REST API」と呼ばれているものの多くは，厳密なRESTの定義(特にハイパーメディア制約(HATEOAS))を満たしておらず，単にHTTPを用いたCRUD型のAPIを指している場合も多い．本演習で扱っているAPIも同様に，HTTPを用いたCRUD型の設計となっている．

なお，`PATCH`というHTTPメソッドがあり，`PUT`と使い分けるようなAPI設計も可能である．RESTでは，`PUT`はリソース全体を置き換える(全更新)操作を指し，部分的な更新は`PATCH`を用いるのが一般的である．たとえば，一部(たとえば`Message`だけ)を更新する場合，REST設計の厳密な原則に従うと`PATCH`を使うのがより適切である．ただし，今回の演習では簡単のため`PUT`のみを利用する．

[^REST]: Fielding, Roy Thomas. Architectural Styles and the Design of Network-based Software Architectures. Doctoral dissertation, University of California, Irvine, 2000.

### 本演習設計者からの思惑に関するややこしいコメント ###

本演習ではバックエンド側の動作を理解することを目的として，バックエンド専用のFastAPIを使用している．一方，近年のモダンなWeb開発では，フロントエンドとバックエンドが同一のプロジェクト内で統合されるフルスタックフレームワークも広く使われている．たとえば以下など．

-   [Next.js](https://nextjs.org/): Reactベースのフレームワークで，Server Side Rendering (SSR)やAPIルートを同じプロジェクト内で提供する．

-   [Ruby on Rails](https://rubyonrails.org/): 長年使われてきたオールインワン型のWebアプリケーションフレームワーク(実は GitHub は Ruby on Rails でできている)．MVC (Model-View-Controller) アーキテクチャに基づき，以下から構成される．

    -   Model: RDBMSのテーブルを表すクラスを定義．データベースとやり取りする部分．
    -   View: レンダリング([ERB](https://en.wikipedia.org/wiki/ERuby)など)．ユーザーに見せるデータ(ページ)を作る部分．
    -   Controller: クライアントから来たリクエストを受け取り，何をするかを定義する部分．

    つまり，Controllerがリクエストを受け取り，Modelを用いてデータのやり取りをし，Viewを通じてページを返す，という流れになる．

こうした統合型のフレームワークは，開発の初学者にとっては「すべてが一体化されている」ことで理解しやすいという利点がある．しかし，ネットワークを含めた全体の動作・構造(バックエンドとフロントエンドの責務分離)を意識することが難しくなる．そのため，今回の演習ではバックエンド側に特化し，APIの基本的な仕組みやREST設計の基礎を学ぶことを優先している．

**さらなる補足(SSR)**: CGIが主流だった時代はサーバ側でページ内容を生成し，クライアントに(生成した完全な) HTMLを返すのが一般的だった(「2ちゃんねる」とか)．しかし，[Ajax技術](https://en.wikipedia.org/wiki/Ajax_(programming))やAPIの台頭により，データはJSON形式などでやり取りし，画面描画や状態管理はクライアント側のJavaScript (フロントエンド)が担うようになった．この場合，クライアントの負担が大きくなり，初期表示が遅くなる，SEO (検索エンジン最適化)が難しくなるなどの課題もある．最近のモダンフレームワーク(Next.jsなど)では，サーバ側でページをレンダリングして返すSSR (Server Side Rendering)の仕組みが注目されるようになっている．つまり，レンダリングの責任が「一周回って」サーバ側に戻る傾向が見られるわけである．なお，単に以前のサーバ側でのページ生成に戻ったのではなく，クライアント・サーバ間の責任分担を柔軟に調整できるよう進化している．

[戻る](./)



