# cmake-GenerateFindModule

A CMake script that automatically generates `Find<package>.cmake` files (a.k.a find module) for dependencies. Now we can create [relocatable](https://cmake.org/cmake/help/v3.5/manual/cmake-packages.7.html?highlight=packages#creating-relocatable-packages) CMake packages by avoiding absolute paths for dependencies (introduced by `find_library` and `find_path`).

## The Problem:

If our CMake project `libFoo` needs the library `libBar` as a dependency but `libBar` doesn't provide a `BarConfig.cmake` file (which exports the `Bar` target) we would write something like:

```
# find_package(Bar) Doesn't work because of missing BarConfig.cmake
find_library(Bar_LIBRARY Bar) # BAD: Bar_LIBRARY contains an absolute path to libBar.so
find_path(Bar_INCLUDE_DIR bar.h) # BAD: Bar_INCLUDE_DIR contains an absolute path

add_library(Foo foo.cpp)
target_link_libraries(Foo PUBLIC ${Bar_LIBRARY})
target_include_directories(Foo PUBLIC ${Bar_INCLUDE_DIR})

install(TARGETS Foo EXPORT FooTarget LIBRARY DESTINATION lib PUBLIC_HEADER DESTINATION include)

# BAD: This package is NOT relocatable because it 
# contains the absolute paths Bar_LIBRARY, Bar_INCLUDE_DIR
install(EXPORT FooTarget FILE FooConfig.cmake DESTINATION lib/cmake)
```
The dependency `libBar` is marked as `PUBLIC`. That means that other projects that directly depend on `libFoo` not only have to link `libFoo.so` but also `libBar.so`. The problem arises if we want to expor the target `Foo` by calling: `install(EXPORT FooTarget FILE FooConfig.cmake DESTINATION lib/cmake)`.
The exported target (in file `FooConfig.cmake`) will now contain both absolute paths: `Bar_LIBRARY` and `Bar_INCLUDE_DIR`. Thus this CMake package isn't relocatable any more (see [CMake doc](https://cmake.org/cmake/help/v3.5/manual/cmake-packages.7.html?highlight=packages#creating-relocatable-packages)).

## The Solution:

We have to write a `FindBar.cmake` file that generates the [importet](https://cmake.org/cmake/help/v3.5/command/add_library.html?highlight=imported#imported-libraries) target `Bar` so we can use `find_package` instead of `find_library` and `find_path`. Now we can remove `Bar_LIBRARY` and `Bar_INCLUDE_DIR` and link against the imported target like so: `target_link_libraries(Foo PUBLIC Bar)`. 

The process of writing `FindXXX.cmake` files can be very annoying because most of the time the find file mostly contains boilerplate code. With this script a find file is automatically generated for an arbitrary library that can be directly used via `find_package`.

The example from above would now look like this:
```
include(GenerateFindModule)
generate_find_module(FIND_FILE_BAR_PATH Bar LIB_NAMES bar HEADER_NAMES bar.h)
find_package(Bar) # GOOD: We can use the imported target Bar::Bar now

add_library(Foo foo.cpp)
target_link_libraries(Foo PUBLIC Bar::Bar)
# target_include_directories() is no longer needed

install(TARGETS Foo EXPORT FooTarget LIBRARY DESTINATION lib PUBLIC_HEADER DESTINATION include)

install(EXPORT FooTarget FILE FooTarget.cmake DESTINATION lib/cmake)

# We have to write a short config file that reuses the find module
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/FooConfig.cmake" 
"include(CMakeFindDependencyMacro)
set(CMAKE_MODULE_PATH \"\${CMAKE_CURRENT_LIST_DIR};\${CMAKE_MODULE_PATH}\")
find_dependency(Bar)
include(\"\${CMAKE_CURRENT_LIST_DIR}/FooTarget.cmake\")")

# We have to install the find moduel plus the config file
install(FILES "${FIND_FILE_BAR_PATH}" DESTINATION lib/cmake)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/FooConfig.cmake" DESTINATION lib/cmake)
```

## Use:

```
generate_find_module(
   <VAR>
   <package>
   LIB_NAMES name1 [name2 ...]
   HEADER_NAMES name1 [name2 ...]
   [LIB_HINTS path1 [path2 ... ENV var]]
   [HEADER_HINTS path1 [path2 ... ENV var]]
   [LIB_PATHS path1 [path2 ... ENV var]]
   [HEADER_PATHS path1 [path2 ... ENV var]]
   [LIB_PATH_SUFFIXES suffix1 [suffix2 ...]]
   [HEADER_PATH_SUFFIXES suffix1 [suffix2 ...]]
   [LIB_DOC "cache documentation string"]
   [HEADER_DOC "cache documentation string"]
   [NO_DEFAULT_PATH]
   [NO_CMAKE_ENVIRONMENT_PATH]
   [NO_CMAKE_PATH]
   [NO_SYSTEM_ENVIRONMENT_PATH]
   [NO_CMAKE_SYSTEM_PATH]
   [CMAKE_FIND_ROOT_PATH_BOTH | ONLY_CMAKE_FIND_ROOT_PATH | NO_CMAKE_FIND_ROOT_PATH]
)
```

`<VAR>` The given variable will contain the absolute path of the find module

`<package>` The name of the generated package. If a pkgconfig package with the same name exists it will be used. The generated target will have the name: `<package>::<package>`.

The following options prefixed by `LIB_` match the options of [find_library](https://cmake.org/cmake/help/v3.5/command/find_library.html?highlight=find_library#find-library). The options prefixed by `HEADER_`  match the options of [find_path](https://cmake.org/cmake/help/v3.5/command/find_path.html?highlight=find_path#find-path). This also applies to options without prefix.

To debug the find module set the CMake cache entry `VERBOSE_FIND_MODULE` to `ON`.

### Example:

Generate a find module for glib-2.0. Will use `glib-2.0.pc` file if found.

`generate_find_module(FIND_FILE_GLIB_PATH glib-2.0 LIB_NAMES glib-2.0 HEADER_NAMES glib.h HEADER_PATH_SUFFIXES glib)`
