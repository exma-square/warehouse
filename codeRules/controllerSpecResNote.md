# Controller Spec Res Note

一切的想了解起因於開 spec

對於回傳的 message 到底該怎麼開 ？  這樣開後端實作會不會有問題 ＝—＝？

... think ... controller ... success ...
... error ... res ... socket ... data ...
... message ... return ... %^&*() ...

才發現真的是 TMDer 完全沒搞清楚訊息 http 的 res 到底是怎麼傳、接到什麼啊 (無數的驚嘆號）


### 來來來，這些東西你真的搞懂了嗎 ？


幾個我們系統中常見的 controller 結尾回傳方式，差異在哪裡你知道嗎 ？

```
res.ok result
res.okWithSocket result
res.serverError error
res.serverErrorWithSocket error
res.serverErrorWithSocket ParserService.errorToJson error
```

- test spec 這裡的 error 哪裡來 ？

- 有沒有發現 error 總是 null 呢 ？

```
request(sails.hooks.http.app)
.post(“/controller/function/")
.send(params).end (error, res)->

  (error == null).should.be.true
  # res.body.should.be ....
  # res.body.should.be ....

```

*如果以上問題能清楚回答，那恭喜你看到這裡就可以了 ～～～～～  YA .*

***

## Controller Return res.XXXX ?

簡單說：帶有 Socket 的都是 TMDer 我們自己定義的，意義與名稱一樣就是跟 Socket 有關係

簡單說的看起來就像廢話一樣，那還是直接來看彼此間的結構關係吧。

```
// sails 提供
res.serverError(err, viewOrRedirect, sendSocket = false) ->

  res.status = 500
  # do sth ...

res.ok(data, viewOrRedirect, sendSocket = false) ->

  res.status = 200
  # do sth ...


res.serverErrorWithSocket(err, viewOrRedirect) ->
  @res.serverError(err, viewOrRedirect, true)

res.okWithSocket(data, viewOrRedirect) ->
  @res.ok(data, viewOrRedirect, true)

```

自己定義的與原生的差別真的只有一個：sendSocket，那傳遞進去的用意是 ？


```
res.ok(data, viewOrRedirect, sendSocket = false) ->
  if sendSocket
    req.socket.emit "error",
      verb: req.options.action
      model: req.options.controller
      data: err
      locals: locals.error
      id: req.param("id") || “”

res.serverError(err, viewOrRedirect, sendSocket = false) ->

  if sails.config.environment isnt "test" && sendSocket
    req.socket.emit "message",
      verb: req.options.action
      model: req.options.controller
      data: data
```

### 小結1：

如果沒有需要在前端 show 出訊息使用 ` res.ok && res.serverError ` 即可，

反之，需要在前端 show 出訊息則使用 ` res.serverErrorWithSocket &&  res.okWithSocket `

* 那 ParserService.errorToJson 呢 ？ *

ParserService.errorToJson 的 code 如下，補充：[Error訊息處理流程](https://github.com/TMDer/warehouse/blob/master/codeRules/Error%20%E8%A8%8A%E6%81%AF%E8%99%95%E7%90%86%E6%B5%81%E7%A8%8B.md)

```
  ###
    前面定義 error.type 預設為 danger，如果自己有設定就覆蓋。
    最重要的是 return 這段 ， 會將 error 轉成只傳 key 與 陣列內容相符的值 。
    例如：
      error = {msg: 'msg', test: 'test'}
      error 傳進去會回傳  {msg: 'msg', type:'danger'}
  ###

  return JSON.parse(JSON.stringify(error, ['message', 'type', 'inner', 'msg', "params"], 2))

```

* 小盲點：使用這個方式，我們要怎麼在錯誤的時候任意的回傳我們要的值呢 T______T ？*


## Controller Spec

前情提要 ` .end (error, res) `，你有發現 error 總是 null 嗎？

```
  request(sails.hooks.http.app)
  .post(“/controller/function/")
  .send(params)
  .expect(200) # !!!!!!!!!!!!!!!  See Hiir
  .end (error, res)->

    (error == null).should.be.true
    # res.body.should.be ....
    # res.body.should.be ....

```
*沒有 expect， `(error == null).should.be.true` 都只是像棒球的快樂槍一樣。*

補充：
1 [sails - Testing controllers](http://sailsjs.org/#/documentation/concepts/Testing?q=testing-controllers)
2 [supertest](https://github.com/tj/supertest)


## 最後總結 .

1. 開立 spec 的時候要確認是否需要於前端顯示 -> 是否使用 Socket

2. spec 最簡單確認的成功與否的方式就是

```
  res.statusCode.should.equal 200 # ok
  res.statusCode.should.equal 500 # error

```