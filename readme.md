# Notes*

*From "Jamie's CMake Training" videos on YouTube

## Intro

- Build System: a tool or set of tools used to compile and link source code (e.g. MSBuild, Make)
- Build system generator: a tool or set of tools used to generate project files for the specified build system(s) (e.g. CMake)

## CMake

- CMake: leading generator (CMake network effect is the fact that it is widely supported)
- Avoid reimplementing functionalities that CMake can handle
- Use CMake macros to make compilation platform-independent

## Common compile options

option | function | variants
------ | -------- | --------
-Wall | Enables all standard warnings, so the compiler will tell you about code it thinks is suspicious | -W<name> and -Wno-<name> allow you to enable or disable specific warnings
--std=c++11 | Tells the compiler to use C++11 | Can also use newer or older C++ standards such as C++98 and C++17. C++11 is the most recent standard available across all systems
-g | Generates debugging information so that you can view stack-traces in your code with GDB and Valgrind | -g3 generates even more debugging info for more detailed views, at the cost of making your code larger
-O0 | Disables all optimizations, which allows you to step through code normally in the debugger | -Og only enables optimizations which do not affect debugging.
-O2 | Enables most optimizations, makes code run faster | O3: optimize for speed at the cost of size, Os: optimize for size at the cost of speed
-I<folder> | Adds a directory to the header search path | -include includes a specific header file at the top of each cpp file
-o <name> | Specifies the name of the output file | If note given, by convention the result is names "a.out"
-D<var> | Defines a preprocessor definition with the given name | -D<var>=<value> gives the definition a specific value


## CMake directory structure

- Helps keep your git repos nice and clean
- Three directories to remember:
    1. Source dir: where your source code is
    2. Binary dir: where the binaries are stored
    3. Install dir: where the binaries will be install when you run `make install`
- When running CMake, you always run in the binary dir and pass CMake the path to the source dir


## What is CMakeLists.txt?

- Top level buildfile that performs some important tasks:
    - Initializing CMake
    - Setting compilation options
    - Inspecting the system and finding libraries
- The buildfiles in each directory take care of building the code there
    - Setting local compile options
    - Building code into various targets
    - Linking targets to the libraries they depend on


## Parts of CMakeLists.txt

1. Initialization code

```s
cmake_minimum_required(VERSION 3.5)
project(TestProject LANGUAGES CXX)
```

- These two lines are required in any CMake builds
- C++ is called `CXX` in CMake

2. Source list

```s
set(TEST_PROJ_SOURCES
    helper/helper.cpp
    main.cpp)
```

3. Compile options

```
include_directories(helper)
add_compile_options(--std=c++11 -Wall)
```

- These two are directory scope functions. It means, they affect the current directory and its subdirectories

4. Create executable

```
add_executable(test_project ${TEST_PROJ_SOURCES})
```

## What is a target?

A target is anything that CMake can build:
- Executable
- Static library
- Shared library
- Object library
- Custom target

The entire purpose of a build system, its reason for existence, is to create targets. 

## Targets

- Executable
    - Created by `add_executable()`
    - Create programs that can be run (.exe in Windows)
- Static library
    - Created by `add_library(STATIC)`
    - Create static libraries of code that can be linked into executables
- Object library
    - Created by `add_library(OBJECT)`
    - Create object libraries: code compiled into .o files but not combined into a single library
    - Very similar to static libraries
    - Most common use is to build a common piece of code just once and link it to multiple targets that need it.
    - Used by experienced developers for complicated builds.
- Dynamic library
    - Created by `add_library()`
    - Create dynamic libraries of code that can be used by executables

## Properties

You can further configure the build by setting properties. The most common ones are:

property | scope | function
-------- | ----- | --------
INCLUDE_DIRECTORIES | Directory, target, source | List of directories to add to the include path
COMPILE_DEFINITIONS | Directory, target, source | List of processor macros to define when compiling the code
COMPILE_OPTIONS | Directory, target, source | List of compiler flags to use when compiling the code
POSITION_INDEPENDENT_CODE | Target | Controls whether the code will be build as position independent, which is required when compiling shared libraries.

