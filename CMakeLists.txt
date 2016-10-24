project(GraphLab)

# We require the most recent version of cmake and automatically
# install the correct version when using the cmake lists
cmake_minimum_required(VERSION 2.8)

# Libraries linked via full path no longer produce linker search paths.
cmake_policy(SET CMP0003 NEW)
# Preprocessor definition values are now escaped automatically.
cmake_policy(SET CMP0005 NEW)

# Determine where additional GraphLab specific cmake modules are
# defined
set(CMAKE_MODULE_PATH ${GraphLab_SOURCE_DIR}/cmake)

# TODO: what is this used by
set(BUILD_SHARED_LIBS ON)

include(CheckLibraryExists)
include(CheckFunctionExists)
include(ExternalProject)
include(CheckCXXSourceCompiles)

# check we are on a 64 bit machine (else fail)
if( CMAKE_SIZEOF_VOID_P EQUAL 4 )
   message( SEND_ERROR "GraphLab does not support 32 bit systems. Please switch to 64 bit system and try again" )
endif ( CMAKE_SIZEOF_VOID_P EQUAL 4 )

# # Display information about cmake variables
# include(CMakePrintSystemInformation)

# set include path
include_directories(
  ${GraphLab_SOURCE_DIR}/src
  ${GraphLab_SOURCE_DIR}/cxxtest
)

add_definitions(-DUSE_DYNAMIC_LOCAL_GRAPH)

if(NO_OPENMP)
  set(OPENMP_C_FLAGS "")
  set(OPENMP_LIBRARIES "")
  add_definitions(-D__NO_OPENMP__)
else()
  set(OPENMP_C_FLAGS "-fopenmp")
  set(OPENMP_LIBRARIES "gomp")
endif()

if (APPLE)
link_libraries(pthread ${OPENMP_LIBRARIES})
else()
link_libraries(pthread rt ${OPENMP_LIBRARIES})
endif()

set(CPP11_FLAGS "")
if(CPP11)
  message(STATUS "C++11 Enabled")
  set(CPP11_FLAGS "-std=c++11 -Wno-enum-compare -Wno-conversion-null")
endif()

if(VID32)
  message(STATUS "Using 32bit vertex id types")
  add_definitions(-DUSE_VID32)
endif()


# Shared compiler flags used by all builds (debug, profile, release)
set(COMPILER_FLAGS "-Wall -g ${CPP11_FLAGS} ${OPENMP_C_FLAGS}" CACHE STRING "common compiler options")

# Disable address space randomization for OSX lion and above
if (APPLE)
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Xlinker -no_pie")
endif()

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-rpath,${GraphLab_SOURCE_DIR}/deps/local/lib")

# Set subdirectories
subdirs(src tests demoapps toolkits)
if(EXPERIMENTAL)
  if (IS_DIRECTORY ${GraphLab_SOURCE_DIR}/experimental)
    subdirs(experimental)
  endif()
endif()

if (IS_DIRECTORY ${GraphLab_SOURCE_DIR}/apps)
  subdirs(apps)
endif()

# Set installation rules
install(DIRECTORY src/
  DESTINATION include
  FILES_MATCHING PATTERN "*.hpp"  PATTERN "*.h"
  PATTERN ".svn" EXCLUDE
)

find_package(Eigen3)
if(EIGEN3_FOUND)
    include_directories(${EIGEN3_INCLUDE_DIR})
endif()

macro(requires_eigen NAME)
    if(NOT EIGEN3_FOUND)
        message(SEND_ERROR "Eigen 3 not found")
    endif()
endmacro(requires_eigen)

if(NOT NO_TCMALLOC)
    set(TCMALLOC-FOUND 1)
    add_definitions(-DHAS_TCMALLOC)
endif()


# OpenCV =================================================================
macro(requires_opencv NAME)
    target_link_libraries(${NAME} ${OpenCV_LIBS})
endmacro(requires_opencv)

## =============================================================
# JNI =========================================================================
# We use JNI for jvm interfacing and for hdfs support.  We currently only
# detect the presence of java and do not install our own version.

if(NOT NO_JAVAC)
  find_package(Java)
  find_package(JNI)
  include_directories(${JNI_INCLUDE_DIRS})
  get_filename_component(JAVA_JVM_LIBRARY_DIR ${JAVA_JVM_LIBRARY} DIRECTORY)
  SET(CMAKE_INSTALL_RPATH ${JAVA_JVM_LIBRARY_DIR})

  find_library(HDFS_LIBRARY hdfs)


  # Ant is used to build the graphlab JVM interface
  find_package(Ant)
  add_definitions(-DHAS_HADOOP)
endif()



# MPI =========================================================================
# MPI is currently used to establish communication between rpc nodes
#sgr this name is confusing. i like WITH_MPI better. likewise for NO_OPENMP
if(NO_MPI)
  message(STATUS "MPI disabled as a configure option.")
