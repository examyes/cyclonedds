#
# Copyright(c) 2006 to 2019 ADLINK Technology Limited and others
#
# This program and the accompanying materials are made available under the
# terms of the Eclipse Public License v. 2.0 which is available at
# http://www.eclipse.org/legal/epl-2.0, or the Eclipse Distribution License
# v. 1.0 which is available at
# http://www.eclipse.org/org/documents/edl-v10.php.
#
# SPDX-License-Identifier: EPL-2.0 OR BSD-3-Clause
#
cmake_minimum_required(VERSION 3.7)
project(CycloneDDS VERSION 0.7.0 LANGUAGES C)
if(CMAKE_VERSION VERSION_LESS 3.12)
  # GENERATE_EXPORT_HEADER requires a C++ compiler up to version 3.12
  enable_language(CXX)
endif()

# Set a default build type if none was specified
set(default_build_type "RelWithDebInfo")
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# By default don't treat warnings as errors, else anyone building it with a different compiler that
# just happens to generate a warning, as well as anyone adding or modifying something and making a
# small mistake would run into errors.  CI builds can be configured differently.
option(WERROR "Treat compiler warnings as errors" OFF)

FUNCTION(PREPEND var prefix)
   SET(listVar "")
   FOREACH(f ${ARGN})
      LIST(APPEND listVar "${prefix}/${f}")
   ENDFOREACH(f)
   SET(${var} "${listVar}" PARENT_SCOPE)
ENDFUNCTION(PREPEND)

# Set module path before defining project so platform files will work.
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/Modules")
set(PROJECT_NAME_FULL "Eclipse Cyclone DDS")

# Set some convenience variants of the project-name
string(REPLACE " " "-" PROJECT_NAME_DASHED "${PROJECT_NAME_FULL}")
string(TOUPPER ${PROJECT_NAME} PROJECT_NAME_CAPS)
string(TOLOWER ${PROJECT_NAME} PROJECT_NAME_SMALL)

# By default the Java-based components get built, but make it possible to disable that: if only the
# core library is required, there's no need to build them, and that in turn eliminates the Maven and
# JDK dependency.
option(BUILD_IDLC "Build IDL preprocessor" ON)
if(BUILD_IDLC STREQUAL "AUTO")
  find_package(Maven 3.0 QUIET)
  set(BUILD_IDLC ${Maven_FOUND})
endif()

set(CMAKE_C_STANDARD 99)
if(CMAKE_SYSTEM_NAME STREQUAL "VxWorks")
  add_definitions(-std=c99)
endif()

if(${CMAKE_C_COMPILER_ID} STREQUAL "SunPro")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m64 -xc99 -D__restrict=restrict -D__deprecated__=")
  set(CMAKE_LINKER_FLAGS "${CMAKE_LINKER_FLAGS} -m64")
