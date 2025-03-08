cmake_minimum_required(VERSION 3.14)

set(CMAKE_LEGACY_CYGWIN_WIN32 0)
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.15" CACHE STRING "Minimum OS X deployment target")

project(g2o)

include(CPack)
include(GNUInstallDirs)

set(LIB_PREFIX g2o_)

set(G2O_LIB_VERSION   "0.2.0" CACHE STRING "g2o library version")
set(G2O_LIB_SOVERSION "0.2"   CACHE STRING "g2o library soversion")
set(G2O_VERSION 1.0.0)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build: Debug, Release, RelWithDebInfo, MinSizeRel." FORCE)
endif()

option (BUILD_SHARED_LIBS "Build Shared Libraries (required for the g2o plugin system)" ON)
set (G2O_LIB_TYPE STATIC)
if (BUILD_SHARED_LIBS)
    set (G2O_LIB_TYPE SHARED)
endif()

if(APPLE)
    set(CMAKE_INSTALL_RPATH "")
    set(CMAKE_MACOSX_RPATH TRUE)
    add_definitions(-DGL_SILENCE_DEPRECATION)
endif()

set(g2o_RUNTIME_OUTPUT_DIRECTORY ${g2o_BINARY_DIR}/bin CACHE PATH "Target for the binaries")
if(WIN32)
    set(g2o_LIBRARY_OUTPUT_DIRECTORY ${g2o_BINARY_DIR}/bin CACHE PATH "Target for the libraries")
else()
    set(g2o_LIBRARY_OUTPUT_DIRECTORY ${g2o_BINARY_DIR}/lib CACHE PATH "Target for the libraries")
endif()
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${g2o_LIBRARY_OUTPUT_DIRECTORY})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${g2o_LIBRARY_OUTPUT_DIRECTORY})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${g2o_RUNTIME_OUTPUT_DIRECTORY})

set(RUNTIME_DESTINATION ${CMAKE_INSTALL_BINDIR})
set(LIBRARY_DESTINATION ${CMAKE_INSTALL_LIBDIR})
set(ARCHIVE_DESTINATION ${CMAKE_INSTALL_LIBDIR})
set(INCLUDES_DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
set(INCLUDES_INSTALL_DIR ${INCLUDES_DESTINATION}/g2o)

list(APPEND CMAKE_MODULE_PATH ${g2o_SOURCE_DIR}/cmake_modules)

# **✅ GCC 8 대응: C++17 및 -lstdc++fs 추가**
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(CMAKE_COMPILER_IS_GNUCXX AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9)
    message(STATUS "Using GCC version < 9: Adding -lstdc++fs")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lstdc++fs")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -lstdc++fs")
endif()

if(WIN32)
    add_definitions(-DWINDOWS)
    message(STATUS "Compiling on Windows")
elseif(CYGWIN)
    message(STATUS "Compiling on Cygwin")
    add_definitions(-DCYGWIN)
elseif(APPLE)
    add_definitions(-DUNIX)
    message(STATUS "Compiling on OSX")
elseif(UNIX)
    add_definitions(-DUNIX)
    message(STATUS "Compiling on Unix")
endif()

option(G2O_USE_CSPARSE "Build g2o with CSparse support" ON)
find_package(CSparse)
if (G2O_USE_CSPARSE AND CSPARSE_FOUND)
    message(STATUS "Enable support for CSparse")
else()
    message(STATUS "Disable support for CSparse")
    set(G2O_USE_CSPARSE FALSE)
endif()

option(G2O_USE_OPENMP "Build g2o with OpenMP support (EXPERIMENTAL)" OFF)
if(G2O_USE_OPENMP)
    find_package(OpenMP)
    if(OPENMP_FOUND)
        set (G2O_OPENMP 1)
        set(g2o_CXX_FLAGS "${g2o_CXX_FLAGS} -DEIGEN_DONT_PARALLELIZE ${OpenMP_CXX_FLAGS}")
        message(STATUS "Compiling with OpenMP support")
    endif()
endif()

set(OpenGL_GL_PREFERENCE "GLVND")
find_package(OpenGL)

option(G2O_USE_OPENGL "Build g2o with OpenGL support for visualization" ON)
set(G2O_HAVE_OPENGL 0)
if (OPENGL_FOUND AND G2O_USE_OPENGL)
    if (TARGET OpenGL::GL)
        set(G2O_OPENGL_TARGET "OpenGL::GL;OpenGL::GLU")
    else()
        set(G2O_OPENGL_TARGET "${OPENGL_LIBRARIES}")
        include_directories(${OPENGL_INCLUDE_DIR})
    endif()
    set(G2O_HAVE_OPENGL 1)
    message(STATUS "Compiling with OpenGL support")
endif()

find_package(QGLViewer)

option(G2O_USE_LOGGING "Try to use spdlog for logging" ON)
set(G2O_HAVE_LOGGING 0)
if (G2O_USE_LOGGING)
    find_package(spdlog 1.6 QUIET)
    if (TARGET spdlog::spdlog OR TARGET spdlog::spdlog_header_only)
        set(G2O_HAVE_LOGGING 1)
        message(STATUS "Compiling with logging support")
    endif()
endif()

option(G2O_BUILD_SLAM2D_TYPES "Build SLAM2D types" ON)
option(G2O_BUILD_SLAM3D_TYPES "Build SLAM 3D types" ON)

if(G2O_BUILD_SLAM2D_TYPES)
    message(STATUS "Compiling SLAM 2D types")
endif()

if(G2O_BUILD_SLAM3D_TYPES)
    message(STATUS "Compiling SLAM 3D types")
endif()

option(G2O_BUILD_APPS "Build g2o apps" ON)
if(G2O_BUILD_APPS)
    message(STATUS "Compiling g2o apps")
endif()

option(G2O_BUILD_EXAMPLES "Build g2o examples" ON)
if(G2O_BUILD_EXAMPLES)
    message(STATUS "Compiling g2o examples")
endif()

option(G2O_FAST_MATH "Enable fast math operations" OFF)
option(G2O_NO_IMPLICIT_OWNERSHIP_OF_OBJECTS "Disable memory management in graph types" OFF)

option(BUILD_CODE_COVERAGE "Enable coverage reporting" OFF)
if(BUILD_CODE_COVERAGE AND CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    message(STATUS "Enabling coverage compiler flags")
    set(g2o_CXX_FLAGS "${g2o_CXX_FLAGS} --coverage")
endif()

option (BUILD_WITH_MARCH_NATIVE "Build with \"-march native\"" OFF)
if(CMAKE_COMPILER_IS_GNUCXX)
    message(STATUS "Compiling with GCC")

    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3")

    if(G2O_FAST_MATH)
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -ffast-math")
    endif()

    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -O0")

    if(BUILD_WITH_MARCH_NATIVE AND NOT "${CMAKE_SYSTEM_PROCESSOR}" MATCHES "arm" AND "${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native")
    endif()
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${g2o_CXX_FLAGS}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${g2o_C_FLAGS}")

find_package(Eigen3 3.3 REQUIRED)
if (TARGET Eigen3::Eigen)
    set(G2O_EIGEN3_EIGEN_TARGET Eigen3::Eigen)
else()
    include_directories(${EIGEN3_INCLUDE_DIR})
endif ()

include_directories(${CMAKE_SOURCE_DIR}/g2o)
add_subdirectory(g2o)

