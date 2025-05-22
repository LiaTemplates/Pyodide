<!--

author:   André Dietrich
email:    LiaScript@web.de
version:  0.3.4
language: en
narrator: US English Male

logo:     logo.jpg

comment:  Use the real Python in your LiaScript courses, by loading this
          template. For more information and to see, which Python-modules are
          accessible visit the [pyodide-website](https://alpha.iodide.io).

@@ snippet start

script:   https://cdn.jsdelivr.net/pyodide/v0.27.3/full/pyodide.js

@onload
async function loadPython() {
    const pyodide = await loadPyodide({ fullStdLib: false })
    await pyodide.runPythonAsync(`
        def _lia_process_exception():
            import sys
            import traceback

            summary = traceback.format_exception_only(sys.last_type, sys.last_value)[-1].strip()
            if isinstance(sys.last_value, SyntaxError):
                lines = [(sys.last_value.lineno, sys.last_value.offset)]
            else:
                frames = traceback.extract_tb(sys.last_traceback)
                lines = [(frame.lineno, frame.colno) for frame in frames if frame.filename == '<exec>']
            return summary, lines

        def _lia_flush_streams():
            import sys

            sys.stdout.flush()
            sys.stderr.flush()
    `)
    return pyodide
}

async function runPython(code, io) {
    const plot = document.getElementById(io.mplout)
    plot.innerHTML = ""
    document.pyodideMplTarget = plot

    if (!window.pyodide) {
        try {
            window.pyodide = await loadPython()
            window.pyodide_running = true
        } catch (e) {
            io.liaerr(e.message)
            io.liaout("LIA: stop")
            return
        }
    }

    try {
        window.pyodide.setStdout(io.stdout)
        window.pyodide.setStderr(io.stderr)
        window.pyodide.setStdin({
            stdin: () => {
                return prompt("stdin")
            }
        })

        await window.pyodide.loadPackagesFromImports(code)
        const rslt = await window.pyodide.runPythonAsync(code)

        if (typeof rslt === 'string') {
            io.liaout(rslt)
        } else if (rslt !== undefined && typeof rslt.toString === 'function') {
            io.liaout(rslt.toString())
        } else if (io.clearOut) {
            io.liaout("")
        }
    } catch (e) {
        if (e instanceof window.pyodide.ffi.PythonError) {
            const out = await window.pyodide.runPythonAsync("_lia_process_exception()")
            const [errString, lines] = out.toJs({create_pyproxies : false})
            io.liaerr(e.message, errString, lines)
        } else {
            io.liaerr(e.message)
        }
    }
    await window.pyodide.runPythonAsync("_lia_flush_streams()")
    io.liaout("LIA: stop")
    window.pyodide_running = false
}

window.runPython = runPython
@end


@Pyodide.exec: @Pyodide.exec_(@uid,```@0```)

@Pyodide.exec_
<script run-once modify="# --python--\n" type="text/python">

async function run_exec() {
    const code = String.raw`# --python--
@1
# --python--
`
    if (!window.pyodide_running) {
        window.pyodide_running = true

        const io = {
            stdout: {batched: console.log},
            stderr: {batched: console.error},
            liaout: send.lia,
            liaerr: (text) => send.lia(`HTML: <pre style='color: red'>${text}</pre>`),
            clearOut: true,
            mplout: "target_@0"
        }

        await window.runPython(code, io)
    } else {
        setTimeout(run_exec, 1000)
    }
}

setTimeout(run_exec, 500)

"calculating, please wait ..."

</script>

<div id="target_@0"></div>
@end


@Pyodide.eval: @Pyodide.eval_(@uid)

@Pyodide.eval_
<script>

async function run_eval() {
    const code = "@'input"
    const io = {
        stdout: {
            write: (buffer) => {
                const decoder = new TextDecoder()
                const string = decoder.decode(buffer)
                console.stream(string)
                return buffer.length
            }
        },
        stderr: {
            write: (buffer) => {
                const decoder = new TextDecoder()
                const string = decoder.decode(buffer)
                send.log("error", '', [string])
                return buffer.length
            }
        },
        liaout: send.lia,
        liaerr: (fullMessage, lineMessage, lines) => {
            window.console.log(lines)
            let lineErrors = [[]]
            for (const [i, line] of lines.entries()) {
                const last = (i + 1 == lines.length)
                lineErrors[0].push({
                    row: line[0] - 1,  // Off-by-one; not sure why
                    column: line[1],
                    text: last ? lineMessage : "Called from here",
                    type: last ? "error" : "warning"
                })
            }
            send.lia(fullMessage, lineErrors, false)
        },
        clearOut: false,
        mplout: "target_@0"
    }

    await window.runPython(code, io)
}

if (window.pyodide_running) {
    setTimeout(() => {
        console.warn("Another process is running, wait until finished")
    }, 500)

    "LIA: stop"
} else {
    window.pyodide_running = true
    setTimeout(run_eval, 500)

    "LIA: wait"
}
</script>

<div id="target_@0"></div>
@end

@@ snippet stop

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
other branch or version. But you can also copy the required functionality
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


## `@Pyodide.exec`

This macro works similar to the previous one, but the code is only passed as a parameter.
The user will only see the result and will not have the chance to directly modify the the Python code.

```python   @Pyodide.exec
import sys

for i in range(5):
	print("Hello", 'World #', i)

sys.version
```


```python   @Pyodide.exec
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

## Loading Libraries

                                   --{{0}}--

Only the Python standard library and `six` are available at the beginning, other
libraries are globally loaded, if defined within the script.

> __Note:__ loading large packages such as `scipy` may take some time, since
>           they might require to download many MB of precompiled packages.

## Implementation
<!--

@loadsnippet
<script style="display: block" modify="false" run-once="true">
async function loadSnippet() {
    const mdPath = document.location.search.slice(1)  // Remove "?"
    try {
        const response = await fetch(mdPath)
        if (!response.ok) {
            throw new Error("Got a non-okay response")
        }
        const text = await response.text()
        const snippet = text.split("@@ snippet start")[1].split("@@ snippet stop")[0].trim()
        send.lia("LIASCRIPT:\n```` @0\n" + snippet.replaceAll("````", "XXXX") + "\n````")
    } catch (e) {
        send.lia(`HTML: <span style='color: red'>Something went wrong, could not load <a href='${mdPath}'>${mdPath}</a></span>`)
    }
}
loadSnippet()

"loading source"
</script>
@end

-->

                                   --{{0}}--
This macro implementation only adds a simple script-tag that pushes the code of
your snippet directly to Pyodide. The `@onload` macro is required to instantiate
Pyodide and load the required libraries, which might require some time, since
the loaded packages might be quite large.

@loadsnippet(js)


                                   --{{1}}--
If you want to minimize loading effort in your LiaScript project, you can also
copy this code and paste it into your main comment header, see the code in the
raw file of this document.


                                     {{1}}
https://raw.githubusercontent.com/LiaTemplates/pyodide/master/README.md
