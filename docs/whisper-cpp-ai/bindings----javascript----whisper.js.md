# `whisper.cpp\bindings\javascript\whisper.js`

```cpp
# 定义一个闭包函数，用于创建 whisper 对象
var whisper_factory = (() => {
  # 获取当前脚本的路径
  var _scriptDir = typeof document !== 'undefined' && document.currentScript ? document.currentScript.src : undefined;
  # 如果存在 __filename，则将其赋值给 _scriptDir
  if (typeof __filename !== 'undefined') _scriptDir = _scriptDir || __filename;
  # 返回一个函数，接受一个模块参数
  return (
    function(moduleArg = {}) {

    }

  );
})();
# 如果 exports 是一个对象且 module 是一个对象，则将 whisper_factory 导出
if (typeof exports === 'object' && typeof module === 'object')
  module.exports = whisper_factory;
# 如果 define 是一个函数且 define['amd'] 存在，则使用 whisper_factory
else if (typeof define === 'function' && define['amd'])
  define([], () => whisper_factory);
```