else()
  #find_package(MPICH2)
  find_package(MPI) 
  if(MPI_FOUND)
    message(STATUS "MPI Found: " ${MPI_LIBRARY} ${MPI_EXTRA_LIBRARY})
    add_definitions(-DHAS_MPI)
    include_directories(${MPI_INCLUDE_PATH})
    set(CMAKE_CXX_COMPILE_FLAGS ${CMAKE_CXX_COMPILE_FLAGS} ${MPI_COMPILE_FLAGS})
    set(CMAKE_CXX_LINK_FLAGS ${CMAKE_CXX_LINK_FLAGS} ${MPI_LINK_FLAGS})
  else(MPI_FOUND)
    message(STATUS "MPI Not Found! Distributed Executables will not be compiled")
    set(MPI_LIBRARY "")
    set(MPI_EXTRA_LIBRARY "")
    # ExternalProject_Add(openmpi
    #   PREFIX ${GraphLab_SOURCE_DIR}/deps/openmpi
    #   URL http://www.open-mpi.org/software/ompi/v1.4/downloads/openmpi-1.4.5.tar.gz
    #   CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=<INSTALL_DIR>
    #   INSTALL_DIR ${GraphLab_SOURCE_DIR}/deps/local)
  endif(MPI_FOUND)
endif()

find_package(Boost 1.59 REQUIRED
    COMPONENTS filesystem program_options system iostreams context date_time)

find_library(Zookeeper_LIBRARY zookeeper_mt)

find_package( OpenCV REQUIRED )

## =============================================================
# Core graphlab requirements
macro(requires_core_deps NAME)
  target_link_libraries(${NAME} z ${Boost_LIBRARIES} event event_pthreads ${Zookeeper_LIBRARY})

  if(MPI_FOUND)
    target_link_libraries(${NAME} ${MPI_LIBRARY} ${MPI_EXTRA_LIBRARY})
  endif(MPI_FOUND)

  target_link_libraries(${NAME} ${HDFS_LIBRARY} ${JAVA_JVM_LIBRARY})

  if(NOT NO_TCMALLOC)
    target_link_libraries(${NAME} tcmalloc)
  endif()
endmacro(requires_core_deps)

## ============================================================================
# System Checks

## ============================================================================
# Test for cpu affinity support
set(CMAKE_REQUIRED_LIBRARIES "pthread")
check_function_exists(pthread_setaffinity_np HAS_SET_AFFINITY)
set(CMAKE_REQUIRED_LIBRARIES ${crlbackup})

include(CheckCXXCompilerFlag)
## ============================================================================
# check if MARCH is set
if (NOT MARCH)
  #set to native if supported
  check_cxx_compiler_flag(-march=native HAS_MARCH_NATIVE)
  if(HAS_MARCH_NATIVE)
    set(MARCH "native")
  else()
    set(MARCH "x86-64")
  endif()
endif()

# check if MTUNE is set
if (NOT MTUNE)
  #set to native if supported
  check_cxx_compiler_flag(-mtune=native HAS_MTUNE_NATIVE)
  if(HAS_MTUNE_NATIVE)
    set(MTUNE "native")
  else()
    set(MTUNE "x86-64")
  endif()
endif()

## ============================================================================
# Setup compiler

# If profiling mode is enabled then additional profiling flags are set for
# the compiler
if (COMPILE_PROFILING MATCHES 1)
  set(PROFILING_FLAGS "-DUSE_EVENT_LOG -DUSE_TRACEPOINT")
else()
  set(PROFILING_FLAGS "")
endif()

# check for SSE instruction set (needed for CRC32)
#check_cxx_compiler_flag(-msse4.2 HAS_SSE42)
#set(SSE42_FLAG "")
#if (HAS_SSE42)
# set(SSE42_FLAG "-msse4.2")
#endif()


#disable Wno-unused-local-typedefs if available
check_cxx_compiler_flag(-Wno-unused-local-typedefs HAS_WNO_LOCAL_TYPEDEFS)
if(HAS_WNO_LOCAL_TYPEDEFS)
  set(WNO_LOCAL_TYPEDEFS "-Wno-unused-local-typedefs")
else()
  set(WNO_LOCAL_TYPEDEFS "")
endif()


# Set the debug flags
set(CMAKE_C_FLAGS_DEBUG
  "-O0 -Wno-attributes -march=${MARCH} -Winit-self ${PROFILING_FLAGS} ${COMPILER_FLAGS}"
  CACHE STRING "compiler options" FORCE)
set(CMAKE_CXX_FLAGS_DEBUG
  "-O0 ${WNO_LOCAL_TYPEDEFS} -Wno-attributes -march=${MARCH} -Winit-self ${PROFILING_FLAGS} ${COMPILER_FLAGS}"
  CACHE STRING "compiler options" FORCE)

set(CMAKE_C_FLAGS_RELEASE
  "-O3 -Wno-attributes -march=${MARCH} -mtune=${MTUNE} ${PROFILING_FLAGS} ${COMPILER_FLAGS}"
  CACHE STRING "compiler options" FORCE)
set(CMAKE_CXX_FLAGS_RELEASE
  "-O3 ${WNO_LOCAL_TYPEDEFS} -Wno-attributes -march=${MARCH} -mtune=${MTUNE} ${PROFILING_FLAGS} ${COMPILER_FLAGS}"
  CACHE STRING "compiler options" FORCE)

