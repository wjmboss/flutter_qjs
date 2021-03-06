<!--
 * @Description: 
 * @Author: ekibun
 * @Date: 2020-08-08 08:16:50
 * @LastEditors: ekibun
 * @LastEditTime: 2020-10-03 00:44:41
-->
# flutter_qjs

A quickjs engine for flutter.

## Feature

This plugin is a simple js engine for flutter using the `quickjs` project with `dart:ffi`. Plugin currently supports all the platforms except web!

Event loop of `FlutterQjs` should be implemented by calling `FlutterQjs.dispatch()`. 

ES6 module with `import` function is supported and can be managed in dart with `setModuleHandler`.

A global function `channel` is presented to invoke dart function. Data conversion between dart and js are implemented as follow:

| dart | js |
| --- | --- |
| Bool | boolean |
| Int | number |
| Double | number |
| String | string |
| Uint8List | ArrayBuffer |
| List | Array |
| Map | Object |
| JSFunction | function(....args) |
| Future | Promise |

**notice:** `function` can only be sent from js to dart. `Promise` return by `evaluate` will be automatically tracked and return the resolved data.

## Getting Started

### Run on main thread

1. Create a `FlutterQjs` object. Call `dispatch` to dispatch event loop.

```dart
final engine = FlutterQjs();
await engine.dispatch();
```

2. Call `setMethodHandler` to implement js-dart interaction. For example, you can use `Dio` to implement http in js:

```dart
await engine.setMethodHandler((String method, List arg) {
  switch (method) {
    case "http":
      return Dio().get(arg[0]).then((response) => response.data);
    default:
      throw Exception("No such method");
  }
});
```

and in javascript, call `channel` function to get data, make sure the second parameter is a list:

```javascript
channel("http", ["http://example.com/"]);
```

3. Call `setModuleHandler` to resolve the js module.

~~I cannot find a way to convert the sync ffi callback into an async function. So the assets files received by async function `rootBundle.loadString` cannot be used in this version. I will appreciate it if you can provide me a solution to make `ModuleHandler` async.~~

To use async function in module handler, try [Run on isolate thread](#Run-on-isolate-thread)

```dart
await engine.setModuleHandler((String module) {
  if(module == "hello") return "export default (name) => `hello \${name}!`;";
  throw Exception("Module Not found");
});
```

and in javascript, call `import` function to get module:

```javascript
import("hello").then(({default: greet}) => greet("world"));
```

4. Use `evaluate` to run js script:

```dart
try {
  print(await engine.evaluate(code ?? '', "<eval>"));
} catch (e) {
  print(e.toString());
}
```

5. Method `recreate` can destroy quickjs runtime that can be recreated again if you call `evaluate`, `recreat` can be used to reset the module cache. Call `close` to stop `dispatch` when you do not need it.

### Run on isolate thread

1. Create a `IsolateQjs` object, pass a handler to implement js-dart interaction. The handler is used in isolate, so the function must be a top-level function or a static method.

```dart
dynamic methodHandler(String method, List arg) {
  switch (method) {
    case "http":
      return Dio().get(arg[0]).then((response) => response.data);
    default:
      throw Exception("No such method");
  }
}
final engine = IsolateQjs(methodHandler);
// not need engine.dispatch();
```

and in javascript, call `channel` function to get data, make sure the second parameter is a list:

```javascript
channel("http", ["http://example.com/"]);
```

2. Call `setModuleHandler` to resolve the js module. Async function such as `rootBundle.loadString` can be used now to get module. The handler is called in main thread.

```dart
await engine.setModuleHandler((String module) async {
  return await rootBundle.loadString(
      "js/" + module.replaceFirst(new RegExp(r".js$"), "") + ".js");
});
```

and in javascript, call `import` function to get module:

```javascript
import("hello").then(({default: greet}) => greet("world"));
```

3. Same as run on main thread, use `evaluate` to run js script:

```dart
try {
  print(await engine.evaluate(code ?? '', "<eval>"));
} catch (e) {
  print(e.toString());
}
```

4. Method `close` (same as `recreate` in main thread) can destroy quickjs runtime that can be recreated again if you call `evaluate`.

[This example](example/lib/main.dart) contains a complete demonstration on how to use this plugin.

## For Mac & IOS developer

I am new to Xcode and iOS developing, and I cannot find a better way to support both simulators and real devices without combining the binary frameworks. To reduce build size, change the `s.vendored_frameworks` in `ios/flutter_qjs.podspec` to the specific framework.

For simulator, use:

```podspec
s.vendored_frameworks = `build/Debug-iphonesimulator/ffiquickjs.framework`
```

For real device, use:

```podspec
s.vendored_frameworks = `build/Debug-iphoneos/ffiquickjs.framework`
```

Two additional notes:

1. quickjs built with `release` config has bug in resolving `Promise`. Please let me know if you know the solution.

2. `ios/make.sh` limit the build architectures to avoid combine conflicts. Change the `make.sh` to support another architectures.