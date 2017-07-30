# cmake tutorial

翻译自 [cmake-tutorial](https://cmake.org/cmake-tutorial/)

译者 zhangmc 邮箱 zhangmc1993@qq.com

Below is a step-by-step tutorial covering common build system issues that CMake helps to address. Many of these topics have been introduced in [Mastering CMake](https://www.kitware.com/what-we-offer/#books) as separate issues but seeing how they all work together in an example project can be very helpful. This tutorial can be found in the [Tests/Tutorial](https://gitlab.kitware.com/cmake/cmake/tree/master/Tests/Tutorial) directory of the CMake source code tree. Each step has its own subdirectory containing a complete copy of the tutorial for that step

如下教程将一步步指引如何使用 CMake 解决通用的编译问题。其中很多问题已经在 [Mastering CMake](https://www.kitware.com/what-we-offer/#books) 作为独立部分予以讨论，但是他们如何在一个样例中协同工作是本文的重点。本教程的源码可以在 [Tests/Tutorial](https://gitlab.kitware.com/cmake/cmake/tree/master/Tests/Tutorial) 中找到，如下的每一步都对应着代码库中的一个子文件夹。

## Step 1 A Basic Starting Point 第一步 开始

The most basic project is an executable built from source code files. For simple projects a two line CMakeLists.txt file is all that is required. This will be the starting point for our tutorial. The CMakeLists.txt file looks like:

最基础的项目就是使用源文件编译可执行文件。简单项目的 CMakeLists.txt 文件只需要如下几行，教程也从简单项目开始。CMakeLists.txt 如下所示:

```
cmake_minimum_required (VERSION 2.6)
project (Tutorial)
add_executable(Tutorial tutorial.cxx)
```

Note that this example uses lower case commands in the CMakeLists.txt file. Upper, lower, and mixed case commands are supported by CMake. The source code for tutorial.cxx will compute the square root of a number and the first version of it is very simple, as follows:

需要注意，本例中的 Cmake 命令全部使用小写。CMake 命令可以使用小写、大写或者混合拼写。源文件 tutorial.cxx 完成计算平方根的工作，第一版的代码非常简单，如下所示:

```cpp
// A simple program that computes the square root of a number
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
int main (int argc, char *argv[]){
    if (argc < 2){
        fprintf(stdout,"Usage: %s number\n", argv[0]);
        return 1;
    }
    double inputValue = atof(argv[1]);
    double outputValue = sqrt(inputValue);
    fprintf(stdout,"The square root of %g is %g\n", inputValue, outputValue);
    return 0;
}
```

### Adding a Version Number and Configured Header File 添加版本号以及配置头文件

The first feature we will add is to provide our executable and project with a version number. While you can do this exclusively in the source code, doing it in the CMakeLists.txt file provides more flexibility. To add a version number we modify the CMakeLists.txt file as follows:

第一个要添加的特性是项目版本号。当然可以在源码中添加版本号，但是在 CMakeLists.txt 中添加更具灵活性。将 CMakeLists.txt 修改如下:

```
cmake_minimum_required (VERSION 2.6)
project (Tutorial)
# The version number.
# 版本号
set (Tutorial_VERSION_MAJOR 1)
set (Tutorial_VERSION_MINOR 0)
 
# configure a header file to pass some of the CMake settings
# to the source code
# 配置头文件，将 CMake 参数传递给源码
configure_file (
  "${PROJECT_SOURCE_DIR}/TutorialConfig.h.in"
  "${PROJECT_BINARY_DIR}/TutorialConfig.h"
  )
 
# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
# 添加 TutorialConfig.h 所在路径到头文件的搜索路径中
include_directories("${PROJECT_BINARY_DIR}")
 
# add the executable
# 生成可执行文件
add_executable(Tutorial tutorial.cxx)
```

Since the configured file will be written into the binary tree we must add that directory to the list of paths to search for include files. We then create a TutorialConfig.h.in file in the source tree with the following contents:

配置文件(.h)必须加入到源码树中，所以需要将配置文件所在路径加入到头文件的搜索路径中。后续创建 TutorialConfig.h.in 文件，内容如下:

```
// the configured options and settings for Tutorial
// 对应 CMakeLists.txt 中的选项和配置
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
```

When CMake configures this header file the values for @Tutorial_VERSION_MAJOR@ and @Tutorial_VERSION_MINOR@ will be replaced by the values from the CMakeLists.txt file. Next we modify tutorial.cxx to include the configured header file and to make use of the version numbers. The resulting source code is listed below.

CMake 会将该文件中的 @Tutorial_VERSION_MAJOR@ 和 @Tutorial_VERSION_MINOR@ 替换为 CMakeLIsts.txt 中的对应值，并生成 TutorialConfig.h。接下来需要修改 tutorial.cxx 文件，让它包含新生成的头文件，代码如下:

```cpp
// A simple program that computes the square root of a number
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include "TutorialConfig.h"
 
int main (int argc, char *argv[]){
    if (argc < 2){
        fprintf(stdout,"%s Version %d.%d\n", argv[0],
                Tutorial_VERSION_MAJOR,
                Tutorial_VERSION_MINOR);
        fprintf(stdout,"Usage: %s number\n", argv[0]);
        return 1;
    }
    double inputValue = atof(argv[1]);
    double outputValue = sqrt(inputValue);
    fprintf(stdout,"The square root of %g is %g\n", inputValue, outputValue);
    return 0;
}
```

The main changes are the inclusion of the TutorialConfig.h header file and printing out a version number as part of the usage message.

主要的修改为在源码中包含了 TutorialConfig.h 头文件，并且在使用说明中打印了版本号。

## Step 2 Adding a Library 第二步 添加链接库

Now we will add a library to our project. This library will contain our own implementation for computing the square root of a number. The executable can then use this library instead of the standard square root function provided by the compiler. For this tutorial we will put the library into a subdirectory called MathFunctions. It will have the following one line CMakeLists.txt file:

现在要为项目添加链接库。该链接库包含了计算平方根的函数。可执行文件可以使用该链接库中的函数替换编译器提供的标准函数。在"第二步"中，计划将生成的链接库 MathFunctions 放入到子目录中。子目录需要有如下的 CMakeLists.txt 文件:

```
add_library(MathFunctions mysqrt.cxx)
```

The source file mysqrt.cxx has one function called mysqrt that provides similar functionality to the compiler’s sqrt function. To make use of the new library we add an add_subdirectory call in the top level CMakeLists.txt file so that the library will get built. We also add another include directory so that the MathFunctions/MathFunctions.h header file can be found for the function prototype. The last change is to add the new library to the executable. The last few lines of the top level CMakeLists.txt file now look like:

子目录中的源文件 mysqrt.cxx 中有 mysqrt 函数，该函数与编译器自带的 sqrt 函数提供相同的功能。