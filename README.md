<!--

author:   André Dietrich
email:    LiaScript@web.de
version:  0.4.0
language: en
narrator: US English Male

logo:     logo.jpg

comment:  Use the real Python in your LiaScript courses, by loading this
          template. For more information and to see, which Python-modules are
          accessible visit the [pyodide-website](https://pyodide.org).

script:   https://cdn.jsdelivr.net/pyodide/v314.0.7/full/pyodide.js

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

        def _lia_input(prompt=""):
            import sys
            import js
            from pyodide.ffi import can_run_sync, run_sync

            prompt = str(prompt)
            sys.stdout.write(prompt)
            sys.stdout.flush()

            # run_sync suspends Python until the terminal sends a line,
            # it needs JSPI (JavaScript Promise Integration) in the browser
            if js.pyodideReadLine and can_run_sync():
                return str(run_sync(js.pyodideReadLine())).rstrip("\\r\\n")

            line = js.prompt(prompt)
            line = "" if line is None else str(line)
            print(line)
            return line

        import builtins
        builtins.input = _lia_input
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
        // input() reads from here, if the macro has a terminal
        window.pyodideReadLine = io.readline || null

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
            lines = lines || []
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
        mplout: "target_@0",
        readline: () => new Promise((resolve, reject) => {
            // open the terminal only once, "LIA: terminal" resets it
            if (!terminal) {
                terminal = true
                send.lia("LIA: terminal")
            }
            send.handle("input", resolve)
            send.handle("stop", () => reject(new Error("KeyboardInterrupt")))
        })
    }
    let terminal = false

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

-->

# Pyodide - Template

                                   --{{0}}--
A template for executing Python code in [LiaScript](https://LiaScript.github.io)
based on [Pyodide](https://pyodide.org), a port of CPython to WebAssembly.
It brings Python and many scientific packages such as numpy, pandas and
matplotlib into the browser, see the
[Pyodide project](https://github.com/pyodide/pyodide) for more information.

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
[Implementation](#5). And of course, you could also clone this project and
change it, as you wish.

                                     {{1}}
********************************************************************************

1. Load the macros via

   `import: https://raw.githubusercontent.com/LiaTemplates/Pyodide/master/README.md`

   or use a fixed version, which will not change:

   `import: https://raw.githubusercontent.com/LiaTemplates/Pyodide/0.4.0/README.md`

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

`input()` reads from a terminal below the code block. As soon as your program
asks for input, the terminal opens: type your answer and press Enter.

```python
name = input("What is your name? ")
age = int(input("How old are you? "))

print(f"Hello {name}, in ten years you will be {age + 10}.")
```
@Pyodide.eval

> __Note:__ This requires a browser with JavaScript Promise Integration (JSPI),
>           such as a current Chrome or Edge. Other browsers ask for input with
>           a popup window instead.


--------------------------------------------------------------------------------

                                   --{{1}}--
If you want to use matplotlib, you will have to pass your figure to the `plot`
function, as it is done in the last line below. This function converts your
image into a base64 representation and passes this string to the DOM. It is currently only possible to plot one figure per snippet.

```python
import sys
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.widgets import Button, Slider
from matplotlib.widgets import Slider

# The parametrized function to be plotted
def f(variable, m):
    return np.sin(np.pi*variable*m)**2//np.sin(np.pi*variable)**2

def main():
    x_min = -1.1
    x_max = 1.1
    x_num = 1000
    y_min = 0
    y_max = 100
    x_label = "$\\frac{\\Delta k \\cdot a }{2\\pi}$"
    #x_label = "xlabel"
    #y_label = "ylabel"
    y_label = "$\\|F|^2$"

    # Define slider parameters
    init_m = 4
    m_min = 1
    m_max = 30
    m_step = 1
    slider_label = "Anzahl der Atome"
 
    x = np.linspace(x_min, x_max, x_num)
    # Create the figure and the line that we will manipulate
    fig, ax = plt.subplots()
    line, = ax.plot(x, f(x, init_m), lw=3)
    ax.set_xlabel(x_label)
    ax.set_ylabel(y_label)
    ax.set_ylim(y_min, y_max)
    # adjust the main plot to make room for the sliders
    fig.subplots_adjust(left=0.25, bottom=0.25)
    # Make a horizontal slider to control the frequency.
    axfreq = fig.add_axes([0.25, 0.1, 0.65, 0.03])
    m_slider = Slider(
        ax=axfreq,
        label=slider_label,
        valmin=m_min,
        valmax=m_max,
        valstep=m_step,
        valinit=init_m,
    )
    def update(val):
        line.set_ydata(f(x, m_slider.val))
        fig.canvas.draw_idle()
    def reset(event):
        m_slider.reset()
    # register the update function with each slider
    m_slider.on_changed(update)

    resetax = fig.add_axes([0.8, 0.025, 0.1, 0.04])
    button = Button(resetax, 'Reset', hovercolor='0.975')
    button.on_clicked(reset)  

    plt.show()

main()
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

                                   --{{0}}--
The `@onload` macro defines `runPython`, which loads Pyodide on the first run,
installs packages that your code imports and passes the output to LiaScript.
The macros only add a script-tag that hands the code of your snippet to
`runPython`.


```` js
script:   https://cdn.jsdelivr.net/pyodide/v314.0.7/full/pyodide.js

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

        def _lia_input(prompt=""):
            import sys
            import js
            from pyodide.ffi import can_run_sync, run_sync

            prompt = str(prompt)
            sys.stdout.write(prompt)
            sys.stdout.flush()

            # run_sync suspends Python until the terminal sends a line,
            # it needs JSPI (JavaScript Promise Integration) in the browser
            if js.pyodideReadLine and can_run_sync():
                return str(run_sync(js.pyodideReadLine())).rstrip("\\r\\n")

            line = js.prompt(prompt)
            line = "" if line is None else str(line)
            print(line)
            return line

        import builtins
        builtins.input = _lia_input
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
        // input() reads from here, if the macro has a terminal
        window.pyodideReadLine = io.readline || null

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
            lines = lines || []
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
        mplout: "target_@0",
        readline: () => new Promise((resolve, reject) => {
            // open the terminal only once, "LIA: terminal" resets it
            if (!terminal) {
                terminal = true
                send.lia("LIA: terminal")
            }
            send.handle("input", resolve)
            send.handle("stop", () => reject(new Error("KeyboardInterrupt")))
        })
    }
    let terminal = false

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
````

                                   --{{1}}--
If you want to minimize loading effort in your LiaScript project, you can also
copy this code and paste it into your main comment header, see the code in the
raw file of this document.


                                     {{1}}
https://raw.githubusercontent.com/LiaTemplates/pyodide/master/README.md
