S-CPM
===

A (Simplified) C++ Package Manager based on CMake and Git.

Forked from https://github.com/iauns/cpm

CPM is designed to promote small, well-tested, composable C++ modules.

SCPM is a relaxed CPM that doesn't manage versioning and namespaces,
reduced to be simply a downloader of projects in a complex hierarchy of
subprojects.

Basic usage is to add CPM to each non-leaf project that you have in your
hierarchy and list its dependancies.

**Table of Contents**

- [Brief Example](#brief-example)
- [Using CPM](#using-cpm)
  - [Quick Setup](#quick-setup)
  - [Things to Note](#things-to-note)
- [Building CPM Modules](#building-cpm-modules)
  - [CMakeLists.txt Entry](#cmakeliststxt-entry)
  - [Library target name](#library-target-name)
  - [Wrapping Namespace](#wrapping-namespace)
  - [Directory Structure](#directory-structure)
  - [Exporting Build Info](#exporting-build-info)
  - [Registering Your Module](#registering-your-module)
  - [Building Externals](#building-externals)
- [CPM Function Reference](#cpm-function-reference)
  - [General Purpose](#general-purpose)
  - [Modules Only](#modules-only)
- [Miscellaneous Issues and Questions](#miscellaneous-issues-and-questions)

Brief Example
=============

Below is a sample of a CMakeLists.txt file that uses 3 modules. These modules
are: platform specific OpenGL headers, an axis aligned bounding box
implementation, and G-truc's GLSL vector math library. See the next section
for a full explanation of how to use CPM. The following CMakeLists.txt is
self-contained and will build as-is:

```cmake
  cmake_minimum_required(VERSION 2.8.7 FATAL_ERROR)
  project(foo)

  #------------------------------------------------------------------------------
  # Required CPM Setup - no need to modify - See: https://github.com/iauns/cpm
  #------------------------------------------------------------------------------
  set(CPM_DIR "${CMAKE_CURRENT_BINARY_DIR}/cpm_packages" CACHE TYPE STRING)
  find_package(Git)
  if(NOT GIT_FOUND)
    message(FATAL_ERROR "CPM requires Git.")
  endif()
  if (NOT EXISTS ${CPM_DIR}/CPM.cmake)
    message(STATUS "Cloning repo (https://github.com/morpheby/scpm)")
    execute_process(
      COMMAND "${GIT_EXECUTABLE}" clone https://github.com/morpheby/scpm ${CPM_DIR}
      RESULT_VARIABLE error_code
      OUTPUT_QUIET ERROR_QUIET)
    if(error_code)
      message(FATAL_ERROR "CPM failed to get the hash for HEAD")
    endif()
  endif()
  include(${CPM_DIR}/CPM.cmake)

  #------------------------------------------------------------------------------
  # CPM Modules
  #------------------------------------------------------------------------------

  # ++ MODULE: OpenGL platform
  CPM_AddModule(
    GIT_REPOSITORY "https://github.com/iauns/cpm-gl-platform"
    GIT_TAG "1.3.5")

  # ++ MODULE: aabb
  CPM_AddModule(
    GIT_REPOSITORY "https://github.com/iauns/cpm-glm-aabb"
    GIT_TAG "1.0.3")

  # ++ EXTERNAL-MODULE: GLM
  CPM_AddModule(
    GIT_REPOSITORY "https://github.com/iauns/cpm-glm"
    GIT_TAG "1.0.2"
    USE_EXISTING_VER TRUE)

  # If the project is the root project:
  CPM_Finish()
  # OR
  # If the project is not the root project:
  CPM_InitModule(my-project-name)

  #-----------------------------------------------------------------------
  # Setup source
  #-----------------------------------------------------------------------

  # NOTE: Feel free to ignore this section. It simply creates a main.cpp source
  # file from scratch instead of relying on the source being present. This is
  # done only to keep this CMakeLists.txt self-contained.

  file(WRITE src/main.cpp "#include <iostream>\n")
  file(APPEND src/main.cpp "#include <glm-aabb/AABB.hpp>\n")
  file(APPEND src/main.cpp "#include <glm/glm.hpp>\n\n")
  file(APPEND src/main.cpp "namespace glm_aabb = CPM_AABB_NS;\n\n")
  file(APPEND src/main.cpp "int main(int argc, char** av)\n")
  file(APPEND src/main.cpp "{\n")
  file(APPEND src/main.cpp "  glm_aabb::AABB aabb(glm::vec3(-1.0), glm::vec3(1.0));\n")
  file(APPEND src/main.cpp "  aabb.extend(glm::vec3(-2.0, 3.0, -0.5));\n")
  file(APPEND src/main.cpp "  glm_aabb::AABB aabb2(glm::vec3(1.0), 1.0);\n")
  file(APPEND src/main.cpp "  std::cout << \"AABB Interesction: \" << aabb.intersect(aabb2) << std::endl;\n")
  file(APPEND src/main.cpp "  return 0;\n")
  file(APPEND src/main.cpp "}\n")

  set(Sources src/main.cpp)

  #-----------------------------------------------------------------------
  # Setup executable
  #-----------------------------------------------------------------------
  set(EXE_NAME cpm-test)
  add_executable(${EXE_NAME} ${Sources})
  target_link_libraries(${EXE_NAME} ${CPM_LIBRARIES})
```

Using CPM
=========

Quick Setup
-----------

To use CPM in your C++ project, include the following at the top of your
CMakeLists.txt:

```cmake
  #------------------------------------------------------------------------------
  # Required CPM Setup - no need to modify - See: https://github.com/iauns/cpm
  #------------------------------------------------------------------------------
  set(CPM_DIR "${CMAKE_CURRENT_BINARY_DIR}/cpm-packages" CACHE TYPE STRING)
  find_package(Git)
  if(NOT GIT_FOUND)
    message(FATAL_ERROR "CPM requires Git.")
  endif()
  if (NOT EXISTS ${CPM_DIR}/CPM.cmake)
    message(STATUS "Cloning repo (https://github.com/morpheby/scpm)")
    execute_process(
      COMMAND "${GIT_EXECUTABLE}" clone https://github.com/morpheby/scpm ${CPM_DIR}
      RESULT_VARIABLE error_code
      OUTPUT_QUIET ERROR_QUIET)
    if(error_code)
      message(FATAL_ERROR "CPM failed to get the hash for HEAD")
    endif()
  endif()
  include(${CPM_DIR}/CPM.cmake)
  
  #------------------------------------------------------------------------------
  # CPM Modules
  #------------------------------------------------------------------------------

  # TODO: Include any modules here...
  
  CPM_Finish()

```

Then use the targets provided by your libraries' CMake files like that:

```cmake
target_link_libraries(my_project anotherProject::anotherProject) 
```

And you're done. You will be able to start using CPM modules right away by
adding the following snippet to the CPM Modules section of your
CMakeLists.txt:

```cmake
  CPM_AddModule("aabb"
    GIT_REPOSITORY "https://github.com/iauns/cpm-glm-aabb"
    GIT_TAG "1.0.2")
```

This snippet will automatically download, build, and add to CMake scope
version 1.0.2 of a simple axis aligned bounding box implementation named `aabb`.

Be sure to place all calls to `CPM_AddModule` before your call to
`CPM_Finish`. The ``# TODO: Include any modules here...``
section mentioned in the first snippet indicates where you should place calls
to ``CPM_AddModule``.


Things To Note
--------------

### Tag Advice

While it may be tempting to use the `origin/master` tag to track the most
recent changes to a module, it is not recommended. Using version tags for a
module (such as `1.0.2`) and upgrading modules when necessary will save you
time in the long run. If you track `origin/master` and upstream decides to
release a major version which includes significant API changes then your
builds will likely break immediately. But if versioned tags are used, you
will maintain your build integrity even through upstream version upgrades.

### Advantages

* Automatically manages code retrieval and building of CMake projects.
* Built entirely in CMake. Nothing else is required.
* Optionally cache downloaded modules and repositories to a central directory.

### Limitations

* Only supports git (with very limited support for SVN).

CPM Function Reference
======================

All CMake functions that SCPM exposes are listed below.

General Purpose
---------------

### CPM_AddModule

Adds a CPM module to your project. All arguments except `<name>` are optional.
Additionally, one of either the `GIT_REPOSITORY` or `SOURCE_DIR` arguments
must be present in your call to `CPM_AddModule`. Should be called before
either `CPM_Finish` or `CPM_InitModule`

```cmake
  CPM_AddModule(
    [GIT_REPOSITORY repo]        # Git repository that corresponds to a CPM module.
                                 # If this argument is not specfied, then SOURCE_DIR must be set.
    [GIT_TAG tag]                # Git tag to checkout. Tags, shas, and branches all work.
    [SOURCE_DIR dir]             # Uses 'dir' as the source directory instead of cloning
                                 # from a repo. If this is not specified, then 
                                 # GIT_REPOSITORY must be specified.
    [SOURCE_GHOST_GIT_REPO repo] # Ghost repository when using SOURCE_DIR.
                                 # Used to correctly correlate SOURCE_DIR modules with their 
                                 # correct upstream repository.
    [SOURCE_GHOST_GIT_TAG tag]   # Ghost git tag when using SOURCE_DIR.
    )
```

### CPM_EnsureRepoIsCurrent

A utility function that allows you to download and ensure that some repository
is up to date and present on the filesystem before proceeding forward with
CMakeLists.txt processing. Useful for building CPM externals. You can use this
function outside of any call to CPM_Finish or CPM_InitModule

```cmake
  CPM_EnsureRepoIsCurrent(
    [TARGET_DIR dir]             # Required - Directory in which to place repository.
    [GIT_REPOSITORY repo]        # Git repository to clone and keep up to date.
    [GIT_TAG tag]                # Git tag to checkout.
    [SVN_REPOSITORY repo]        # SVN repository to checkout.
    [SVN_REVISION rev]           # SVN revision.
    [SVN_TRUST_CERT 1]           # Trust the Subversion server site certificate
    [USE_CACHING 1]              # Enables caching of repositories if the user 
                                 # has specified CPM_MODULE_CACHING_DIR.
                                 # Not enabled by default.
    )
```

### CPM_Finish

This function is for the top-level application only. Call when you are
finished issuing calls to `CPM_AddModule`.

```cmake
CPM_Finish()
```

### CPM_InitModule

This function is the module's counterpart to `CPM_Finish`. Call this to
indicate to CPM that you have finished issuing calls to `CPM_AddModule`.
The only argument indicates the name of the module. This name will only be
used for generating the preprocessor definition you should use for your
module.

```cmake
CPM_InitModule("my_module")
```

Miscellaneous Issues and Questions
==================================

Below are some common issues users encounter and solutions to them.

When are modules downloaded and updated?
----------------------------------------

During the CMake configure step. No repository cloning or fetching occurs
during the build step.

How do I cache modules?
-----------------------

CPM supports cached repositories and modules by setting either the
`CPM_MODULE_CACHE_DIR` CMake variable or the `CPM_CACHE_DIR` environment
variable to an appropriate cache directory (such as `~/.cpm_cache`). When
set, a search will be performed in the cache directory for all modules that
don't already exist in your project's build directory. If the module is not
found in the cache directory, CPM will download the module into the cache
directory. This is useful if you find yourself with no or limited internet
access from time to time as your cache directory will be searched before
attempting to download the repository from the internet.

Here's a quick example of using this variable from the command line:

```
  cmake -DCPM_MODULE_CACHE_DIR=~/.cpm_cache ...
```

The cache directory is searched only when a module is not found in your
project's build directory. If the module is then found in the cache directory,
the cache directory will be updated using the appropriate SCM and its
directory contents will be copied into your project's build directory. Any
subsequent invokation of CMake will find the module in your project's build
directory and will not search in the cache directory. Unless you have cleaned
the project or removed the build directory's modules.

How do I cache CPM itself?
--------------------------

Use the following code snippet instead of the one given above. This snippet
checks to see if CPM exists in the cache directory before attempting to
download it.

```cmake
#------------------------------------------------------------------------------
# Required CPM Setup - See: http://github.com/iauns/cpm
#------------------------------------------------------------------------------
set(CPM_DIR "${CMAKE_CURRENT_BINARY_DIR}/cpm-packages" CACHE TYPE STRING)
find_package(Git)
if(NOT GIT_FOUND)
  message(FATAL_ERROR "CPM requires Git.")
endif()
if ((NOT DEFINED CPM_MODULE_CACHE_DIR) AND (NOT "$ENV{CPM_CACHE_DIR}" STREQUAL ""))
  set(CPM_MODULE_CACHE_DIR "$ENV{CPM_CACHE_DIR}")
endif()
if ((NOT EXISTS ${CPM_DIR}/CPM.cmake) AND (DEFINED CPM_MODULE_CACHE_DIR))
  if (EXISTS "${CPM_MODULE_CACHE_DIR}/github_morpheby_scpm")
    message(STATUS "Found cached version of SCPM.")
    file(COPY "${CPM_MODULE_CACHE_DIR}/github_morpheby_scpm/" DESTINATION ${CPM_DIR})
  endif()
endif()
if (NOT EXISTS ${CPM_DIR}/CPM.cmake)
  message(STATUS "Cloning repo (https://github.com/morpheby/scpm)")
  execute_process(
    COMMAND "${GIT_EXECUTABLE}" clone https://github.com/morpheby/scpm ${CPM_DIR}
    RESULT_VARIABLE error_code
    OUTPUT_QUIET ERROR_QUIET)
  if(error_code)
    message(FATAL_ERROR "CPM failed to get the hash for HEAD")
  endif()
endif()
include(${CPM_DIR}/CPM.cmake)

# Modules go here...

CPM_Finish()
```

<a name="dependency-hierarchy"></a>
How do I see the module dependency hierarchy?
---------------------------------------------

When building your project define: ``CPM_SHOW_HIERARCHY=TRUE``.

On the command line this would look something like

```
  cmake -DCPM_SHOW_HIERARCHY=TRUE ...
```

It is best to run this command after you have successfully built your project
so the output is not muddied by status messages.

I get errors regarding reused binary directories
------------------------------------------------

If you get errors similar to:

```
  The binary directory

    /Users/jhughes/me/cpp/cpm/modules/ ... /bin

  is already used to build a source directory.  It cannot be used to build
  source directory.
```

This means that there exists a circular module reference which is not allowed
in CPM. The module graph must not contain cycles. For example, if Module A
adds Module B, and Module B adds Module A, you will get this error.