## Linking targets

- Libraries can be linked to a target using `target_link_libraries()`.
- This allows the target to reference code stored in the given libraries.
- Link dependencies are multi level: if you link A->B and B->C then CMake will automatically link A->C.
- Linking will also pull in the _interface options_ of the libraries being linked.
- Interface properties are used to carry dependencies between targets
- Why are interface properties are useful?
    - Let's say we have _foo.h_ which requires the flag `-I/usr/include/somelib`
    - And we have another target, _bar.h_ that includes _foo.h_
    - However, _foo.h_ is not aware that _bar.h_ requires a specific flag
    - We can pass that info using the _interface properties_
    - `set_property(TARGET foo PROPERTY INTERFACE_INCLUDE_DIRECTORIES /usr/include/somelib)`

## Cache variables

- CMake provides a way for you to pass command line options that affect the build system's behavior
- It is also desirable to store some build results to run faster for subsequent builds
- For CMake, both of these are achieved through the same mechanism: cache variables
- So, on one hand the user should be able to pass options using cache variables (they must be editable) on the other hand cache variables need to retain their values between invocations of CMake
- This causes some weirdness in using cache variables. For example, you can set a cache variable inside the CMakeLists.txt, but next time you run CMake, it doesn't update the cache variable if it is already exists. So, you may need to clean the cache for some changes to take affect.
- Viewing cache variables: Useful to understand what CMake did to the project.
    - Editing CMakeCache.txt
    - Using the ccmake tool
    - Using the cmake-gui tool
- Most users don't know these tools are exists, so developers are less interested making them more readable.

## Examples setting the cache variable

```
set(MY_BOOL_VAR TRUE CACHE BOOL "My custom boolean")
set(MY_PATHVAR "usr/local/" CACHE PATH "Path to my thing")
set(MY_INTERNAL_VAR "don't touch me" CACHE INTERNAL "Internal data")
```

## Cache Variable Overriding

Defined | Set with the same name | Result
------- | -----------------------| ------
local variable | local variable | The local variable's value changed
cache variable | cache variable w/o FORCE | The cache variable's original value is NOT changed
cache variable | cache variable w/ FORCE | The cache variable's original value is overwritten
cache variable | local variable | The cache variable is shadowed while the local variable is in scope
local variable | cache variable | If the cache variable was already in the cache, then the cache variable stays out of scope. However, if the cache variable did not exist, the local variable is deleted and the new cache variable is put into scope.

<b></b>

What will it print?

```
set(INSTALL_LOCATION "/usr")
set(INSTALL_LOCATION "/opt" CACHE STRING "Install location for stuff")
message(STATUS "Stuff will be installed to: ${INSTALL_LOCATION}")
```

The first time CMake run:

 -- Stuff will be installed to: /opt

Subsequent runs:

 -- Stuff will be installed to: /usr


## Build configurations

- Debug configurations: Optimization turned down to a low level or disabled, debug information generated.
- Release configuration: Optimization at full, no debug information.
- Use CMAKE_BUILD_TYPE cache variable to set.
- Four types: Debug, Release, RelWithDebInfo, MinSizeRel

## Global compile flags

- CMake reads the global flags for each language from the cache variable "CMAKE_<language>_FLAGS"
- CMAKE_<language>_FLAGS is used in CMake's testing of the compiler itself. So, if certain flags are required to make your compiler works, you need to provide them to CMAKE_<language>_FLAGS. For example, when compiling for embedded ARM, you must pass -mcpu=<your CPU>
- CMake also reads the flags for each configuration from the variable "CMAKE_<language>_FLAGS_<configuration>" (e.g. CMAKE_C_FLAGS_DEBUG).
- These are initialized to sensible defaults by CMake.
    - For example, for Release, CMAKE_C_FLAGS_RELEASE is initialized to "-O2 -DNDEBUG"
- You cans set these to your own values _after_ the `project()` command