<!--

author:   André Dietrich
email:    LiaScript@web.de
version:  0.5.0
language: en
narrator: US English Male

logo:     logo.jpg

comment:  Use the real Python in your LiaScript courses, by loading this
          template. For more information and to see, which Python-modules are
          accessible visit the [pyodide-website](https://pyodide.org).

script:   https://cdn.jsdelivr.net/pyodide/v314.0.8/full/pyodide.js

@onload
async function loadPython() {
    const pyodide = await loadPyodide({ fullStdLib: false })
    pyodide.setStdin({ stdin: () => prompt("stdin") })
    await pyodide.runPythonAsync(`
        import sys
        import contextvars

        # Every run gets its own io (output, input, plot target) through a
        # context variable, a run that waits for input() can be passed by
        # other runs. The namespace is shared by all code blocks.
        _lia_io = contextvars.ContextVar("_lia_io", default=None)

        class _LiaBuffer:
            def __init__(self, stream):
                self.stream = stream

            def write(self, data):
                self.stream.write(bytes(data).decode("utf-8", "replace"))
                return len(data)

            def flush(self):
                pass

        class _LiaStream:
            def __init__(self, name, fallback):
                self.name = name
                self.fallback = fallback
                self.encoding = "utf-8"
                self.errors = "strict"
                self.buffer = _LiaBuffer(self)

            def write(self, text):
                io = _lia_io.get()
                if io is None:
                    return self.fallback.write(text)
                getattr(io, self.name)(text)
                return len(text)

            def flush(self):
                pass

            def isatty(self):
                return False

            def writable(self):
                return True

        sys.stdout = _LiaStream("stdout", sys.stdout)
        sys.stderr = _LiaStream("stderr", sys.stderr)

        # Callbacks from JavaScript (e.g. matplotlib widgets) run outside of
        # any run, they keep the io of the run that created them
        import inspect
        import pyodide.ffi

        _lia_create_proxy = pyodide.ffi.create_proxy

        def _lia_proxy(obj, *args, **kwargs):
            io = _lia_io.get()
            if io is not None and (inspect.isfunction(obj) or inspect.ismethod(obj)):
                func = obj

                def obj(*a, **kw):
                    if _lia_io.get() is not None:
                        return func(*a, **kw)
                    token = _lia_io.set(io)
                    try:
                        return func(*a, **kw)
                    finally:
                        _lia_io.reset(token)

            return _lia_create_proxy(obj, *args, **kwargs)

        pyodide.ffi.create_proxy = _lia_proxy

        def _lia_plot_target():
            import js

            io = _lia_io.get()
            if io is not None:
                js.document.pyodideMplTarget = js.document.getElementById(io.mplout)

        def _lia_figures():
            plt = sys.modules.get("matplotlib.pyplot")
            return set(plt.get_fignums()) if plt else set()

        # plt.show() draws all open figures, so figures of finished runs are
        # closed when the next run starts, figures of waiting runs are kept
        _lia_finished_figures = set()

        def _lia_close_finished_figures():
            plt = sys.modules.get("matplotlib.pyplot")
            if plt:
                for number in _lia_finished_figures & _lia_figures():
                    plt.close(number)
            _lia_finished_figures.clear()

        def _lia_error(error):
            import traceback

            summary = traceback.format_exception_only(type(error), error)[-1].strip()
            if isinstance(error, SyntaxError):
                lines = [(error.lineno, error.offset)]
            else:
                frames = traceback.extract_tb(error.__traceback__)
                lines = [(frame.lineno, frame.colno) for frame in frames if frame.filename == "<exec>"]

            details = traceback.TracebackException.from_exception(error)
            details.stack = traceback.StackSummary.from_list(
                [frame for frame in details.stack if frame.filename == "<exec>"]
            )
            return ["".join(details.format()), summary, lines]

        async def _lia_run(code, io):
            import js
            from pyodide.code import eval_code_async
            from pyodide.ffi import to_js

            _lia_io.set(io)
            _lia_close_finished_figures()
            _lia_plot_target()
            before = _lia_figures()
            try:
                result = await eval_code_async(code, globals())
                output = {"result": None if result is None else str(result)}
            except BaseException as error:
                output = {"error": _lia_error(error)}
            finally:
                _lia_finished_figures.update(_lia_figures() - before)
            return to_js(output, dict_converter=js.Object.fromEntries)

        def _lia_input(prompt=""):
            from pyodide.ffi import can_run_sync, run_sync

            prompt = str(prompt)
            sys.stdout.write(prompt)

            # run_sync suspends this run until the terminal sends a line,
            # it needs JSPI (JavaScript Promise Integration) in the browser
            io = _lia_io.get()
            readline = getattr(io, "readline", None) if io is not None else None
            if readline and can_run_sync():
                from pyodide.ffi import JsException

                try:
                    line = run_sync(readline())
                except JsException:
                    raise KeyboardInterrupt from None  # stop button
                _lia_plot_target()  # another run might have changed it
                return str(line).rstrip("\\r\\n")

            import js
            line = js.prompt(prompt)
            line = "" if line is None else str(line)
            print(line)
            return line

        import builtins
        builtins.input = _lia_input

        # matplotlib's Pyodide backend draws with float coordinates,
        # which matplotlib >= 3.10 reports as deprecated on every plot
        import warnings
        warnings.filterwarnings(
            "ignore",
            message="The [a-z0-9_]+ parameter as float was deprecated",
            category=DeprecationWarning,
        )
    `, { filename: "<lia-pyodide>" })
    window.pyodideRun = pyodide.globals.get("_lia_run")
    return pyodide
}

// Runs are queued and executed one after the other, in the order they were
// started. A run that waits for input() leaves the queue and rejoins it
// after the answer, so other code blocks can run in the meantime.
function pyodideQueue() {
    let release
    const turn = window.pyodideQueueTail || Promise.resolve()
    window.pyodideQueueTail = turn.then(() => new Promise((resolve) => { release = resolve }))
    return turn.then(() => release)
}

async function runPython(code, io) {
    const plot = document.getElementById(io.mplout)
    if (plot) {
        plot.innerHTML = ""
    }

    let release = await pyodideQueue()
    const runIO = Object.assign({}, io)
    if (io.readline) {
        runIO.readline = async () => {
            release()
            try {
                return await io.readline()
            } finally {
                release = await pyodideQueue()
            }
        }
    }

    try {
        // Pyodide is loaded once, all runs wait for the same promise
        window.pyodideReady = window.pyodideReady || loadPython()
        window.pyodide = await window.pyodideReady

        await window.pyodide.loadPackagesFromImports(code, {
            messageCallback: (msg) => io.stdout(msg + "\n"),
            errorCallback: (msg) => io.stderr(msg + "\n")
        })
        const out = await window.pyodideRun(code, runIO)

        if (out.error) {
            io.liaerr(...out.error)
        } else if (out.result !== undefined) {
            io.liaout(out.result)
        } else if (io.clearOut) {
            io.liaout("")
        }
    } catch (e) {
        if (!window.pyodide) {
            window.pyodideReady = null
        }
        io.liaerr(e.message)
    } finally {
        release()
    }
    io.liaout("LIA: stop")
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
    // print() is written to the browser console, line by line
    let line = ""
    const byLine = (log) => (text) => {
        const lines = (line + text).split("\n")
        line = lines.pop()
        lines.forEach((l) => log(l))
    }

    const io = {
        stdout: byLine(console.log),
        stderr: byLine(console.error),
        liaout: send.lia,
        liaerr: (text) => send.lia(`HTML: <pre style='color: red'>${text}</pre>`),
        clearOut: true,
        mplout: "target_@0"
    }

    await window.runPython(code, io)
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
    let terminal = false

    const io = {
        stdout: (text) => console.stream(text),
        stderr: (text) => send.log("error", '', [text]),
        liaout: send.lia,
        liaerr: (fullMessage, lineMessage, lines) => {
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

    await window.runPython(code, io)
}

setTimeout(run_eval, 500)

"LIA: wait"
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

   `import: https://raw.githubusercontent.com/LiaTemplates/Pyodide/0.5.0/README.md`

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
Code blocks run one after the other, in the order they were started. A code
block that waits for input steps aside, so other code blocks can be run in the
meantime. All code blocks share one Python namespace, so variables defined in
one block can be used in the others.

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
script:   https://cdn.jsdelivr.net/pyodide/v314.0.8/full/pyodide.js

@onload
async function loadPython() {
    const pyodide = await loadPyodide({ fullStdLib: false })
    pyodide.setStdin({ stdin: () => prompt("stdin") })
    await pyodide.runPythonAsync(`
        import sys
        import contextvars

        # Every run gets its own io (output, input, plot target) through a
        # context variable, a run that waits for input() can be passed by
        # other runs. The namespace is shared by all code blocks.
        _lia_io = contextvars.ContextVar("_lia_io", default=None)

        class _LiaBuffer:
            def __init__(self, stream):
                self.stream = stream

            def write(self, data):
                self.stream.write(bytes(data).decode("utf-8", "replace"))
                return len(data)

            def flush(self):
                pass

        class _LiaStream:
            def __init__(self, name, fallback):
                self.name = name
                self.fallback = fallback
                self.encoding = "utf-8"
                self.errors = "strict"
                self.buffer = _LiaBuffer(self)

            def write(self, text):
                io = _lia_io.get()
                if io is None:
                    return self.fallback.write(text)
                getattr(io, self.name)(text)
                return len(text)

            def flush(self):
                pass

            def isatty(self):
                return False

            def writable(self):
                return True

        sys.stdout = _LiaStream("stdout", sys.stdout)
        sys.stderr = _LiaStream("stderr", sys.stderr)

        # Callbacks from JavaScript (e.g. matplotlib widgets) run outside of
        # any run, they keep the io of the run that created them
        import inspect
        import pyodide.ffi

        _lia_create_proxy = pyodide.ffi.create_proxy

        def _lia_proxy(obj, *args, **kwargs):
            io = _lia_io.get()
            if io is not None and (inspect.isfunction(obj) or inspect.ismethod(obj)):
                func = obj

                def obj(*a, **kw):
                    if _lia_io.get() is not None:
                        return func(*a, **kw)
                    token = _lia_io.set(io)
                    try:
                        return func(*a, **kw)
                    finally:
                        _lia_io.reset(token)

            return _lia_create_proxy(obj, *args, **kwargs)

        pyodide.ffi.create_proxy = _lia_proxy

        def _lia_plot_target():
            import js

            io = _lia_io.get()
            if io is not None:
                js.document.pyodideMplTarget = js.document.getElementById(io.mplout)

        def _lia_figures():
            plt = sys.modules.get("matplotlib.pyplot")
            return set(plt.get_fignums()) if plt else set()

        # plt.show() draws all open figures, so figures of finished runs are
        # closed when the next run starts, figures of waiting runs are kept
        _lia_finished_figures = set()

        def _lia_close_finished_figures():
            plt = sys.modules.get("matplotlib.pyplot")
            if plt:
                for number in _lia_finished_figures & _lia_figures():
                    plt.close(number)
            _lia_finished_figures.clear()

        def _lia_error(error):
            import traceback

            summary = traceback.format_exception_only(type(error), error)[-1].strip()
            if isinstance(error, SyntaxError):
                lines = [(error.lineno, error.offset)]
            else:
                frames = traceback.extract_tb(error.__traceback__)
                lines = [(frame.lineno, frame.colno) for frame in frames if frame.filename == "<exec>"]

            details = traceback.TracebackException.from_exception(error)
            details.stack = traceback.StackSummary.from_list(
                [frame for frame in details.stack if frame.filename == "<exec>"]
            )
            return ["".join(details.format()), summary, lines]

        async def _lia_run(code, io):
            import js
            from pyodide.code import eval_code_async
            from pyodide.ffi import to_js

            _lia_io.set(io)
            _lia_close_finished_figures()
            _lia_plot_target()
            before = _lia_figures()
            try:
                result = await eval_code_async(code, globals())
                output = {"result": None if result is None else str(result)}
            except BaseException as error:
                output = {"error": _lia_error(error)}
            finally:
                _lia_finished_figures.update(_lia_figures() - before)
            return to_js(output, dict_converter=js.Object.fromEntries)

        def _lia_input(prompt=""):
            from pyodide.ffi import can_run_sync, run_sync

            prompt = str(prompt)
            sys.stdout.write(prompt)

            # run_sync suspends this run until the terminal sends a line,
            # it needs JSPI (JavaScript Promise Integration) in the browser
            io = _lia_io.get()
            readline = getattr(io, "readline", None) if io is not None else None
            if readline and can_run_sync():
                from pyodide.ffi import JsException

                try:
                    line = run_sync(readline())
                except JsException:
                    raise KeyboardInterrupt from None  # stop button
                _lia_plot_target()  # another run might have changed it
                return str(line).rstrip("\\r\\n")

            import js
            line = js.prompt(prompt)
            line = "" if line is None else str(line)
            print(line)
            return line

        import builtins
        builtins.input = _lia_input

        # matplotlib's Pyodide backend draws with float coordinates,
        # which matplotlib >= 3.10 reports as deprecated on every plot
        import warnings
        warnings.filterwarnings(
            "ignore",
            message="The [a-z0-9_]+ parameter as float was deprecated",
            category=DeprecationWarning,
        )
    `, { filename: "<lia-pyodide>" })
    window.pyodideRun = pyodide.globals.get("_lia_run")
    return pyodide
}

// Runs are queued and executed one after the other, in the order they were
// started. A run that waits for input() leaves the queue and rejoins it
// after the answer, so other code blocks can run in the meantime.
function pyodideQueue() {
    let release
    const turn = window.pyodideQueueTail || Promise.resolve()
    window.pyodideQueueTail = turn.then(() => new Promise((resolve) => { release = resolve }))
    return turn.then(() => release)
}

async function runPython(code, io) {
    const plot = document.getElementById(io.mplout)
    if (plot) {
        plot.innerHTML = ""
    }

    let release = await pyodideQueue()
    const runIO = Object.assign({}, io)
    if (io.readline) {
        runIO.readline = async () => {
            release()
            try {
                return await io.readline()
            } finally {
                release = await pyodideQueue()
            }
        }
    }

    try {
        // Pyodide is loaded once, all runs wait for the same promise
        window.pyodideReady = window.pyodideReady || loadPython()
        window.pyodide = await window.pyodideReady

        await window.pyodide.loadPackagesFromImports(code, {
            messageCallback: (msg) => io.stdout(msg + "\n"),
            errorCallback: (msg) => io.stderr(msg + "\n")
        })
        const out = await window.pyodideRun(code, runIO)

        if (out.error) {
            io.liaerr(...out.error)
        } else if (out.result !== undefined) {
            io.liaout(out.result)
        } else if (io.clearOut) {
            io.liaout("")
        }
    } catch (e) {
        if (!window.pyodide) {
            window.pyodideReady = null
        }
        io.liaerr(e.message)
    } finally {
        release()
    }
    io.liaout("LIA: stop")
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
    // print() is written to the browser console, line by line
    let line = ""
    const byLine = (log) => (text) => {
        const lines = (line + text).split("\n")
        line = lines.pop()
        lines.forEach((l) => log(l))
    }

    const io = {
        stdout: byLine(console.log),
        stderr: byLine(console.error),
        liaout: send.lia,
        liaerr: (text) => send.lia(`HTML: <pre style='color: red'>${text}</pre>`),
        clearOut: true,
        mplout: "target_@0"
    }

    await window.runPython(code, io)
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
    let terminal = false

    const io = {
        stdout: (text) => console.stream(text),
        stderr: (text) => send.log("error", '', [text]),
        liaout: send.lia,
        liaerr: (fullMessage, lineMessage, lines) => {
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

    await window.runPython(code, io)
}

setTimeout(run_eval, 500)

"LIA: wait"
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
