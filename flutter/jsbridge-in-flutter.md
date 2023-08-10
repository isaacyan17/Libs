# JsBridgeInFlutter
为了解决H5在Flutter中的通讯问题，我分别在Vue和React项目中做了JsBridge的实践尝试。下面简单记录下实现方式。


## 原理

JS和Flutter之间的通讯，简单来说就是在双方预先创建好通讯方法。
在Flutter侧向JS发送消息 -> 利用webview的`evaluateJavaScript` 调用预先在H5定义好的方法。当然，首先我们要允许webview对JavaScript的调用(`setJavaScriptEnabled`)。
在Flutter侧接收JS传过来的消息 -> 利用`webview.addJavaScriptChannel` 定义一个接收js数据的方法。 （JS 通过 [your_bridge_name].postMessage() 分发过来。）

在JS侧接收Flutter传过来的消息 -> 定义一个`receiveMessage`方法在`<script/>`标签内。
在JS侧向Flutter发送消息 ->  通过 [your_bridge_name].postMessage() 发送消息。


> 介绍了大概的原理，接下来介绍具体的实现。本次的所有实现都是基于https://github.com/Fitem/native_bridge  进行的改动和处理。
着重说明：我在前端领域的知识积累非常薄弱，下面在前端方面的尝试都是我摸索尝试的思路，可能在很多实现方式上并不合理。它应该是一个思路不是一个方案。并且理应可以优化的更完美。

## BridgeInVue

以Vue3项目为例，在index.html中定义一个接收消息的方法：

```javascript
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <link rel="icon" href="/favicon.ico">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.js"></script>
    <script type="module"   src="./jsBridgeHelper.js"></script>
    <script>
      function receiveMessage(jsonStr) {
           if(jsonStr != undefined && jsonStr != "") {
               let data = JSON.parse(JSON.stringify(jsonStr));
               console.log(`来自App的消息----> ${data.api}`);
               window.jsBridgeHelper.receiveMessage(data);
           }
       }
   </script>
  </body>
</html>

```

其中引入了一个js模块(第11行) ： `jsBridgeHelper.js`。其中主要是定义了一些消息处理的方法，并定义成一个`window`属性变量。
```javascript
import pinia from "./src/main";
import { msgStore } from "./src/store/msgStore";


let callbacks = {};
let callbackId = 1;
let useMsgStore = msgStore(pinia);

class JSBridgeHelper {
    /**
     * 发送消息
     * @param api
     * @param data
     * @returns {Promise<unknown>}
     */
    sendMessage(api, data) {
        return new Promise((resolve, reject) => {
            if (!api || api.length <= 0) {
                reject('api is invalid');
                return;
            }
            let nativeBridge = window.nativeBridge;
            if (nativeBridge === null || nativeBridge === undefined) {
                reject(`
        channel named nativeBridge not found in flutter. please add channel:
        WebView(
          url: ...,
          ...
          javascriptChannels: {
            JavascriptChannel(
              name: nativeBridge,
              onMessageReceived: (message) {
                (instance of WebViewFlutterJavaScriptBridge).parseJavascriptMessage(message);
              },
            ),
          },
        )
        `);
                return;
            }
            // encode message
            const callbackId = this._pushCallback(resolve);
            // 发送消息
            this._postMessage(api, data, callbackId)
            // 增加回调异常容错机制，避免消息丢失导致一直阻塞
            setTimeout(() => {
                const cb = this._popCallback(callbackId)
                if (cb) {
                    cb(null)
                }
            }, 500)
        });
    }

    /**
     * 接受消息处理
     * @param message
     */
    receiveMessage(message) {
        // 新增isResponseFlag为true，避免App收到消息后需要再回复问题
        if (message.isResponseFlag) {
            // 通过callbackId 获取对应Promise
            const cb = this._popCallback(message.callbackId);
            if (cb) { // 有值，则直接调用对应函数
                cb(message.data);
            }
        } else if (message.callbackId) {
            useMsgStore.receiveMessage(message);
            this._postMessage(message.api, null, message.callbackId, true)
            
            // if (message.api === 'isHome') {
            //     this._postMessage(message.api, true, message.callbackId, true)
            // } else {
            //     // 对为支持的api返回默认null
            //     this._postMessage(message.api, null, message.callbackId, true)
            // }
        }
    }

    /**
     * 给App发送消息
     * @param api
     * @param data
     * @param callbackId
     * @param isResponseFlag
     * @private
     */
    _postMessage(api, data, callbackId, isResponseFlag = false) {
        const encoded = JSON.stringify({
            api: api,
            data: data,
            callbackId: callbackId,
            isResponseFlag: isResponseFlag,
        })
        let nativeBridge = window.nativeBridge
        nativeBridge.postMessage(encoded)
    }

    /**
     * 记录一个函数并返回其对应的记录id
     * @param cb 需要记录的函数
     */
    _pushCallback(cb) {
        let id = callbackId++;
        let key = `api_${id}`;
        callbacks[key] = cb;
        return key;
    }

    /**
     * 删除id对应的函数
     * @param {string} id 函数的id
     */
    _popCallback(id) {
        if (callbacks[id]) {
            const cb = callbacks[id];
            callbacks[id] = null;
            return cb;
        }
        return null
    }
}

const jsBridgeHelper = new JSBridgeHelper()
window.jsBridgeHelper = jsBridgeHelper;
```

