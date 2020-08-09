<!--

author:   André Dietrich
email:    andre.dietrich@ovgu.de
version:  0.0.2
language: en
narrator: US English Male

script:  https://pyodide-cdn2.iodide.io/v0.15.0/full/pyodide.js

@onload
window.languagePluginUrl = 'https://pyodide-cdn2.iodide.io/v0.15.0/full/'

window.pyodide_ready = false;

languagePluginLoader.then(() => {
  console.log("pyodide is ready")
  if (window.py_packages) {
    pyodide.loadPackage(window.py_packages).then(() => {
      console.log("all packages loaded")
      window.pyodide_ready = true;
    });
  }
  else {
    window.pyodide_ready = true;
  }
});
var module = {};

window.load_packages = function (list) {
  window.py_packages = list;
}

@end

@Pyodide.eval
<script>
if(window.pyodide_ready) {
  pyodide.globals.print = (...e) => { e = e.slice(0,-1); console.log(...e) };
  pyodide.runPython(`@input`);
}
else {
  console.warn("Please wait, Pyodide is not ready yet...");
  "LIA: stop";
}
</script>

@end

-->

# Pyodide - Template

                                   --{{0}}--
A template for executing Python code in [LiaScript](https://LiaScript.github.io)
based on the [Pyodide](https://github.com/iodide-project/pyodide) webassembly
port to JavaScript. This port tries to make Python scientific programming
accessible within the browser, see the [Iodide project](https://iodide.io) for
more information.

__Try it on LiaScript:__

https://liascript.github.io/course/?https://github.com/LiaTemplates/pyodide

__See the project on Github:__

https://github.com/LiaTemplates/pyodide

                                   --{{1}}--
There are three ways to use this template. The easiest way is to use the
`import` statement and the url of the raw text-file of the master branch or any
other branch or version. But you can also copy the required functionionality
directly into the header of your Markdown document, see therefor the
[Implementation](#3). And of course, you could also clone this project and
change it, as you wish.

                                     {{1}}
1. Load the macros via

   `import: https://github.com/LiaTemplates/Pyodide`

2. Copy the definitions into your Project

3. Clone this repository on GitHub

## `@Pyodide.eval`

                                   --{{0}}--
Simply attach the macro `@Pyodide.eval` to the end of your code-block to make
your Python code executable.

```python
import sys

for i in range(5):
	print("Hello", 'World #', i)

sys.version
```
@Pyodide.eval


## Loading Libraries

                                   --{{0}}--
Only the Python standard library and `six` are available at runtime, if you want
to make further libraries accessible in your scripts, then you have to load them
at startup with the `@onload` macro. Simply call the function `load_packages`
with an array of all required Python libraries.


``` markdown
<!--
author:  ...
email:   ...

import:  https://github.com/LiaTemplates/Pyodide

@onload: load_packages(["matplotlib", "numpy"]);
-->
...
```

> __Note:__ loading large packages such as `scipy` may take some time, since
>           they might require to download many MB of precompiled packages.

## Implementation

                                   --{{0}}--
This macro implementation only adds a simple script-tag that pushes the code of
your snippet directly to Pyodide. The `@onload` macro is required to instantiate
Pyodide and load the required libraries, which might require some time, since
the loaded packages might be quite large.


```js
script:  https://pyodide-cdn2.iodide.io/v0.15.0/full/pyodide.js

@onload
window.languagePluginUrl = 'https://pyodide-cdn2.iodide.io/v0.15.0/full/'

window.pyodide_ready = false;

languagePluginLoader.then(() => {
  console.log("pyodide is ready")
  if (window.py_packages) {
    pyodide.loadPackage(window.py_packages).then(() => {
      console.log("all packages loaded")
      window.pyodide_ready = true;
    });
  }
  else {
    window.pyodide_ready = true;
  }
});
var module = {};

window.load_packages = function (list) {
  window.py_packages = list;
}

@end

@Pyodide.eval
<script>
if(window.pyodide_ready) {
  pyodide.globals.print = (...e) => { e = e.slice(0,-1); console.log(...e) };
  pyodide.runPython(`@input`);
}
else {
  console.warn("Please wait, Pyodide is not ready yet...");
  "LIA: stop";
}
</script>

@end
```

                                   --{{1}}--
If you want to minimize loading effort in your LiaScript project, you can also
copy this code and paste it into your main comment header, see the code in the
raw file of this document.


                                     {{1}}
https://raw.githubusercontent.com/LiaTemplates/pyodide/master/README.md
