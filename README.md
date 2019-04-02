<!--

author:   André Dietrich
email:    andre.dietrich@ovgu.de
version:  0.0.1
language: en
narrator: US English Female

script:  ./js/pyodide.js

script:  https://aframe.io/releases/0.8.0/aframe.min.js

@onload
languagePluginLoader.then(() => {
  console.log("pyodide is ready")
  if (window.py_packages) {
    pyodide.loadPackage(window.py_packages).then(() => {
      console.log("all packages loaded")
    });
  }
});
var module = {};

window.load_packages = function (list) {
  window.py_packages = list;
}

@end

@Pyodide.eval
<script>
pyodide.globals.print = (...e) => { e = e.slice(0,-1); console.log(...e) };
pyodide.runPython(`@input`);
</script>

@end

-->

# Pyodide_template

                                   --{{0}}--
A template for executing Python code in [LiaScript](https://LiaScript.github.io)
based on the [Pyodide](https://github.com/iodide-project/pyodide) webassembly
port to JavaScript. This port tries to make Python scientific programming
accessible within the browser, see the [Iodide project](https://iodide.io) for
more information.

__Try it on LiaScript:__

https://liascript.github.io/course/?https://raw.githubusercontent.com/liaScript/pyodide_template/master/README.md

__See the project on Github:__

https://github.com/liaScript/pyodide_template

                                   --{{1}}--
There are three ways to use this template. The easiest way is to use the
`import` statement and the url of the raw text-file of the master branch or any
other branch or version. But you can also copy the required functionionality
directly into the header of your Markdown document, see therefor the
[Implementation](#3). And of course, you could also clone this project and
change it, as you wish.

                                     {{1}}
1. Load the macros via

   `import: https://raw.githubusercontent.com/liaScript/pyodide_template/master/README.md`

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

import:  https://raw.githubusercontent.com/liaScript/pyodide_template/master/README.md

@onload: load_packages(["matplotlib", "numpy"]);
-->
...
```

> __Note:__ `scipy` is currently not available, due to memory restrictions at
>            github, which does not allow to upload files larger than 100 MB.


## Implementation

```js

```


## webgl

``` js
var loader = new THREE.GLTFLoader()
var scene = document.querySelector('a-scene').object3D; // THREE.Scene
loader.load("https://devinbayly.github.io/host_brain/iodide.gltf",
    //standard threejs gltf loading call
    function(gltf) {
        gltf.scene.position.z = -20
        gltf.scene.rotation.y = (3.14 / 2)
        scene.add(gltf.scene);
    },
    // called while loading is progressing
    function(xhr) {

        console.log((xhr.loaded / xhr.total * 100) + '% loaded');

    },
    // called when loading has errors
    function(error) {

        console.log('An error happened');

    }
);

var COM = {
    gravity: 0.017,
    position: new THREE.Vector3(0, 5, -20),
    perturb: function() {
        //purturb the objects within?
    },
    init: function() {
        scene.add(this.mesh)
    }
}

// orbit code courtesy of Tyler j Gabb
function animate() {
    requestAnimationFrame(animate);
    orbiters.forEach(o => o.update());
    //renderer.render(scene, camera);
}

let COMgeom = new THREE.SphereGeometry(1, 1, 1);
let COMmat = new THREE.MeshBasicMaterial({
    color: 0xFF0000
});
let COMmesh = new THREE.Mesh(COMgeom, COMmat);
//scene.add(COMmesh);

class Orbiter {
    constructor() {
        this.velocity = new THREE.Vector2(0.3 * Math.random()/100, 0.3 * Math.random()/100);
        let geom = new THREE.SphereGeometry(.5, .5, .5);
        let mat = new THREE.MeshBasicMaterial({
            color: 0x0f97fa, //seagreen
            //wireframe: true
        });
        this.mesh = new THREE.Mesh(geom, mat);
        this.mesh.position.set(Math.random() * 10, Math.random() * 10, -20);
        scene.add(this.mesh);
    }

    update() {
        let acc = new THREE.Vector2(0, 0);
        acc.subVectors(COM.position, this.mesh.position);
        acc.normalize();
        acc.multiplyScalar(COM.gravity);

        let vel = new THREE.Vector2(0, 0);
        vel.addVectors(this.velocity, acc);
        if (vel.length() > 1) vel.setLength(1);
        this.velocity.set(vel.x, vel.y);

        let pos = new THREE.Vector2(0, 0);
        pos.addVectors(this.mesh.position, this.velocity);
        this.mesh.position.set(pos.x, pos.y, -20);
    }
}

var orbiters = []
for (let i = 0; i < 7; i++) {
    orbiters.push(new Orbiter());
}

animate();

function update(x, y) {
    var ruler = new THREE.Vector2(0, 0);
    ruler.subVectors(COM.position, new THREE.Vector2(x , y ));
    COM.position.set(x , y );
    COMmesh.position.set(x , y , -20);
    //console.log(ruler.length());
    if (ruler.length() > 5) setTimeout(() => calm(), 1000);
}

function increaseGravity() {
    if (COM.gravity < 0.145) return COM.gravity += 0.01;
    return COM.gravity;
}

function decreaseGravity() {
    if (COM.gravity > 0.015) return COM.gravity -= 0.01;
    return COM.gravity;
}

var calming = false;

function calm() {
    if (calming) return;
    setTimeout(() => calming = false, 4000);
    calming = true;
    COM.gravity = 0.1;
    for (let i = 1; i < 4; i++) {
        setTimeout(() => decreaseGravity(), 1000 * i); //0.1, 0.09, 0.08, 0.07
    }
    COM.gravity = 0.07;
}

setInterval(()=> {
  update(Math.random()*2,Math.random()*2)
},2000)
```
<script>@input</script>


<div>
<a-scene embedded style="height:300px; width:100%">
<a-sky color="black"></a-sky>

</a-scene>
</div>