到这一步，我们在JS侧处理消息的流程告一段落。Flutter传递过来的消息，在 `receiveMessage`接受，并分发到`JsBridgeHelper`中的方法进行处理。那么处理过后的数据如何传递到Vue页面中去呢？ 我这里使用了`Pinia`状态管理库去存储一个全局的状态。
在`main.js`中：

```javascript
import { createPinia } from 'pinia'

const pinia  = createPinia()
const app = createApp(App)
app.use(pinia)
app.mount('#app')

export default pinia;

```

**需要注意的是**：我在`main.js`中暴露出了创建好的pinia变量。是为了能够在`JsBridgeHelper.js`中引入并注册到vue的全局组件中(`app.use`)。保证了pinia在App中的唯一性。
我们可以简单定义一个`pinia` 变量`msgStore`：
```javascript
import { defineStore } from "pinia";

export const msgStore = defineStore("msg", {
    state: () => ({
        id:null,
        api:null,
        data:null,
        isResponseFlag:false
    }),
    actions:{
        receiveMessage(message){
            console.log(message)
            this.id = message.id
            this.api = message.api
            this.data = message.data
            this.isResponseFlag = message.isResponseFlag
        }
    }
})

```

`msgStore`定义了`Message`各个属性的字段。我们可以这样使用它：

```javascript
import { msgStore } from "./src/store/msgStore";
let useMsgStore = msgStore(pinia);
...
useMsgStore.receiveMessage(message);
```

在vue组件中，我们可以根据Pinia的文档灵活定义接受`msgStore`的数据。比如我希望在vue组件中监听`msgStore`的方法产生了变化，我们可以这样：

```javascript
this.useMsgStore.$onAction(
        ({
          name, // action 的名字
          store, // store 实例
          args, // 调用这个 action 的参数
          after, // 在这个 action 执行完毕之后，执行这个函数
          onError, // 在这个 action 抛出异常的时候，执行这个函数
        }) => {
          // 记录开始的时间变量
          const startTime = Date.now()
          // 这将在 `store` 上的操作执行之前触发
          // console.log(`Start "${name}" with params [${args.join(', ')}].`)

          // 如果 action 成功并且完全运行后，after 将触发。
          // 它将等待任何返回的 promise
          after((result) => {
            // console.log(
            //   `Finished "${name}" after ${Date.now() - startTime
            //   }ms.\nResult: ${result}.`
            // );
            console.log(this.useMsgStore.api);
          })

          // 如果 action 抛出或返回 Promise.reject ，onError 将触发
          onError((error) => {
            console.warn(
              `Failed "${name}" after ${Date.now() - startTime}ms.\nError: ${error}.`
            )
          })
        }
      )
