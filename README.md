<!--

author:   André Dietrich
email:    LiaScript@web.de
version:  0.1.4
language: en
narrator: US English Male


logo:     logo.jpg

comment:  Use the real Python in your LiaScript courses, by loading this
          template. For more information and to see, which Python-modules are
          accessible visit the [pyodide-website](https://alpha.iodide.io).

script:   https://pyodide-cdn2.iodide.io/v0.15.0/full/pyodide.js

@onload
window.languagePluginUrl = 'https://pyodide-cdn2.iodide.io/v0.15.0/full/'

window.pyodide_ready = false;

window.pyodide_modules = new Set()

// window.py_packages = ["matplotlib", "numpy"]

window.loadModules = function() {
  languagePluginLoader.then(() => {
    console.log("pyodide is ready")
    if (window.py_packages) {

      for( let i = 0; i < window.py_packages.length; i++ ) {
        window.pyodide_modules.add(window.py_packages[i])
      }

      pyodide.loadPackage(window.py_packages).then(() => {
        console.log("all packages loaded")
        window.pyodide_ready = true;
      });
    }
    else {
      window.pyodide_ready = true;
    }
  })
}

window.loadModules()

@end


@Pyodide.eval: @Pyodide.eval_(@uid)

@Pyodide.eval_
<script>

function initPlot() {
try {

pyodide.runPython(`
import io, base64

try:
  img_str_
except NameError:
  img_str_ = {}

def plot(fig, id="plot-@0"):
  buf = io.BytesIO()
  fig.savefig(buf, format='png')
  buf.seek(0)
  img_str_[id] = "data:image/png;base64," + base64.b64encode(buf.read()).decode('UTF-8')
`)
} catch (e) {}
}

function copyPlot() {
  if ( pyodide.globals.img_str_["plot-@0"] ) {
    //document.getElementById("plot-@0").src = pyodide.globals.img_str_["plot-@0"]
    //document.getElementById("plot-@0").parentElement.style = ""

    console.html("<hr/>")
    console.html("<img src='" + pyodide.globals.img_str_["plot-@0"] + "' onclick='window.img_Click(\"" + pyodide.globals.img_str_["plot-@0"] + "\")'>")
  }
}

////////////////////////////////////////////////////

function runPython() {
  if (window.pyodide_ready) {
    pyodide.globals.print = (...e) => { e = e.slice(0,-1); console.log(...e) };

    setTimeout(() => {

      try {
        initPlot()

        let fin = pyodide.runPython(`@input`)
        if (fin) {
          console.log(fin)
        }

        copyPlot()

        send.lia("LIA: stop")
      } catch(e) {
        //window.py_packages = ["matplotlib"]

        let module = e.message.match(/ModuleNotFoundError: No module named '([^']+)/g)

        if (! module) {
          console.error(e)
          //let msg = e.message.match(/File "<unknown>", line (\d+)\n.*\n.*\n.*/g)

          //window.console.log(msg[0])

          send.lia("LIA: stop")
        }
        else if (module.length != 0) {
          module = module[0].split("'")[1]

          if (window.pyodide_modules.has(module)) {
            console.error(e)

            send.lia("LIA: stop")
          } else {
            console.debug("downloading module =>", module)
            window.py_packages = [ module ]
            window.pyodide_ready = false
            window.loadModules()
            runPython()
          }
        }
        else {
          console.error(e)

          send.lia("LIA: stop")
        }
      }
    }, 100)
  } else {
    setTimeout(runPython, 234)
  }
}

runPython()

"LIA: wait";
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


![demo](demo.gif)<!-- style="display:none" -->


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
********************************************************************************

1. Load the macros via

   `import: https://github.com/LiaTemplates/Pyodide`

2. Copy the definitions into your Project

3. Clone this repository on GitHub

********************************************************************************

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

--------------------------------------------------------------------------------

                                   --{{1}}--
If you want to use matplotlib, you will have to pass your figure to the `plot`
function, as it is done in the last line below. This function converts your
image into a base64 representation and passes this string to the DOM. It is currently only possible to plot one figure per snippet.