set(CMAKE_REQUIRED_FLAGS ${CMAKE_CXX_FLAGS_RELEASE})
check_cxx_source_compiles("int main(int argc, char** argv) { return __builtin_ia32_crc32di(0,0);}" HAS_CRC32)
set(CMAKE_REQUIRED_FLAGS "")
if(HAS_CRC32)
  set(CMAKE_CXX_FLAGS_RELEASE
      "${CMAKE_CXX_FLAGS_RELEASE} -DHAS_BUILTIN_CRC32"
      CACHE STRING "compiler options" FORCE)
endif()

# check for gcc 4.8

if (CMAKE_BUILD_TYPE MATCHES "Release")
  message(STATUS "Release build with C++ flags: " ${CMAKE_CXX_FLAGS_RELEASE})
  message(STATUS "Release build with C flags: " ${CMAKE_C_FLAGS_RELEASE})
elseif(CMAKE_BUILD_TYPE MATCHES "Debug")
  message(STATUS "Debug build with C++ flags: " ${CMAKE_CXX_FLAGS_DEBUG})
  message(STATUS "Debug build with C flags: " ${CMAKE_C_FLAGS_DEBUG})
elseif(CMAKE_BUILD_TYPE MATCHES "Mex")
  message(STATUS "Mex CXX" ${CMAKE_CXX_FLAGS_MEX})
  message(STATUS "Mex C" ${CMAKE_C_FLAGS_MEX})
else()
  message(WARNING "Unknown build type: " ${CMAKE_BUILD_TYPE} "!")
endif()







## ============================================================================
## ============================================================================
## ============================================================================
# Setup testing tools
# Make sure testing is enabled
enable_testing()
# Use Python interpreter
find_package(PythonInterp)
set(CXXTESTGEN ${CMAKE_SOURCE_DIR}/cxxtest/cxxtestgen)
# create a macro to define a test
macro(ADD_CXXTEST NAME)
  if(PYTHONINTERP_FOUND)
    add_custom_command(
      OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp
      COMMAND
      ${PYTHON_EXECUTABLE} ${CXXTESTGEN}
      --runner=ErrorPrinter
      -o ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp ${ARGV}
      DEPENDS ${ARGV}
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
      )
  endif(PYTHONINTERP_FOUND)
  add_graphlab_executable(${NAME}test ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp)
  set_source_files_properties( ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp
    PROPERTIES COMPILE_FLAGS "-I${CMAKE_CURRENT_SOURCE_DIR}" )

  add_test(${NAME} ${NAME}test)
endmacro(ADD_CXXTEST)


# macro(ADD_CPPTEST NAME)
#   add_graphlab_executable(${NAME}test ${NAME})
#   set_source_files_properties(${NAME}
#     PROPERTIES COMPILE_FLAGS "-I${CMAKE_CURRENT_SOURCE_DIR}" )
#   add_test(${NAME} ${NAME}test)
# endmacro(ADD_CPPTEST)






## ============================================================================
## ============================================================================
## ============================================================================
# Build Macros

# copy_file ===================================================================
# copy a single file into build environment
macro(copy_file NAME)
  message(STATUS "Copying File: " ${NAME})
  file(INSTALL ${CMAKE_CURRENT_SOURCE_DIR}/${NAME}
    DESTINATION   ${CMAKE_CURRENT_BINARY_DIR} )
endmacro(copy_file NAME)

# copy_files ==================================================================
# copy all files matching a pattern into the build environment
macro(copy_files NAME)
  message(STATUS "Copying Files: " ${NAME})
  file(INSTALL ${CMAKE_CURRENT_SOURCE_DIR}/
    DESTINATION  ${CMAKE_CURRENT_BINARY_DIR}
    FILES_MATCHING PATTERN ${NAME} )
endmacro(copy_files NAME)


# macro(EXEC file prog)
#   add_custom_command(
#     OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${file}
#     COMMAND
#     ${prog}
#     DEPENDS ${prog}
#     WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
#     )
# endmacro(EXEC)

# add_graphlab_executable =====================================================
# copy all files matching a pattern into the build environment
macro(add_graphlab_executable NAME)
  add_executable(${NAME} ${ARGN})
  target_link_libraries(${NAME} graphlab)
  add_dependencies(${NAME} graphlab)
  install(TARGETS ${NAME} DESTINATION bin)
  requires_core_deps(${NAME})
endmacro(add_graphlab_executable)


macro(add_extension_executable NAME)
  add_executable(${NAME} extension_main.cpp ${ARGN})
  target_link_libraries(${NAME} graphlab graphlab_extension)
  add_dependencies(${NAME} graphlab graphlab_extension)
  requires_core_deps(${NAME})
endmacro(add_extension_executable)


# add_jni_library =============================================================
# If jni was found then create a jni library.  Otherwise generate a warning
macro(add_jni_library NAME)
  target_link_libraries(${NAME}  graphlab_pic)
endmacro(add_jni_library)
