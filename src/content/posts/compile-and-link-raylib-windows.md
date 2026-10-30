---
title: "[Windows] Compiling and adding raylib to Visual Studio 2022 project"
date: 2022-12-22
---

*Video guide can be found on my youtube channel here: [[Windows] Compiling and adding raylib to Visual Studio 2022 project](https://www.youtube.com/watch?v=ZZV4VrAasEQ)*

If, like me, you've struggled to find online documentation about how to compile [raylib](https://www.raylib.com/) from source and then link it in a project so that you have full access to the internals of the engine to modify at your will, then this tutorial should hopefully streamline that process for you.

To do this, I'm using Visual Studio 2022 on Windows.

## Quick Guide

1. Clone the framework repository from https://github.com/raysan5/raylib (e.g. through `$ git clone https://github.com/raysan5/raylib`)

2. Open Visual Studio 2022 and click **Open a local folder**. This is important - make sure you don't click **Open a project or solution**. Choose the raylib folder you just cloned.

3. Right-click the `CMakeLists.txt` file in the root directory of the project and click **Build**.

4. Once the build is complete, a new folder called "out" should have been created in the root directory of the project, i.e. `raylib/out/`. Take note of the paths `raylib/out/build/x64-Debug/raylib/raylib.lib` and `raylib/out/build/x64-Debug/raylib/include/`, noting that the relative path may be different depending on your architecture. You should note the absolute path to the root (`raylib/out/build/x64-Debug/raylib`), the `.lib` file, and the `/include/` folder, as you will need them for the next steps.

5. You can now open a new project in Visual Studio that you want to link to the compiled raylib framework. If it's a new project, you can copy one of the examples from [raylib.com/examples](https://www.raylib.com/examples.html) into a `main.c` file.

6. Click **Project > \<project-name\> Properties** (where \<project-name\> is the name of your project).

7. Append the absolute path of the `/include/` directory that you noted in step 4 into the value of *Additional Include Directories* in **Configuration Properties > C/C++ > General**

8. Append the absolute path of the root directory of the built library that you noted in step 4 into the value of *Additional Library Directories* in **Configuration Properties > Linker > General**

9. PREPEND `winmm.lib` into the value of *Additional Dependencies* in **Configuration Properties > Linker > Input**. Also note that this should be appended with a delimiting semi-colon if the field isn't empty. This is important that `winmm.lib` should be placed BEFORE any additional dependencies.

10. Append `raylib.lib` into the value of *Additional Dependencies* in **Configuration Properties > Linker > Input**. Also note that this should be prepended with a delimiting semi-colon if the field isn't empty.

11. You can now build and run this project which should now be correctly linked to the locally compiled raylib project. You should make sure to rebuild the local raylib project every time you change the project as per step 3.

And that's it! This guide was designed to be as simple and to-the-point as possible, but if you're having any trouble, please refer to the youtube video linked at the top of this article, and if you're still having trouble, please feel free to leave a youtube comment or email me at [corey@batchler.me](mailto:corey@batchler.me).