```

## BridgeInReact

在React中的实现和vue相似。但是有了一些改动。以下是使用 `npx create-react-app` 为例。
我们同样需要在`index.html`中加入`receiveMessage`的方法。打开React工程目录一看，没有`index.html` !?
翻阅了React文档发现，React将webpack,babel等配置放在了`react-script`中。这里我们采用`ReactDom`创建script标签：

```javascript
 function bridgeEffect() {
    try {
       let msgScript = document.createElement('script');
        msgScript.text = `${receiveMessage}`;
        msgScript.async = true;
        document.body.appendChild(msgScript);
    } catch (e) {
        console.log(e);
    }
}
export  default bridgeEffect;
```

其中的`receiveMessage`:

```javascript
export function receiveMessage(jsonStr) {
    if(jsonStr !== undefined && jsonStr !== "") {
        let data = JSON.parse(JSON.stringify(jsonStr));
        console.log(`来自Native的消息----> ${data.api}`);
        window.jsBridgeHelper.receiveMessage(data);
    }
}
``` 

我们在`index.js`中引用`bridgeEffect`，保证只会初始化一次：

```javascript
import bridgeEffect from './xx/bridgeEffect';
bridgeEffect();
```

接下来我们还需要将`JsBridgeHelper.js`引入到window变量中。在React中我将`JsBridgeHelper`改成了一个立即执行函数：

```javascript
 const JSBridgeHelper = (function() {
    if(window.jsBridgeHelper && window.jsBridgeHelper.inited){
        //已经初始化过了
        return ;
    }

    let callbacks = {};
    let callbackId = 1;
    // const {update} = useMsg();

    /**
     * 发送消息
     * @param api
     * @param data
     * @returns {Promise<unknown>}
     */
   function sendMessage(api, data) {
        return new Promise((resolve, reject) => {
            console.log(`resolve: ${resolve}`);
            if (!api || api.length <= 0) {
                reject('api is invalid');
                return;
            }
            let nativeBridge = window.nativeBridge;
            if (nativeBridge === null || nativeBridge === undefined) {
                reject(`
        channel named nativeBridge not found in flutter. please add channel:
        WebView(
          url: ...,
          ...
          javascriptChannels: {
            JavascriptChannel(
              name: nativeBridge,
              onMessageReceived: (message) {
                (instance of WebViewFlutterJavaScriptBridge).parseJavascriptMessage(message);
              },
            ),
          },
        )
        `);
                return;
            }
            // encode message
            const callbackId = _pushCallback(resolve);
            // 发送消息
            _postMessage(api, data, callbackId)
            // 增加回调异常容错机制，避免消息丢失导致一直阻塞
            setTimeout(() => {
                const cb = _popCallback(callbackId)
                if (cb) {
                    cb(null)
                }
            }, 500)
        });
    }

    /**
     *  注册消息处理，等待native调用
     * @param {方法名} apiName 
     * @param {回调参数} handler 
     */
    function registerHandler(apiName,handler){
        callbacks[apiName] = handler;
    }

    /**
     * 移除消息处理
     * @param {方法名} key 
     */
    function _removeHandler(key){
        delete callbacks[key];
    }

    /**
     * 接受消息处理
     * @param message
     */
    function receiveMessage(message) {
        console.log('--->jsBridgeHelper#receiveMessage '+JSON.stringify(message));

         
        // 新增isResponseFlag为true，避免App收到消息后需要再回复问题
        if (message.isResponseFlag) {
            // 通过callbackId 获取对应Promise
            const cb = _popCallback(message.callbackId);
            if (cb) { // 有值，则直接调用对应函数
                cb(message.data);
            }
        } else if (message.callbackId) {

            _postMessage(message.api, null, message.callbackId, true)

            //zustand
            // update(message.data)
            
            // if (message.api === 'isHome') {
            //     this._postMessage(message.api, true, message.callbackId, true)
            // } else {
            //     // 对为支持的api返回默认null
            //     this._postMessage(message.api, null, message.callbackId, true)
             // }
         }
         if (message.api) {
             var handler = callbacks[message.api];
             if (handler) {
                 try {
                     handler(message.data);
                 } catch (exception) {
                     if (typeof console != 'undefined') {
                         console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
                     }
                     _removeHandler(message.api);
                 }
             }
         }
     }

    /**
     * 给App发送消息
     * @param api
     * @param data
     * @param callbackId
     * @param isResponseFlag
     * @private
     */
    function _postMessage(api, data, callbackId, isResponseFlag = false) {
        const encoded = JSON.stringify({
            api: api,
            data: data,
            callbackId: callbackId,
            isResponseFlag: isResponseFlag,
        })
        let nativeBridge = window.nativeBridge
        nativeBridge.postMessage(encoded)
    }

    /**
     * 记录一个函数并返回其对应的记录id
     * @param cb 需要记录的函数
     */
    function _pushCallback(cb) {
        let id = callbackId++;
        let key = `api_${id}`;
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
            callbacks[id] = null;
            return cb;
        }
        return null
    }

    var JSBridgeHelper = {};
    //保证只初始化一次
    JSBridgeHelper.inited = true;
    JSBridgeHelper.sendMessage = sendMessage;
    JSBridgeHelper.receiveMessage = receiveMessage;
    JSBridgeHelper.registerHandler = registerHandler;
    return JSBridgeHelper;

}())

