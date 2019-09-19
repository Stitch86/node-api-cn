
Node.js 插件可能需要在多个上下文中多次加载的环境。
比如，[Electron] 的运行时会在单个进程中运行多个 Node.js 实例。
每个实例都有它自己的 `require()` 缓存，因此当通过 `require()` 进行加载时，每个实例需要一个原生的插件才能正常运行。
从插件程序的角度来看，这意味着它必须支持多次初始化。

可以使用 `NODE_MODULE_INITIALIZER` 宏来构建上下文感知的插件，它扩展为 Node.js 函数的名字使得 Node.js 能在加载时找到。
可以按如下例子初始化一个插件：

```cpp
using namespace v8;

extern "C" NODE_MODULE_EXPORT void
NODE_MODULE_INITIALIZER(Local<Object> exports,
                        Local<Value> module,
                        Local<Context> context) {
  /* Perform addon initialization steps here. */
}
```

另一个选择是使用 `NODE_MODULE_INIT()` 宏，这也可以用来构建上下文感知的插件。
但和 `NODE_MODULE()` 不一样的是，它可以基于一个给定的函数来构建一个插件，`NODE_MODULE_INIT()` 充当了指定初始化函数体的函数声明。

可以在调用 `NODE_MODULE_INIT()` 之后在函数体内部使用以下三个变量：
* `Local<Object> exports`
* `Local<Value> module`
* `Local<Context> context`

这种构建上下文感知的插件的方式顺带了管理全局静态数据的职责。
由于插件可能被多次加载，甚至会由不同线程加载，插件中任何全局静态数据必须被加以保护，并且不能包含对JavaScript 对象的任何持久性引用。
因为 JavaScript 对象只在一个上下文中有效，从错误的上下文或者另外的线程中访问有可能会导致崩溃。

可以通过执行以下步骤来构造上下文感知的插件以避免全局静态数据：

* 定义一个持有每个插件实例数据的类。这样的类应该包含一个 `v8::Persistent<v8::Object>` 持有 `exports` 对象的弱引用。与该弱引用关联的回调函数将会破坏该类的实例。
* 在插件实例化过程中构造这个类的实例，把`v8::Persistent<v8::Object>` 挂到 `exports` 对象上去。
* 在 `v8::External` 中保存这个类的实例，然后
* 将 `v8::External` 传给 `v8::FunctionTemplate` 构造函数，该函数会创建本地支持的 JavaScript 函数，把 `v8::External` 传递给所有暴露给 JavaScript 的方法。
  `v8::FunctionTemplate` 构造函数的第三个参数接受 `v8::External`。

这确保了每个扩展实例数据到达每个能被 JavaScript 访问的绑定。每个扩展实例数据也必须通过其创建的任何异步回调函数。

下面的例子说明了一个上下文感知的插件的实现：

```cpp
#include <node.h>

using namespace v8;

class AddonData {
 public:
  AddonData(Isolate* isolate, Local<Object> exports):
      call_count(0) {
    // 将次对象的实例挂到 exports 上。
    exports_.Reset(isolate, exports);
    exports_.SetWeak(this, DeleteMe, WeakCallbackType::kParameter);
  }

  ~AddonData() {
    if (!exports_.IsEmpty()) {
      // 重新设置引用以避免数据泄露。
      exports_.ClearWeak();
      exports_.Reset();
    }
  }

  // 每个插件的数据。
  int call_count;

 private:
  // 导出即将被回收时调用的方法。
  static void DeleteMe(const WeakCallbackInfo<AddonData>& info) {
    delete info.GetParameter();
  }

  // 导出对象弱句柄。该类的实例将与其若绑定的 exports 对象一起销毁。
  v8::Persistent<v8::Object> exports_;
};

static void Method(const v8::FunctionCallbackInfo<v8::Value>& info) {
  // 恢复每个插件实例的数据。
  AddonData* data =
      reinterpret_cast<AddonData*>(info.Data().As<External>()->Value());
  data->call_count++;
  info.GetReturnValue().Set((double)data->call_count);
}

// context-aware 初始化
NODE_MODULE_INIT(/* exports, module, context */) {
  Isolate* isolate = context->GetIsolate();

  // 为该扩展实例的AddonData创建一个新的实例
  AddonData* data = new AddonData(isolate, exports);
  // 在 v8::External 中包裹数据，这样我们就可以将它传递给我们暴露的方法。
  Local<External> external = External::New(isolate, data);

  // 把 "Method" 方法暴露给 JavaScript，并确保其接收我们通过把 `external` 作为 FunctionTemplate 构造函数第三个参数时创建的每个插件实例的数据。
  exports->Set(context,
               String::NewFromUtf8(isolate, "method", NewStringType::kNormal)
                  .ToLocalChecked(),
               FunctionTemplate::New(isolate, Method, external)
                  ->GetFunction(context).ToLocalChecked()).FromJust();
}
```