elseif(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m64")
  set(CMAKE_LINKER_FLAGS "${CMAKE_LINKER_FLAGS} -m64")
endif()

# Conan
if(EXISTS "${CMAKE_BINARY_DIR}/conanbuildinfo.cmake")
  include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
  if(APPLE)
    # By default Conan strips all RPATHs (see conanbuildinfo.cmake), which
    # causes tests to fail as the executables cannot find the library target.
    # By setting KEEP_RPATHS, Conan does not set CMAKE_SKIP_RPATH and the
    # resulting binaries still have the RPATH information. This is fine because
    # CMake will strip the build RPATH information in the install step.
    #
    # NOTE:
    # Conan's default approach is to use the "imports" feature, which copies
    # all the dependencies into the bin directory. Of course, this doesn't work
    # quite that well for libraries generated in this Project (see Conan
    # documentation).
    #
    # See the links below for more information.
    # https://github.com/conan-io/conan/issues/337
    # https://docs.conan.io/en/latest/howtos/manage_shared_libraries/rpaths.html
    # https://gitlab.kitware.com/cmake/community/wikis/doc/cmake/RPATH-handling
    conan_basic_setup(KEEP_RPATHS)
  else()
    conan_basic_setup()
  endif()
  conan_define_targets()
endif()

# Set reasonably strict warning options for clang, gcc, msvc
# Enable coloured ouput if Ninja is used for building
if("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" OR
   "${CMAKE_C_COMPILER_ID}" STREQUAL "AppleClang")
  #message(STATUS clang)
  set(wflags "-Wall"
             "-Wextra"
             "-Wconversion"
             "-Wunused"
             "-Wmissing-prototypes"
             "-Winfinite-recursion"
             "-Wassign-enum"
             "-Wcomma"
             "-Wmissing-prototypes"
             "-Wdocumentation"
             "-Wstrict-prototypes"
             "-Wconditional-uninitialized"
             "-Wshadow")
  add_compile_options(${wflags})
  if(${WERROR})
    add_compile_options(-Werror)
  endif()
  if("${CMAKE_GENERATOR}" STREQUAL "Ninja")
    add_compile_options(-Xclang -fcolor-diagnostics)
  endif()
elseif("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
  #message(STATUS gcc)
  add_compile_options(-Wall -Wextra -Wconversion -Wmissing-prototypes)
  if(${WERROR})
    add_compile_options(-Werror)
  endif()
  if("${CMAKE_GENERATOR}" STREQUAL "Ninja")
    add_compile_options(-fdiagnostics-color=always)
  endif()
elseif("${CMAKE_C_COMPILER_ID}" STREQUAL "MSVC")
  #message(STATUS msvc)
  add_compile_options(/W3)
  if(${WERROR})
    add_compile_options(/WX)
  endif()
endif()

# I don't know how to enable warnings properly so that they are enabled in Xcode projects as well
if(${CMAKE_GENERATOR} STREQUAL "Xcode")
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_EMPTY_BODY YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_SHADOW YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_BOOL_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_CONSTANT_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_64_TO_32_BIT_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_ENUM_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_FLOAT_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_INT_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_NON_LITERAL_NULL_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_IMPLICIT_SIGN_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_INFINITE_RECURSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_INITIALIZER_NOT_FULLY_BRACKETED YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_ABOUT_RETURN_TYPE YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_MISSING_PARENTHESES YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_ABOUT_MISSING_FIELD_INITIALIZERS YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_ABOUT_MISSING_NEWLINE YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_ASSIGN_ENUM YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_SEMICOLON_BEFORE_METHOD_BODY YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_SIGN_COMPARE YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_STRICT_PROTOTYPES YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_COMMA YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_SUSPICIOUS_IMPLICIT_CONVERSION YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_UNINITIALIZED_AUTOS YES_AGGRESSIVE)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_UNUSED_FUNCTION YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_UNUSED_LABEL YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_UNUSED_PARAMETER YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_UNUSED_VALUE YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_UNUSED_VARIABLE YES)
  set (CMAKE_XCODE_ATTRIBUTE_CLANG_WARN_DOCUMENTATION_COMMENTS YES)
  set (CMAKE_XCODE_ATTRIBUTE_GCC_WARN_ABOUT_MISSING_PROTOTYPES YES)
endif()


# Make it easy to enable Clang's/gcc's analyzers
set(USE_SANITIZER "" CACHE STRING "Sanitizers to enable on the build.")
if(NOT("${USE_SANITIZER}" STREQUAL ""))
  foreach(san "${USE_SANITIZER}")
    message(STATUS "Enabling sanitizer: '${san}'")

    if("${san}" STREQUAL address)
      add_compile_options(-fno-omit-frame-pointer)
      link_libraries(-fno-omit-frame-pointer)
    endif()

    if(NOT("${san}" STREQUAL "none"))
      add_compile_options("-fsanitize=${san}")
      link_libraries("-fsanitize=${san}")
    endif()
  endforeach()
endif()


include(GNUInstallDirs)
include(AnalyzeBuild)
if(APPLE)
  set(CMAKE_INSTALL_RPATH "@loader_path/../${CMAKE_INSTALL_LIBDIR}")
else()
  set(CMAKE_INSTALL_RPATH "$ORIGIN/../${CMAKE_INSTALL_LIBDIR}")
endif()
set(MEMORYCHECK_COMMAND_OPTIONS "--track-origins=yes --leak-check=full --trace-children=yes --child-silent-after-fork=yes --xml=yes --xml-file=TestResultValgrind_%p.xml --tool=memcheck --show-reachable=yes --leak-resolution=high")

# By default building the testing tree is enabled by including CTest, but
# since not everybody has CUnit, and because it is not strictly required to
# build the product itself, switch to off by default.
option(BUILD_TESTING "Build the testing tree." OFF)
include(CTest)

option(BUILD_DOCS "Build documentation." OFF)
if(CMAKE_CROSSCOMPILING)
  set(not_crosscompiling OFF)
else()
  set(not_crosscompiling ON)
endif()
option(BUILD_SCHEMA "Build generated schema for configuration options." ${not_crosscompiling})
# Build all executables and libraries into the top-level /bin and /lib folders.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")

add_subdirectory(src)
if(BUILD_IDLC)
  add_subdirectory(examples)
endif()


add_subdirectory(docs)

# Pull-in CPack and support for generating <Package>Config.cmake and packages.
include(Packaging)
