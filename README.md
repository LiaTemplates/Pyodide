<!--

author:   André Dietrich
email:    LiaScript@web.de
version:  0.2.1
language: en
narrator: US English Male

logo:     logo.jpg

comment:  Use the real Python in your LiaScript courses, by loading this
          template. For more information and to see, which Python-modules are
          accessible visit the [pyodide-website](https://alpha.iodide.io).

script:   https://cdn.jsdelivr.net/pyodide/v0.24.0/full/pyodide.js

@Pyodide.eval: @Pyodide.eval_(@uid)

@Pyodide.eval_
<script>
async function run(code) {

    const plot = document.getElementById('target_@0')
    plot.innerHTML = ""
    document.pyodideMplTarget = plot

    if (!window._py) {
        try {
            window._py = await loadPyodide({fullStdLib: false})
            window._py_modules = []
            window._py_running = true
        } catch(e) {
            console.error(e.message)
            send.lia("LIA: stop")
        }
    }

    try {
        window._py.setStdout({ write: (buffer) => {
            const decoder = new TextDecoder()
            const string = decoder.decode(buffer)
            console.stream(string)
            return buffer.length
        }})

        window._py.setStderr({ write: (buffer) => {
            const decoder = new TextDecoder()
            const string = decoder.decode(buffer)
            console.err(string)
            return buffer.length
        }})

        window._py.setStdin({stdin: () => {
          return prompt("stdin")
        }}) 
       
        const rslt = await window._py.runPython(code)

        if (typeof rslt === 'string') {
            send.lia(rslt)
        }
    } catch(e) {
        let module = e.message.match(/ModuleNotFoundError: The module '([^']+)/i)

        window.console.warn("Pyodide", e.message)
    
        if (!module) {
            const err = e.message.match(/File "<exec>", line (\d+).*\n((.*\n){1,3})/i)

            if (err!== null && err.length >= 3) {
                send.lia( e.message,
                  [[{ row : parseInt(err[1]) - 1,
                      column : 1,
                      text : err[2],
                      type : "error"
                  }]],
                  false)
            } else {
                console.error(e.message)
            }
        } else {
            if (module.length > 1) {
                module = module[1]

                if (window._py_modules.includes(module)) {
                    console.error(e.message)
                } else {
                    console.debug("downloading module =>", module)
                    window._py_modules.push(module)
                    await window._py.loadPackage(module)
                    await run(code)
                }
            }
        }
    }
    send.lia("LIA: stop")
    window._py_running = false
}

if (window._py_running) {
  setTimeout(() => {
    console.warn("Another process is running, wait until finished")
  }, 500)
  "LIA: stop"
} else {
  window._py_running = true

  setTimeout(() => {
    run(`@input`)
  }, 500)

  "LIA: wait"
}
</script>

<div id="target_@0"></div>
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

https://liascript.github.io/course/?https://raw.githubusercontent.com/LiaTemplates/Pyodide/master/README.md


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

   `import: https://raw.githubusercontent.com/LiaTemplates/Pyodide/master/README.md`

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
```
@Pyodide.eval


``` python
import pandas as pd
d = {'col1': [1, 5, 7], 'col2': [3, .4, -2], 'col3':["yes", "no","blue"]};
df = pd.DataFrame(data=d);
df
print(df)
```
@Pyodide.eval



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
script:   https://cdn.jsdelivr.net/pyodide/v0.24.0/full/pyodide.js

@Pyodide.eval: @Pyodide.eval_(@uid)

@Pyodide.eval_
<script>
async function run(code) {

    const plot = document.getElementById('target_@0')
    plot.innerHTML = ""
    document.pyodideMplTarget = plot

    if (!window._py) {
        try {
            window._py = await loadPyodide({fullStdLib: false})
            window._py_modules = []
            window._py_running = true
        } catch(e) {
            console.error(e.message)
            send.lia("LIA: stop")
        }
    }

    try {
        window._py.setStdout({ write: (buffer) => {
            const decoder = new TextDecoder()
            const string = decoder.decode(buffer)
            console.stream(string)
            return buffer.length
        }})

        window._py.setStderr({ write: (buffer) => {
            const decoder = new TextDecoder()
            const string = decoder.decode(buffer)
            console.err(string)
            return buffer.length
        }})

        window._py.setStdin({stdin: () => {
          return prompt("stdin")
        }}) 
       
        const rslt = await window._py.runPython(code, {})

        if (typeof rslt === 'string') {
            send.lia(rslt)
        }
    } catch(e) {
        let module = e.message.match(/ModuleNotFoundError: The module '([^']+)/i)

        window.console.warn("Pyodide", e.message)
    
        if (!module) {
            const err = e.message.match(/File "<exec>", line (\d+).*\n((.*\n){1,3})/i)

            if (err!== null && err.length >= 3) {
                send.lia( e.message,
                  [[{ row : parseInt(err[1]) - 1,
                      column : 1,
                      text : err[2],
                      type : "error"
                  }]],
                  false)
            } else {
                console.error(e.message)
            }
        } else {
            if (module.length > 1) {
                module = module[1]

                if (window._py_modules.includes(module)) {
                    console.error(e.message)
                } else {
                    console.debug("downloading module =>", module)
                    window._py_modules.push(module)
                    await window._py.loadPackage(module)
                    await run(code)
                }
            }
        }
    }
    send.lia("LIA: stop")
    window._py_running = false
}

if (window._py_running) {
  setTimeout(() => {
    console.warn("Another process is running, wait until finished")
  }, 500)
  "LIA: stop"
} else {
  window._py_running = true

  setTimeout(() => {
    run(`@input`)
  }, 500)

  "LIA: wait"
}
</script>

<div id="target_@0"></div>
@end
```

                                   --{{1}}--
If you want to minimize loading effort in your LiaScript project, you can also
copy this code and paste it into your main comment header, see the code in the
raw file of this document.


                                     {{1}}
https://raw.githubusercontent.com/LiaTemplates/pyodide/master/README.md
