# Project
cmake_minimum_required(VERSION 3.16 FATAL_ERROR)
project(libtgchat LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)

# Ccache
find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

# Compiler specifics
set(ATOMIC_TEST_SOURCE
  "
  #include <atomic>
  #include <cstdint>

  std::atomic<int> x;
  std::atomic<uint64_t> y(0);

  int main()
  {
    uint64_t i = y.load(std::memory_order_relaxed);
    return std::atomic_is_lock_free(&y) && std::atomic_is_lock_free(&x);
  }
  ")
check_cxx_source_compiles("${ATOMIC_TEST_SOURCE}" HAVE_BUILTIN_ATOMIC)
if (NOT HAVE_BUILTIN_ATOMIC)
  message(STATUS "No built-in atomic, using libatomic.")
  set (USE_LIBATOMIC TRUE)
else()
  message(STATUS "Using built-in atomic.")
endif()

# Platform specifics
if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
  add_definitions(-D_XOPEN_SOURCE_EXTENDED)
  list(APPEND CMAKE_PREFIX_PATH /opt/homebrew/opt/ncurses)
  list(APPEND CMAKE_PREFIX_PATH /usr/local/opt/ncurses)
  list(APPEND OPENSSL_ROOT_DIR /opt/homebrew/opt/openssl)
  list(APPEND OPENSSL_ROOT_DIR /usr/local/opt/openssl)
endif()

# Library
add_library(tgchat ${LIBRARY_LINKING_TYPE}
  src/tgchat.cpp
  src/tgchat.h
)
install(TARGETS tgchat DESTINATION ${CMAKE_INSTALL_LIBDIR})

# Common
target_common_config(tgchat)

# Headers
target_include_directories(tgchat PRIVATE "../../ext/apathy")
target_include_directories(tgchat PRIVATE "../common/src")
target_include_directories(tgchat PRIVATE "../ncutil/src")
target_include_directories(tgchat PRIVATE "ext/td")

# Dependency tdlib
add_subdirectory(ext/td EXCLUDE_FROM_ALL)

if(LIBRARY_LINKING_TYPE STREQUAL "SHARED")
  set(TDLIBNAME "tdclientshared")
  set(TDLIBFILENAME "libtdclientshared${CMAKE_SHARED_LIBRARY_SUFFIX}")
  install(FILES ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/${TDLIBFILENAME} DESTINATION ${CMAKE_INSTALL_LIBDIR})
else()
  set(TDLIBNAME "tdclient")
endif()

# Linking
target_link_libraries(tgchat PUBLIC ncutil ${TDLIBNAME})
if (USE_LIBATOMIC)
  set_target_properties(tgchat PROPERTIES LINK_FLAGS "-latomic")
endif()

# Compiler flags
set_target_properties(tgchat PROPERTIES COMPILE_FLAGS
                      "-Wall -Wextra -Wpedantic -Wshadow -Wpointer-arith \
                       -Wcast-qual -Wno-missing-braces -Wswitch-default \
                       -Wunreachable-code -Wundef -Wuninitialized \
                       -Wcast-align")
