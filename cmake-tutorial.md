# cmake tutorial

翻译自 [cmake-tutorial](https://cmake.org/cmake-tutorial/)

本文采用英中对照的方式予以展示，部分翻译采用意译的方式呈现。

译者 zhangmc 邮箱 zhangmc1993@qq.com

Below is a step-by-step tutorial covering common build system issues that CMake helps to address. Many of these topics have been introduced in [Mastering CMake](https://www.kitware.com/what-we-offer/#books) as separate issues but seeing how they all work together in an example project can be very helpful. This tutorial can be found in the [Tests/Tutorial](https://gitlab.kitware.com/cmake/cmake/tree/master/Tests/Tutorial) directory of the CMake source code tree. Each step has its own subdirectory containing a complete copy of the tutorial for that step

如下教程将一步步指引如何使用 CMake 解决通用的编译链接问题。其中很多问题已经在 [Mastering CMake](https://www.kitware.com/what-we-offer/#books) 作为独立部分予以讨论，但是他们如何在一个样例中协同工作是本文的重点。本教程的源码可以在 [Tests/Tutorial](https://gitlab.kitware.com/cmake/cmake/tree/master/Tests/Tutorial) 中找到，如下的每一步都对应着代码库中的一个子文件夹。

## Step 1 A Basic Starting Point 第一节 开始

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

## Step 2 Adding a Library 第二节 添加链接库

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

## Step 3 Installing and Testing 第三节 安装与测试

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

## Step 4 Adding System Introspection 第四节 添加系统自检

Next let us consider adding some code to our project that depends on features the target platform may not have. For this example we will add some code that depends on whether or not the target platform has the log and exp functions. Of course almost every platform has these functions but for this tutorial assume that they are less common. If the platform has log then we will use that to compute the square root in the mysqrt function. We first test for the availability of these functions using the CheckFunctionExists.cmake macro in the top level CMakeLists.txt file as follows:

如果目标平台没有某种特性，那么就需要在项目代码中做出相应的修改。比如，如果目标平台没有 log 和 exp 函数，那么就需要在代码添加相关功能。当然几乎所有的平台支持这些函数，但是在教程中假设他们缺少这些函数。如果当前平台有 log 和 exp 函数，那么就将其应用在平方根的计算之中。在顶层 CMakeLists.txt 中使用 CheckFunctionExists.cmake 宏检测系统中是否提供这些函数:

```cmake
# does this system provide the log and exp functions?
# 系统是否提供 log 和 exp 函数
include (CheckFunctionExists)
check_function_exists (log HAVE_LOG)
check_function_exists (exp HAVE_EXP)
```

Next we modify the TutorialConfig.h.in to define those values if CMake found them on the platform as follows:

接下来修改 TutorialConfig.h.in 文件。如果 CMake 在系统中找到了对应的函数，就在 TutorialConfig.h 中添加这些宏定义:

```
// does the platform provide exp and log functions?
// 平台是否提供 log 和 exp 函数
#cmakedefine HAVE_LOG
#cmakedefine HAVE_EXP
```

It is important that the tests for log and exp are done before the configure_file command for TutorialConfig.h. The configure_file command immediately configures the file using the current settings in CMake. Finally in the mysqrt function we can provide an alternate implementation based on log and exp if they are available on the system using the following code:

CMakeLists.txt 中的 configure_file 命令使用当前设置配置 TutorialConfig.h 文件，所以 check_function_exists 应该写在 configure_file 之前。最后在 mysqrt 函数中添加如下代码，基于 log 和 exp 函数提供计算平方根的功能。

```cpp
// if we have both log and exp then use them
// 如果系统提供 log 和 exp 函数，使用他们计算平方根
#if defined (HAVE_LOG) && defined (HAVE_EXP)
    result = exp(log(x)*0.5);

// otherwise use an iterative approach
// 不提供则使用其他替代方式
#else 
    . . .
```

## Step 5 Adding a Generated File and Generator 第五节 添加程序生成的文件与生成器

In this section we will show how you can add a generated source file into the build process of an application. For this example we will create a table of precomputed square roots as part of the build process, and then compile that table into our application. To accomplish this we first need a program that will generate the table. In the MathFunctions subdirectory a new source file named MakeTable.cxx will do just that.

本节讲述如何在项目中添加程序生成的源文件。比如，在编译过程中创建预先计算好的平方根列表，并将该列表加入项目编译。为实现这个特性，首先编写可以生成列表的程序。在 MathFunctions 子目录中添加新的源文件 MakeTable.cxx:

```cpp
// A simple program that builds a sqrt table 
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
 
int main (int argc, char *argv[]){
    int i;
    double result;

    // make sure we have enough arguments
    // 检查参数
    if (argc < 2){
        return 1;
    }
  
    // open the output file
    // 打开文件
    FILE *fout = fopen(argv[1],"w");
    if (!fout){
        return 1;
    }
  
    // create a source file with a table of square roots
    // 在文件中写入列表
    fprintf(fout,"double sqrtTable[] = {\n");
    for (i = 0; i < 10; ++i){
        result = sqrt(static_cast<double>(i));
        fprintf(fout,"%g,\n",result);
    }
 
    // close the table with a zero
    // 列表以0结尾，关闭文件
    fprintf(fout,"0};\n");
    fclose(fout);
    return 0;
}
```

Note that the table is produced as valid C++ code and that the name of the file to write the output to is passed in as an argument. The next step is to add the appropriate commands to MathFunctions’ CMakeLists.txt file to build the MakeTable executable, and then run it as part of the build process. A few commands are needed to accomplish this, as shown below.

需要注意的是，列表由 C++ 代码生成，该程序的输入参数是列表输出的文件名。下一步就是在 MathFunctions 文件夹的 CMakeLists.txt 中添加适当的命令编译并执行 MakeTable。如下所示，需要添加几行命令:

```cmake
# first we add the executable that generates the table
# 生成可执行文件
add_executable(MakeTable MakeTable.cxx)
 
# add the command to generate the source code
# 生成源代码
add_custom_command (
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
  COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
  DEPENDS MakeTable
  )
 
# add the binary tree directory to the search path for 
# include files
# 添加头文件搜索路径
include_directories( ${CMAKE_CURRENT_BINARY_DIR} )
 
# add the main library
# 添加链接库
add_library(MathFunctions mysqrt.cxx ${CMAKE_CURRENT_BINARY_DIR}/Table.h  )
```

First the executable for MakeTable is added as any other executable would be added. Then we add a custom command that specifies how to produce Table.h by running MakeTable. Next we have to let CMake know that mysqrt.cxx depends on the generated file Table.h. This is done by adding the generated Table.h to the list of sources for the library MathFunctions. We also have to add the current binary directory to the list of include directories so that Table.h can be found and included by mysqrt.cxx. When this project is built it will first build the MakeTable executable. It will then run MakeTable to produce Table.h. Finally, it will compile mysqrt.cxx which includes Table.h to produce the MathFunctions library. At this point the top level CMakeLists.txt file with all the features we have added looks like the following:

第一条命令将 MakeTable.cxx 编译成可执行文件 MakeTable。第二条命令运行可执行文件生成 Table.h。在 mysqrt.cxx 中包含新生成的头文件，将当前文件夹加入到头文件搜索目录中。当项目整体编译时，先生成可执行文件 MakeTable，再执行 MakeTable 生成 Table.h，最后编译 mysqrt.cxx 生成链接库。顶层的 CMakeLists.txt 文件如下:

```cmake
cmake_minimum_required (VERSION 2.6)
project (Tutorial)
include(CTest)
 
# The version number.
# 版本号
set (Tutorial_VERSION_MAJOR 1)
set (Tutorial_VERSION_MINOR 0)
 
# does this system provide the log and exp functions?
# 系统是否提供 log 和 exp 函数
include (${CMAKE_ROOT}/Modules/CheckFunctionExists.cmake)
 
check_function_exists (log HAVE_LOG)
check_function_exists (exp HAVE_EXP)
 
# should we use our own math functions
# 是否使用自己编写的函数
option(USE_MYMATH 
  "Use tutorial provided math implementation" ON)
 
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
include_directories ("${PROJECT_BINARY_DIR}")
 
# add the MathFunctions library?
# 是否添加 MathFunctions 链接库
if (USE_MYMATH)
  include_directories ("${PROJECT_SOURCE_DIR}/MathFunctions")
  add_subdirectory (MathFunctions)
  set (EXTRA_LIBS ${EXTRA_LIBS} MathFunctions)
endif (USE_MYMATH)
 
# add the executable
# 生成可执行文件
add_executable (Tutorial tutorial.cxx)
target_link_libraries (Tutorial  ${EXTRA_LIBS})
 
# add the install targets
# 添加安装目标
install (TARGETS Tutorial DESTINATION bin)
install (FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"        
         DESTINATION include)
 
# does the application run
# 程序能否运行
add_test (TutorialRuns Tutorial 25)
 
# does the usage message work?
# 使用说明是否正确打印
add_test (TutorialUsage Tutorial)
set_tests_properties (TutorialUsage
  PROPERTIES 
  PASS_REGULAR_EXPRESSION "Usage:.*number"
  )
 
 
# define a macro to simplify adding tests
# 定义宏简化测试
macro (do_test arg result)
  add_test (TutorialComp${arg} Tutorial ${arg})
  set_tests_properties (TutorialComp${arg}
    PROPERTIES PASS_REGULAR_EXPRESSION ${result}
    )
endmacro (do_test)
 
# do a bunch of result based tests
# 运行测试验证结果
do_test (4 "4 is 2")
do_test (9 "9 is 3")
do_test (5 "5 is 2.236")
do_test (7 "7 is 2.645")
do_test (25 "25 is 5")
do_test (-25 "-25 is 0")
do_test (0.0001 "0.0001 is 0.01")
```

TutorialConfig.h.in looks like:

TutorialConfig.h.in 文件:

```
// the configured options and settings for Tutorial
// 对应 CMakeLists.txt 中的选项和配置
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
#cmakedefine USE_MYMATH
 
// does the platform provide exp and log functions?
// 平台是否提供 log 和 exp 函数
#cmakedefine HAVE_LOG
#cmakedefine HAVE_EXP
```

And the CMakeLists.txt file for MathFunctions looks like:

MathFunctions 目录中 CMakeLists.txt 文件:

```cmake
# first we add the executable that generates the table
# 生成可执行文件
add_executable(MakeTable MakeTable.cxx)
# add the command to generate the source code
# 生成源代码
add_custom_command (
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
  DEPENDS MakeTable
  COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
  )
# add the binary tree directory to the search path 
# for include files
# 添加头文件搜索路径
include_directories( ${CMAKE_CURRENT_BINARY_DIR} )
 
# add the main library
# 添加链接库
add_library(MathFunctions mysqrt.cxx ${CMAKE_CURRENT_BINARY_DIR}/Table.h)
 
install (TARGETS MathFunctions DESTINATION bin)
install (FILES MathFunctions.h DESTINATION include)
```

## Step 6 Building an Installer 第六节 打包

Next suppose that we want to distribute our project to other people so that they can use it. We want to provide both binary and source distributions on a variety of platforms. This is a little different from the install we did previously in section Installing and Testing (Step 3), where we were installing the binaries that we had built from the source code. In this example we will be building installation packages that support binary installations and package management features as found in cygwin, debian, RPMs etc. To accomplish this we will use CPack to create platform specific installers as described in Chapter Packaging with CPack. Specifically we need to add a few lines to the bottom of our toplevel CMakeLists.txt file.

接下来就是面向其他用户发行项目，可以提供二进制包以及源码包的形式支持不同的平台。这与在第三节中介绍的安装有些许区别。第三节中安装的二进制文件是由源代码编译而来；此节将要介绍的是如何生成二进制安装包以及如何让安装包支持包管理系统，如 cygwin、debian、RPM 等。为达成此目标需要使用 CPack 创建特定的安装器；需要在顶层的 CMakeLists.txt 文件的末尾加入如下几行:

```cmake
# build a CPack driven installer package
# 使用 CPack 生成安装包
include (InstallRequiredSystemLibraries)
set (CPACK_RESOURCE_FILE_LICENSE  
     "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
set (CPACK_PACKAGE_VERSION_MAJOR "${Tutorial_VERSION_MAJOR}")
set (CPACK_PACKAGE_VERSION_MINOR "${Tutorial_VERSION_MINOR}")
include (CPack)
```

That is all there is to it. We start by including InstallRequiredSystemLibraries. This module will include any runtime libraries that are needed by the project for the current platform. Next we set some CPack variables to where we have stored the license and version information for this project. The version information makes use of the variables we set earlier in this tutorial. Finally we include the CPack module which will use these variables and some other properties of the system you are on to setup an installer.

首先加入安装软件所需的系统链接库 InstallRequiredSystemLibraries。该模块包含了当前平台软件运行所需的所有运行库。接着设置 CPack 变量，比如项目的许可证以及版本信息。最后将 CPack 模块包含进来，CPack 模块使用上述变量以及其他特性生成安装器。

The next step is to build the project in the usual manner and then run CPack on it. To build a binary distribution you would run:

下一步编译整个项目。创建二进制安装包，执行如下命令:

```
cpack --config CPackConfig.cmake
```

To create a source distribution you would type:

创建源码包，执行如下命令:

```
cpack --config CPackSourceConfig.cmake
```

## Step 7 Adding Support for a Dashboard 第七节 添加对 Dashboard 的支持

Adding support for submitting our test results to a dashboard is very easy. We already defined a number of tests for our project in the earlier steps of this tutorial. We just have to run those tests and submit them to a dashboard. To include support for dashboards we include the CTest module in our toplevel CMakeLists.txt file.

向 dashboard 提交测试结果非常简单。在前几节中已经为项目添加了测试，只需执行测试，并将测试结果提交到 dashboard 上即可。为此，需要在顶层 CMakeLists.txt 中添加 CTest 模块:

```cmake
# enable dashboard scripting
# 启用 dashboard
include (CTest)
```

We also create a CTestConfig.cmake file where we can specify the name of this project for the dashboard.

还需创建 CTestConfig.cmake 文件，在文件中明确 dashboard 上的项目名称。

```cmake
set (CTEST_PROJECT_NAME "Tutorial")
```

CTest will read in this file when it runs. To create a simple dashboard you can run CMake on your project, change directory to the binary tree, and then run ctest –D Experimental. The results of your dashboard will be uploaded to Kitware’s public dashboard [here](https://open.cdash.org/index.php?project=PublicDashboard).

CTest 在运行时会查看 CTestConfig.cmake 文件。生成 dashboard，需要运行 CMake，然后切换到编译路径执行 ctest –D Experimental。执行结果会被上传到[公共的 dashboard](https://open.cdash.org/index.php?project=PublicDashboard)。