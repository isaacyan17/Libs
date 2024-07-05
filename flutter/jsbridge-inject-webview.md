# JsBridge 在Flutter webview中的注入式实践

在上文:[[JsBridge在Flutter中的实践](https://isaacyan17.github.io/Libs/#/flutter/jsbridge-in-flutter)]

中, 桥方法实现在前端工程中, 在多前端工程的实践来说不够效率. 我考虑将JsBridge 以注入的方式加载到前端模板里, 即webview加载完HTML模板后, 注入一段JavaScript, 类似于`document.body.appendChild(script);`



## 编译环境

```
Flutter: >=3.17
flutter_inappwebview: ^6.0.0
```



## 实现步骤

我使用的Flutter webview插件是`flutter_inappwebview`, 它提供了一个api: `InAppWebViewController.addUserScript()` 来允许用户将自定义的JavaScript注入到网页文本中 . 

利用这个方法, 我们可以便捷的实现注入JsBridge的目的: 

- 读取存储在Flutter工程中的JsBridge.js

  ```
  Future<String> get jsAsserts async =>
        rootBundle.loadString('assets/jsBridgeHelper.js');
  ```

- webview初始化完成后 , 设置注入时机调用注入方法

```
   var bridgeHelper = UserScript(source: '''
    window.jsBridgeHelper = ${await jsAsserts};
    ''', injectionTime: UserScriptInjectionTime.AT_DOCUMENT_START);
    
    ...
    
    controller?.addUserScript(userScript: bridgeHelper);
```



### 附自定义的JsbridgeHelper

```
(function () {
    if (window.jsBridgeHelper && window.jsBridgeHelper.inited) {
        //已经初始化过了
        return;
    }

    ///回复方法
    let callbacks = {};
    // let callbackId = 1;

    ///注册方法
    let registerCallbackMap = {}

    // const {update} = useMsg();

    /**
     * 发送消息
     * @param api
     * @param data
     * @returns {Promise<unknown>}
     */
    function sendMessage(api, data) {
        return new Promise((resolve, reject) => {
            // console.log(`resolve: ${resolve}`);
            if (!api || api.length <= 0) {
                reject('api is invalid');
                return;
            }
            // 判断环境中是否存在桥接方法
            let nativeBridge = window.flutter_inappwebview;
            if (nativeBridge === null || nativeBridge === undefined) {
                reject(`
        channel named nativeBridge not found in flutter. please add channel in flutter
        guide https://medium.com/flutter-community/flutter-webview-javascript-communication-inappwebview-5-403088610949
        `);
                return;
            }
            // encode message
            const callbackId = _pushCallback(api, resolve);
            // 发送消息
            _postMessage(api, data, callbackId)
            // 增加回调异常容错机制，避免消息丢失导致一直阻塞
           /// 2023年11月13日:  回调超时的时间由前端判定
//            setTimeout(() => {
//                const cb = _popCallback(callbackId)
//                if (cb) {
//                    cb('回调超时')
//                }
//            }, 15000)///回调超时时间改成15秒
        });
    }

    /**
     *  注册消息处理，等待native调用
     * @param {方法名} apiName
     * @param {回调参数} handler
     */
    function registerHandler(apiName, handler) {
        registerCallbackMap[apiName] = handler;
    }

    /**
     * 移除消息处理
     * @param {方法名} key
     */
    function removeHandler(key) {
        delete registerCallbackMap[key];
    }

    /**
     * 接受消息处理
     * @param message
     */
    function receiveMessage(message) {
        console.log('--->jsBridgeHelper#receiveMessage ' + JSON.stringify(message));


        // // 新增responseFlag为true，避免App收到消息后需要再回复问题
        // if (message.responseFlag) {
        //     // 通过callbackId 获取对应Promise
        //     const cb = _popCallback(message.callbackId);
        //     if (cb) { // 有值，则直接调用对应函数
        //         cb(message.data);
        //     }
        // }

        if(message.nativeResponseFlag){
            if(message.callbackId){
                /// 【回调】执行到这里表示原生需要一个返回数据，利用  window.postMessage(message) 分发到页面中，然后在页面中返回数据给原生
                // _postMessage(message.api, '', message.callbackId)
            }
        }

        if (message.api) {
            var handler = registerCallbackMap[message.api];
            ///在本地存储的回调函数中找到对应的handler，说明这是回调方法
            if (handler) {
                try {
                    handler(message.data);
                } catch (exception) {
                    if (typeof console != 'undefined') {
                        console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
                    }
                    removeHandler(message.api);
                }
            }else{
                  const cb = _popCallback(message.api)
                  if(cb){
                   ///表示这是js向native的一个回调方法api， 走到这里表示native向js执行回调
                   cb(message.data);
                  }else{
                      window.postMessage(message);
                  }
            }
        }
        ///检查是否有未使用的callback
        _checkUnusedCallback();
    }

    /**
     * 给App发送消息
     * @param api
     * @param data
     * @param callbackId
     * @param nativeResponseFlag ,true表示原生需要返回数据， false表示原生不需要返回数据
     * @private
     */
    function _postMessage(api, data, callbackId, responseFlag = false) {
        const encoded = JSON.stringify({
            api: api,
            data: data,
            callbackId: callbackId,
            nativeResponseFlag: responseFlag,
        })
        // let nativeBridge = window.nativeBridge
        // nativeBridge.postMessage(encoded)
        window.flutter_inappwebview.callHandler('receiveMessage', encoded).then(result => {
         // -1表示结果不需要回调给业务js
         /// -1是从原生业务侧传递过来的特殊值。因为项目中，null关键字在部分api被要求传递给业务侧。
         /// -1表示未来会有数据返回。但此刻的数据不要被传递。

            if (result!=-1) {
                const cb = _popCallback(callbackId);
                if (cb) { // 有值，直接调用回调函数
                    cb(result);
                }
            }
        });
    }

    /**
     * 记录一个函数并返回其对应的记录id
     * key为调用方法名+id
     * @param cb 需要记录的函数
     */
    function _pushCallback(apiName, cb) {
        let id = new Date().getTime();
        let key = `${apiName}_${id}`;
        callbacks[key] = cb;
        return key;
    }

    /**
     * 删除id对应的函数
     * @param {string} id 函数的id
     */
    function _popCallback(id) {
        if (callbacks[id]) {
            const cb = callbacks[id];
            delete callbacks[id];
            return cb;
        }
        return null
    }

    function _checkUnusedCallback(){
        let cur  = new Date().getTime();
        for(let key in callbacks){
            let t = key.split('_')[1];
            if(cur - t > 1000 * 60 * 60){
                console.log('jsBridgeHelper#_checkUnusedCallback 释放回调函数',key)
                delete callbacks[key]
            }
        }
    }

    var JSBridgeHelper = {};
    //保证只初始化一次
    JSBridgeHelper.inited = true;
    JSBridgeHelper.sendMessage = sendMessage;
    JSBridgeHelper.receiveMessage = receiveMessage;
    JSBridgeHelper.registerHandler = registerHandler;
    return JSBridgeHelper;

}())
```