```python
import numpy as np
import matplotlib.pyplot as plt

t = np.arange(0.0, 2.0, 0.01)
s = np.sin(2 * np.pi * t)

fig, ax = plt.subplots()
ax.plot(t, s)

ax.grid(True, linestyle='-.')
ax.tick_params(labelcolor='r', labelsize='medium', width=3)

plt.show()

plot(fig) # <- this is required to plot the fig also on the LiaScript canvas
```
@Pyodide.eval

> For more information see the following
> [codepen-example](https://codepen.io/aagostini/pen/LYExVJL?editors=1000), it
> shows basically how you can add further images.


## Loading Libraries

                                   --{{0}}--

Only the Python standard library and `six` are available at the beginning, other
libraries are globally loaded, if defined within the script. If you know, that certain modules are required, you can speed up their loading by defining them
manually in your `onload` macro, as it is shown below.


``` markdown
<!--
author:  ...
email:   ...

import:  https://github.com/LiaTemplates/Pyodide

@onload: window.py_packages = ["matplotlib", "numpy"]
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
script:   https://pyodide-cdn2.iodide.io/v0.15.0/full/pyodide.js

@onload
window.languagePluginUrl = 'https://pyodide-cdn2.iodide.io/v0.15.0/full/'

window.pyodide_ready = false;

window.pyodide_modules = new Set()

// window.py_packages = ["matplotlib", "numpy"]

window.loadModules = function() {
  languagePluginLoader.then(() => {
    console.log("pyodide is ready")
    if (window.py_packages) {

      for( let i = 0; i < window.py_packages.length; i++ ) {
        window.pyodide_modules.add(window.py_packages[i])
      }

      pyodide.loadPackage(window.py_packages).then(() => {
        console.log("all packages loaded")
        window.pyodide_ready = true;
      });
    }
    else {
      window.pyodide_ready = true;
    }
  })
}

window.loadModules()

@end


@Pyodide.eval: @Pyodide.eval_(@uid)

@Pyodide.eval_
<script>

function initPlot() {
try {

pyodide.runPython(`
import io, base64

try:
  img_str_
except NameError:
  img_str_ = {}

def plot(fig, id="plot-@0"):
  buf = io.BytesIO()
  fig.savefig(buf, format='png')
  buf.seek(0)
  img_str_[id] = "data:image/png;base64," + base64.b64encode(buf.read()).decode('UTF-8')
`)
} catch (e) {}
}

function copyPlot() {
  if ( pyodide.globals.img_str_["plot-@0"] ) {
    //document.getElementById("plot-@0").src = pyodide.globals.img_str_["plot-@0"]
    //document.getElementById("plot-@0").parentElement.style = ""

    console.html("<hr/>")
    console.html("<img src='" + pyodide.globals.img_str_["plot-@0"] + "' onclick='window.img_Click(\"" + pyodide.globals.img_str_["plot-@0"] + "\")'>")
  }
}

////////////////////////////////////////////////////

function runPython() {
  if (window.pyodide_ready) {
    pyodide.globals.print = (...e) => { e = e.slice(0,-1); console.log(...e) };

    setTimeout(() => {

      try {
        initPlot()

        let fin = pyodide.runPython(`@input`)
        if (fin) {
          console.log(fin)
        }

        copyPlot()

        send.lia("LIA: stop")
      } catch(e) {
        //window.py_packages = ["matplotlib"]

        let module = e.message.match(/ModuleNotFoundError: No module named '([^']+)/g)

        if (! module) {
          console.error(e)
          //let msg = e.message.match(/File "<unknown>", line (\d+)\n.*\n.*\n.*/g)

          //window.console.log(msg[0])

          send.lia("LIA: stop")
        }
        else if (module.length != 0) {
          module = module[0].split("'")[1]

          if (window.pyodide_modules.has(module)) {
            console.error(e)

            send.lia("LIA: stop")
          } else {
            console.debug("downloading module =>", module)
            window.py_packages = [ module ]
            window.pyodide_ready = false
            window.loadModules()
            runPython()
          }
        }
        else {
          console.error(e)

          send.lia("LIA: stop")
        }
      }
    }, 100)
  } else {
    setTimeout(runPython, 234)
  }
}

runPython()

"LIA: wait";
</script>

@end
```

                                   --{{1}}--
If you want to minimize loading effort in your LiaScript project, you can also
copy this code and paste it into your main comment header, see the code in the
raw file of this document.


                                     {{1}}
https://raw.githubusercontent.com/LiaTemplates/pyodide/master/README.md
