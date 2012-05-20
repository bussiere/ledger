cmake_minimum_required(VERSION 2.8.8)

project(Ledger)

set(Ledger_VERSION_MAJOR 3)
set(Ledger_VERSION_MINOR 0)
set(Ledger_VERSION_PATCH 0)
set(Ledger_VERSION_DATE  20120518)

enable_testing()

########################################################################

option(USE_PYTHON "Build support for the Python scripting bridge" OFF)
option(USE_DOXYGEN "Build reference documentation using Doxygen" OFF)

option(NO_ASSERTS "Build without any internal consistency checks" OFF)
option(BUILD_DEBUG "Build support for runtime debugging" ON)

option(BUILD_LIBRARY "Build and install Ledger as a library" ON)
option(BUILD_DOCS "Build and install documentation" OFF)
option(BUILD_EMACSLISP "Build and install ledger-mode for Emacs" OFF)

if(BUILD_DEBUG)
  set(CMAKE_BUILD_TYPE Debug)
  set(DEBUG_MODE 1)
  set(NDEBUG 0)
elseif(NO_ASSERTS)
  set(CMAKE_BUILD_TYPE Release)
  set(DEBUG_MODE 0)
  set(NDEBUG 1)
else()
  set(CMAKE_BUILD_TYPE Release)
  set(DEBUG_MODE 0)
  set(NDEBUG 0)
endif()

########################################################################

find_package(PythonInterp)      # Used for running tests

if(USE_PYTHON)
  find_package(PythonLibs)
  if(PYTHONLIBS_FOUND)
    set(BOOST_PYTHON python)
    set(HAVE_BOOST_PYTHON 1)
    include_directories(SYSTEM ${PYTHON_INCLUDE_DIRS})
  else()
    set(HAVE_BOOST_PYTHON 0)
    message("Could not find a Python library to use with Boost.Python")
  endif()
else()
  set(HAVE_BOOST_PYTHON 0)
endif()

set(Boost_USE_MULTITHREADED OFF)

find_package(Boost 1.47.0
  REQUIRED date_time filesystem system iostreams regex 
           unit_test_framework test_exec_monitor
  OPTIONAL_COMPONENTS ${BOOST_PYTHON})

include_directories(SYSTEM ${Boost_INCLUDE_DIRS})

########################################################################

include(CheckIncludeFiles)
include(CheckLibraryExists)
include(CheckFunctionExists)
include(CheckCSourceCompiles)
include(CheckCXXSourceRuns)
include(CMakePushCheckState)

check_function_exists(access HAVE_ACCESS)
check_function_exists(realpath HAVE_REALPATH)
check_function_exists(getpwuid HAVE_GETPWUID)
check_function_exists(getpwnam HAVE_GETPWNAM)
check_function_exists(isatty HAVE_ISATTY)

