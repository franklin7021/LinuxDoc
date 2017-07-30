# cmake tutorial

翻译自 [cmake-tutorial](https://cmake.org/cmake-tutorial/)

本文采用英中对照的方式予以展示，部分翻译采用意译的方式呈现。

译者 zhangmc 邮箱 zhangmc1993@qq.com

Below is a step-by-step tutorial covering common build system issues that CMake helps to address. Many of these topics have been introduced in [Mastering CMake](https://www.kitware.com/what-we-offer/#books) as separate issues but seeing how they all work together in an example project can be very helpful. This tutorial can be found in the [Tests/Tutorial](https://gitlab.kitware.com/cmake/cmake/tree/master/Tests/Tutorial) directory of the CMake source code tree. Each step has its own subdirectory containing a complete copy of the tutorial for that step

如下教程将一步步指引如何使用 CMake 解决通用的编译链接问题。其中很多问题已经在 [Mastering CMake](https://www.kitware.com/what-we-offer/#books) 作为独立部分予以讨论，但是他们如何在一个样例中协同工作是本文的重点。本教程的源码可以在 [Tests/Tutorial](https://gitlab.kitware.com/cmake/cmake/tree/master/Tests/Tutorial) 中找到，如下的每一步都对应着代码库中的一个子文件夹。

## Step 1 A Basic Starting Point 第一步 开始

The most basic project is an executable built from source code files. For simple projects a two line CMakeLists.txt file is all that is required. This will be the starting point for our tutorial. The CMakeLists.txt file looks like:

最基础的项目就是使用源文件编译链接生成可执行文件。简单项目的 CMakeLists.txt 文件只需要如下几行，教程也从简单项目开始。CMakeLists.txt 如下所示:

```cmake
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

```cmake
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

现在要为项目添加链接库。该链接库包含了计算平方根的函数。可执行文件可以使用该链接库中的函数替换编译器提供的标准函数。在"第二步"中，计划将生成的链接库放入到子目录 MathFunctions 中。子目录需要有如下的 CMakeLists.txt 文件:

```cmake
add_library(MathFunctions mysqrt.cxx)
```

The source file mysqrt.cxx has one function called mysqrt that provides similar functionality to the compiler’s sqrt function. To make use of the new library we add an add_subdirectory call in the top level CMakeLists.txt file so that the library will get built. We also add another include directory so that the MathFunctions/MathFunctions.h header file can be found for the function prototype. The last change is to add the new library to the executable. The last few lines of the top level CMakeLists.txt file now look like:

子目录中的源文件 mysqrt.cxx 中有 mysqrt 函数，该函数与编译器自带的 sqrt 函数提供相同的功能。在顶层的 CMakeLists.txt 中添加 add_subdirectory，那么子目录就会得到编译；也要添加头文件目录，目的是找到 MathFunctions/MathFunctions.h 头文件中的函数原型。最后还需要想可执行文件添加链接可。顶层的 CMakeLists.txt 如下所示:

```cmake
include_directories ("${PROJECT_SOURCE_DIR}/MathFunctions")
add_subdirectory (MathFunctions) 
 
# add the executable
# 生成可执行文件
add_executable (Tutorial tutorial.cxx)
target_link_libraries (Tutorial MathFunctions)
```

Now let us consider making the MathFunctions library optional. In this tutorial there really isn’t any reason to do so, but with larger libraries or libraries that rely on third party code you might want to. The first step is to add an option to the top level CMakeLists.txt file.

如何让链接 MathFunctions 库变成可选项?在当前教程中看似没有必要，但是当项目使用依赖第三方代码的大型链接库时，将选用链接库变成可选项就很有必要了。首先在顶层 CMakeLists.txt 中添加如下代码:

```cmake
# should we use our own math functions?
# 使用自己的函数?
option (USE_MYMATH 
        "Use tutorial provided math implementation" ON) 
```

This will show up in the CMake GUI with a default value of ON that the user can change as desired. This setting will be stored in the cache so that the user does not need to keep setting it each time they run CMake on this project. The next change is to make the build and linking of the MathFunctions library conditional. To do this we change the end of the top level CMakeLists.txt file to look like the following:

在 CMake GUI 中以默认值 "on" 的形式呈现，用户可以随意更改。这个设置会保留在缓存中，不需要每次运行时重新设置。下一步就是依照条件选项编译链接 MathFunctions 库。在顶层 CMakeLists.txt 的末尾作如下改动:

```cmake
# add the MathFunctions library?
# 是否使用 MathFunctions 库?
if (USE_MYMATH)
  include_directories ("${PROJECT_SOURCE_DIR}/MathFunctions")
  add_subdirectory (MathFunctions)
  set (EXTRA_LIBS ${EXTRA_LIBS} MathFunctions)
endif (USE_MYMATH)
 
# add the executable
# 生成可执行文件
add_executable (Tutorial tutorial.cxx)
target_link_libraries (Tutorial  ${EXTRA_LIBS})
```

This uses the setting of USE_MYMATH to determine if the MathFunctions should be compiled and used. Note the use of a variable (EXTRA_LIBS in this case) to collect up any optional libraries to later be linked into the executable. This is a common approach used to keep larger projects with many optional components clean. The corresponding changes to the source code are fairly straight forward and leave us with:

USE_MYMATH 决定 MathFunctions 库是否编译并且使用。变量 EXTRA_LIBS 是选择使用的链接库的集合，最后会被链接入可执行文件。使用 EXTRA_LIBS 是大型项目链接的常用方式。源代码也需要做相应的修改:

```cpp
/*
A simple program that computes the square root of a number
计算平方跟的程序
*/
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include "TutorialConfig.h"

#ifdef USE_MYMATH
#include "MathFunctions.h"
#endif
 
int main (int argc, char *argv[]){
    if (argc < 2){
        fprintf(stdout,"%s Version %d.%d\n", argv[0],
            Tutorial_VERSION_MAJOR,
            Tutorial_VERSION_MINOR);
        fprintf(stdout,"Usage: %s number\n",argv[0]);
        return 1;
    }
    double inputValue = atof(argv[1]);
 
#ifdef USE_MYMATH
    double outputValue = mysqrt(inputValue);
#else
    double outputValue = sqrt(inputValue);
#endif
 
    fprintf(stdout,"The square root of %g is %g\n",
        inputValue, outputValue);
    return 0;
}
```

In the source code we make use of USE_MYMATH as well. This is provided from CMake to the source code through the TutorialConfig.h.in configured file by adding the following line to it:

源代码中也使用到了 USE_MYMATH。可以通过编辑配置文件 TutorialConfig.h.in 将 USE_MYMATH 从 CMake 传递给源代码:

```
#cmakedefine USE_MYMATH
```

## Step 3 Installing and Testing 第三步 安装与测试

For the next step we will add install rules and testing support to our project. The install rules are fairly straight forward. For the MathFunctions library we setup the library and the header file to be installed by adding the following two lines to MathFunctions’ CMakeLists.txt file:

下一步是添加安装以及测试规则。安装规则非常直接，对于 MathFunctions 库来说，需要安装链接库和头文件。在 MathFunctions 中的 CMakeLists.txt 添加如下两行:

```cmake
install (TARGETS MathFunctions DESTINATION bin)
install (FILES MathFunctions.h DESTINATION include)
```

For the application the following lines are added to the top level CMakeLists.txt file to install the executable and the configured header file:

对于应用程序来说，需要在顶层的 CMakeLists.txt 中添加可执行文件以及头文件的安装命令:

```cmake
# add the install targets
# 添加安装目标
install (TARGETS Tutorial DESTINATION bin)
install (FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"  
         DESTINATION include)
```

That is all there is to it. At this point you should be able to build the tutorial, then type make install (or build the INSTALL target from an IDE) and it will install the appropriate header files, libraries, and executables. The CMake variable CMAKE_INSTALL_PREFIX is used to determine the root of where the files will be installed. Adding testing is also a fairly straight forward process. At the end of the top level CMakeLists.txt file we can add a number of basic tests to verify that the application is working correctly.

安装部分就全部介绍完了。使用 `make install` 命令(或使用 IDE 的安装功能)安装头文件、链接库和可执行文件。CMake 中的变量 CMAKE_INSTALL_PREFIX 用来决定安装路径。添加测试也非常简明。在顶层 CMakeLists.txt 的末尾添加基础测试，以验证应用可以正常工作。

```cmake
include(CTest)

# does the application run
# 应用能否使用
add_test (TutorialRuns Tutorial 25)

# does it sqrt of 25
# 测试计算25的平方根
add_test (TutorialComp25 Tutorial 25)
set_tests_properties (TutorialComp25 PROPERTIES PASS_REGULAR_EXPRESSION "25 is 5")

# does it handle negative numbers
# 处理负数
add_test (TutorialNegative Tutorial -25)
set_tests_properties (TutorialNegative PROPERTIES PASS_REGULAR_EXPRESSION "-25 is 0")

# does it handle small numbers
# 数值较小时的测试
add_test (TutorialSmall Tutorial 0.0001)
set_tests_properties (TutorialSmall PROPERTIES PASS_REGULAR_EXPRESSION "0.0001 is 0.01")

# does the usage message work?
# 使用说明测试
add_test (TutorialUsage Tutorial)
set_tests_properties (TutorialUsage PROPERTIES PASS_REGULAR_EXPRESSION "Usage:.*number")
```

After building one may run the “ctest” command line tool to run the tests. The first test simply verifies that the application runs, does not segfault or otherwise crash, and has a zero return value. This is the basic form of a CTest test. The next few tests all make use of the PASS_REGULAR_EXPRESSION test property to verify that the output of the test contains certain strings. In this case verifying that the computed square root is what it should be and that the usage message is printed when an incorrect number of arguments are provided. If you wanted to add a lot of tests to test different input values you might consider creating a macro like the following:

在编译连接后，使用命令行工具 `ctest` 运行测试。第一项测试是最为基础的形式，检验程序能否执行，是否有段错误或者是否有其他原因导致的程序崩溃，返回值为0。后续几个测试使用 PASS_REGULAR_EXPRESSION 属性验证输出是否包含特定字符；这就可以验证计算结果是否正确，使用说明是否正确打印。如需要添加很多测试，可以创建如下宏:

```cmake
# define a macro to simplify adding tests, then use it
macro (do_test arg result)
# 定义宏简化测试
  add_test (TutorialComp${arg} Tutorial ${arg})
  set_tests_properties (TutorialComp${arg}
    PROPERTIES PASS_REGULAR_EXPRESSION ${result})
endmacro (do_test)
 
# do a bunch of result based tests
# 运行测试验证结果
do_test (25 "25 is 5")
do_test (-25 "-25 is 0")
```

For each invocation of do_test, another test is added to the project with a name, input, and results based on the passed arguments.

每次调用 do_test，后续的测试就会被添加，参数为输入以及期待的结果。