export default JSBridgeHelper;
```

然后再赋值即可：

```javascript
import JSBridgeHelper from './jsBridgeHelper.js';
window.jsBridgeHelper = JSBridgeHelper;

```

那么又遇到了同样的问题，怎么将收到的Message传入到React组件呢？因为JsBridgeHelper不是一个`React component`，所以我们无法利用状态组件库。我决定采用Java中的回调监听的思路去实现。
在`JsBridgeHelper.js`中定义一个存储`callback`函数的方法：

```javascript
  /**
     *  注册消息处理，等待native调用
     * @param {方法名} apiName 
     * @param {回调参数} handler 
     */
    function registerHandler(apiName,handler){
        callbacks[apiName] = handler;
    }
```

在处理`recieveMessage`的方法中，判断如果存在回调方法则直接调用：

```javascript
         if (message.api) {
             var handler = callbacks[message.api];
             if (handler) {
                 try {
                     handler(message.data);
                 } catch (exception) {
                     if (typeof console != 'undefined') {
                         console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
                     }
                     _removeHandler(message.api);
                 }
             }
         }

```

至此我们处理消息的部分就结束了，在React组件中，如果希望接收某个api的回调，可以使用上述定义的`registerHandler`：

```javascript
    window.jsBridgeHelper.registerHandler('xxx',(e)=>{
        console.log(`${e}`);
    });

```

## BridgeInFlutter

Flutter中的核心方法是在webview 的controller中注册一个Message处理方法：

```dart
//NativeBridgeController.dart 

controller
  ..setJavaScriptMode(JavaScriptMode.unrestricted)
  ..addJavaScriptChannel(
  name,
  onMessageReceived: onMessageReceived,
);

...

Future<void> onMessageReceived(JavaScriptMessage message) async {
  String? messageJson = message.message;
  Message messageItem = messageFromJson(messageJson);
  bool isResponseFlag = messageItem.isResponseFlag ?? false;
  if (isResponseFlag) {
    // 是返回的请求消息，则处理H5回调的值
    NativeBridgeHelper.receiveMessage(messageJson);
  } else {
    // 不是返回的请求消息，处理H5端的请求
    var callMethod = callMethodMap[messageItem.api];
    if (callMethod != null) {
      // 有相应的JS方法，则处理
      var data = await callMethod(messageItem.data);
      messageItem.data = data.toString();
    } else {
      // 若没有对应方法，则返回null，避免低版本未支持Api阻塞H5
      messageItem.data = null;
    }
    // 回调js，类型为回复消息
    messageItem.isResponseFlag = true;
    var json = messageToJson(messageItem);
    runJavaScript("receiveMessage($json)");
  }
}
```

实现`abstract class NativeBridgeController`，并在需要使用webview的widget中，把`webviewcontroller`传入实现类中初始化：

```dart
class WebBridgeController extends NativeBridgeController {
  WebBridgeController(WebViewController controller) : super(controller);

  @override
  Map<String, Function?> get callMethodMap => <String, Function?>{
    "": (data) {
      print('native收到消息：$data');
    },

  };

  @override
  get name => 'nativeBridge';
}


...

  //use:
  bridgeController = WebBridgeController(webController!);
```

## 完整使用流程

实现思路大概已经介绍完了，这里写一下使用流程：

### Flutter调用js

```
 bridgeController?.sendMessage(Message(api: 'xxx',data: 'hello world'));
```

### js调用Flutter

```
window.jsBridgeHelper.sendMessage("xxx", e);
```

### js注册一个回调监听方法(react)

```
window.jsBridgeHelper.registerHandler('xxx',(e)=>{
        console.log(`${e}`);
});


// flutter 调用：
 bridgeController?.sendMessage(Message(api: 'xxx',data: 'hello world'));
```