check_c_source_compiles("
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

int main() {
  int status, pfd[2];
  status = pipe(pfd);
  status = fork();
  if (status < 0) {
    ;
  } else if (status == 0) {
    char *arg0 = NULL;

    status = dup2(pfd[0], STDIN_FILENO);

    close(pfd[1]);
    close(pfd[0]);

    execlp(\"\", arg0, (char *)0);
    perror(\"execl\");
    exit(1);
  } else {
    close(pfd[0]);
  }
  return 0;
}" UNIX_PIPES_COMPILES)

if(UNIX_PIPES_COMPILES)
  set(HAVE_UNIX_PIPES 1)
else()
  set(HAVE_UNIX_PIPES 0)
endif()

cmake_push_check_state()

set(CMAKE_REQUIRED_INCLUDES ${CMAKE_INCLUDE_PATH} ${Boost_INCLUDE_DIRS})
set(CMAKE_REQUIRED_LIBRARIES ${Boost_LIBRARIES} icuuc)

check_cxx_source_runs("
#include <boost/regex/icu.hpp>

using namespace boost;

int main() {
  std::string text = \"Активы\";
  u32regex r = make_u32regex(\"активы\", regex::perl | regex::icase);
  return u32regex_search(text, r) ? 0 : 1;
}" BOOST_REGEX_UNICODE_RUNS)

if(BOOST_REGEX_UNICODE_RUNS)
  set(HAVE_BOOST_REGEX_UNICODE 1)
else()
  set(HAVE_BOOST_REGEX_UNICODE 0)
endif()

cmake_pop_check_state()

#cmake_push_check_state()
#
#set(CMAKE_REQUIRED_FLAGS -std=c++11)
#set(CMAKE_REQUIRED_INCLUDES ${CMAKE_INCLUDE_PATH})
#
#check_cxx_source_runs("
##include <regex>
##include <vector>
##include <iostream>
#
#int main() {
#  std::vector<int> x {0, 1, 2, 3, 4};
#  for (auto i : x)
#    std::cout << i << std::endl;
#
#  std::regex r(\"foo\");
#  std::cout << std::regex_match(\"foobar\", r) << std::endl;
#  return 0;
#}" CXX11_RUNS)
#
#cmake_pop_check_state()
#
#if(CXX11_RUNS)
#  set(HAVE_CXX11 1)
#  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
#else()
  set(HAVE_CXX11 0)
#endif()
#
#cmake_pop_check_state()

########################################################################

include_directories(${CMAKE_INCLUDE_PATH})

macro(find_opt_library_and_header _header_var _header _lib_var _lib _have_var)
  if(${_have_var})
    find_path(${_header_var} ${_header})
    if(NOT ${_header_var})
      set(${_have_var} 0)
    else()
      find_library(${_lib_var} ${_lib})
      if(NOT ${_lib_var})
        set(${_have_var} 0)
      else()
        include_directories(SYSTEM "${${_header_var}}")
        set(${_have_var} 1)
      endif()
    endif()
  else()
    set(${_have_var} 0)
  endif()
endmacro(find_opt_library_and_header _header_var _header _lib_var _lib _have_var)

macro(find_req_library_and_header _header_var _header _lib_var _lib)
  find_path(${_header_var} ${_header})
  if(NOT ${_header_var})
    message(SEND_ERROR "Could not find ${_header} on your system")
  else()
    include_directories(SYSTEM "${${_header_var}}")
    find_library(${_lib_var} ${_lib})
    if(NOT ${_lib_var})
      message(SEND_ERROR "Could not find library ${_lib} on your system")
    endif()
  endif()
endmacro(find_req_library_and_header _header_var _header _lib_var _lib)

find_req_library_and_header(GMP_PATH gmp.h GMP_LIB gmp)
find_req_library_and_header(MPFR_PATH mpfr.h MPFR_LIB mpfr)

check_library_exists(edit readline "" HAVE_EDIT)
find_opt_library_and_header(EDIT_PATH histedit.h EDIT_LIB edit HAVE_EDIT)

find_package(Gettext)           # Used for running tests

#if(GETTEXT_FOUND)
#  set(HAVE_GETTEXT 1)
#else()
  set(HAVE_GETTEXT 0)
#endif()

#find_path(INTL_PATH libintl.h)
#find_library(INTL_LIB intl)
#include_directories(SYSTEM "${INTL_PATH}")

########################################################################

macro(add_ledger_library_dependencies _target)
  if(BUILD_LIBRARY)
    target_link_libraries(${_target} libledger)
  endif()
  target_link_libraries(${_target} ${MPFR_LIB})
  target_link_libraries(${_target} ${GMP_LIB})
  if(HAVE_EDIT)
    target_link_libraries(${_target} ${EDIT_LIB})
  endif()
  if(HAVE_GETTEXT)
    target_link_libraries(${_target} ${INTL_LIB})
  endif()
  if(HAVE_BOOST_PYTHON)
    target_link_libraries(${_target} ${Boost_LIBRARIES} ${PYTHON_LIBRARIES})
  else()
    target_link_libraries(${_target} ${Boost_LIBRARIES})
  endif()
  if(HAVE_BOOST_REGEX_UNICODE)
    target_link_libraries(${_target} icuuc)
  endif()
endmacro(add_ledger_library_dependencies _target)

########################################################################

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR})

# add the binary tree to the search path for include files so that we will
# find TutorialConfig.h
include_directories("${PROJECT_SOURCE_DIR}/lib")
include_directories("${PROJECT_SOURCE_DIR}/lib/utfcpp/source")
include_directories("${PROJECT_BINARY_DIR}")

configure_file(
  ${PROJECT_SOURCE_DIR}/src/system.hh.in
  ${PROJECT_BINARY_DIR}/system.hh)

add_subdirectory(src)
if(BUILD_DOCS)
  add_subdirectory(doc)
endif()
if(BUILD_EMACSLISP)
  add_subdirectory(lisp)
endif()
add_subdirectory(test)

########################################################################

# build a CPack driven installer package
include (InstallRequiredSystemLibraries)

set (CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/doc/LICENSE.rtf")
set (CPACK_PACKAGE_VERSION_MAJOR "${Ledger_VERSION_MAJOR}")
set (CPACK_PACKAGE_VERSION_MINOR "${Ledger_VERSION_MINOR}")
set (CPACK_PACKAGE_VERSION_PATCH "${Ledger_VERSION_PATCH}")

include (CPack)

### CMakeLists.txt ends here