#
# Copyright (c) 1994 by Xerox Corporation.  All rights reserved.
# Copyright (c) 1996 by Silicon Graphics.  All rights reserved.
# Copyright (c) 1998 by Fergus Henderson.  All rights reserved.
# Copyright (c) 2000-2010 by Hewlett-Packard Company.  All rights reserved.
# Copyright (c) 2010-2024 Ivan Maidanski
##
# THIS MATERIAL IS PROVIDED AS IS, WITH ABSOLUTELY NO WARRANTY EXPRESSED
# OR IMPLIED.  ANY USE IS AT YOUR OWN RISK.
##
# Permission is hereby granted to use or copy this program
# for any purpose,  provided the above notices are retained on all copies.
# Permission to modify the code and to distribute modified code is granted,
# provided the above notices are retained, and a notice that the code was
# modified is included with the above copyright notice.
##

#
#  get cmake and run:
#    cmake -G "Visual Studio 8 2005"
#  in the same dir as this file
#  this will generate gc.sln
#

cmake_minimum_required(VERSION 3.10)

set(PACKAGE_VERSION 8.2.12)
# Version must match that in AC_INIT of configure.ac and that in README.
# Version must conform to: [0-9]+[.][0-9]+[.][0-9]+

# Info (current:revision:age) for the Libtool versioning system.
# These values should match those in cord/cord.am and Makefile.am.
set(LIBCORD_VER_INFO    6:1:5)
set(LIBGC_VER_INFO      6:6:5)
set(LIBGCCPP_VER_INFO   6:0:5)

option(enable_cplusplus "C++ support" OFF)
if (enable_cplusplus)
  project(gc)
else()
  project(gc C)
endif()

if (POLICY CMP0057)
  # Required for CheckLinkerFlag, at least.
  cmake_policy(SET CMP0057 NEW)
endif()

include(CheckCCompilerFlag)
include(CheckCSourceCompiles)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckSymbolExists)
include(CMakePackageConfigHelpers)
include(CTest)
include(GNUInstallDirs)

if (NOT (${CMAKE_VERSION} VERSION_LESS "3.18.0"))
  include(CheckLinkerFlag)
endif()

# Customize the build by passing "-D<option_name>=ON|OFF" in the command line.
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(build_cord "Build cord library" ON)
option(build_tests "Build tests" OFF)
option(enable_docs "Build and install documentation" ON)
option(enable_threads "Support threads" ON)
option(enable_parallel_mark "Parallelize marking and free list construction" ON)
option(enable_thread_local_alloc "Turn on thread-local allocation optimization" ON)
option(enable_threads_discovery "Enable threads discovery in GC" ON)
option(enable_throw_bad_alloc_library "Turn on C++ gctba library build" ON)
option(enable_gcj_support "Support for gcj" ON)
option(enable_sigrt_signals "Use SIGRTMIN-based signals for thread suspend/resume" OFF)
option(enable_gc_debug "Support for pointer back-tracing" OFF)
option(disable_gc_debug "Disable debugging like GC_dump and its callees" OFF)
option(enable_java_finalization "Support for java finalization" ON)
option(enable_atomic_uncollectable "Support for atomic uncollectible allocation" ON)
option(enable_redirect_malloc "Redirect malloc and friends to GC routines" OFF)
option(enable_disclaim "Support alternative finalization interface" ON)
option(enable_large_config "Optimize for large heap or root set" OFF)
option(enable_gc_assertions "Enable collector-internal assertion checking" OFF)
option(enable_mmap "Use mmap instead of sbrk to expand the heap" OFF)
option(enable_munmap "Return page to the OS if empty for N collections" ON)
option(enable_dynamic_loading "Enable tracing of dynamic library data roots" ON)
option(enable_register_main_static_data "Perform the initial guess of data root sets" ON)
option(enable_checksums "Report erroneously cleared dirty bits" OFF)
option(enable_werror "Pass -Werror to the C compiler (treat warnings as errors)" OFF)
option(enable_single_obj_compilation "Compile all libgc source files into single .o" OFF)
option(enable_handle_fork "Attempt to ensure a usable collector after fork()" ON)
option(disable_handle_fork "Prohibit installation of pthread_atfork() handlers" OFF)
option(enable_emscripten_asyncify "Use Emscripten asyncify feature" OFF)
option(install_headers "Install header and pkg-config metadata files" ON)
option(without_libatomic_ops "Use atomic_ops.h in libatomic_ops/src" OFF)

# Override the default build type to RelWithDebInfo (this instructs cmake to
# pass -O2 -g -DNDEBUG options to the compiler by default).
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE
      STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
               STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")
endif()

# Convert VER_INFO values to [SO]VERSION ones.
if (BUILD_SHARED_LIBS)
  # cord:
  string(REGEX REPLACE "(.+):.+:.+"  "\\1" cord_cur ${LIBCORD_VER_INFO})
  string(REGEX REPLACE ".+:(.+):.+"  "\\1" cord_rev ${LIBCORD_VER_INFO})
  string(REGEX REPLACE ".+:.+:(.+)$" "\\1" cord_age ${LIBCORD_VER_INFO})
  math(EXPR CORD_SOVERSION "${cord_cur} - ${cord_age}")
  set(CORD_VERSION_PROP "${CORD_SOVERSION}.${cord_age}.${cord_rev}")
  message(STATUS "CORD_VERSION_PROP = ${CORD_VERSION_PROP}")
  # gc:
  string(REGEX REPLACE "(.+):.+:.+"  "\\1" gc_cur ${LIBGC_VER_INFO})
  string(REGEX REPLACE ".+:(.+):.+"  "\\1" gc_rev ${LIBGC_VER_INFO})
  string(REGEX REPLACE ".+:.+:(.+)$" "\\1" gc_age ${LIBGC_VER_INFO})
  math(EXPR GC_SOVERSION "${gc_cur} - ${gc_age}")
  set(GC_VERSION_PROP "${GC_SOVERSION}.${gc_age}.${gc_rev}")
  message(STATUS "GC_VERSION_PROP = ${GC_VERSION_PROP}")
  # gccpp and gctba:
  string(REGEX REPLACE "(.+):.+:.+"  "\\1" gccpp_cur ${LIBGCCPP_VER_INFO})
  string(REGEX REPLACE ".+:(.+):.+"  "\\1" gccpp_rev ${LIBGCCPP_VER_INFO})
  string(REGEX REPLACE ".+:.+:(.+)$" "\\1" gccpp_age ${LIBGCCPP_VER_INFO})
  math(EXPR GCCPP_SOVERSION "${gccpp_cur} - ${gccpp_age}")
  set(GCCPP_VERSION_PROP "${GCCPP_SOVERSION}.${gccpp_age}.${gccpp_rev}")
  message(STATUS "GCCPP_VERSION_PROP = ${GCCPP_VERSION_PROP}")
endif(BUILD_SHARED_LIBS)

# Compile definitions, compile options and include directories collected to be
# applied to all the targets of this project.
set(GC_DEFINITIONS)
set(GC_COMPILE_OPTIONS)
set(GC_INCLUDE_DIRS)

list(APPEND GC_DEFINITIONS ALL_INTERIOR_POINTERS NO_EXECUTE_PERMISSION)

# Set struct packing alignment to word (instead of 1-byte).
if (BORLAND)
  list(APPEND GC_COMPILE_OPTIONS /a4)
elseif (WATCOM)
  list(APPEND GC_COMPILE_OPTIONS /zp4)
endif()

# Output all warnings.
if (BORLAND)
  # All warnings except for particular ones.
  list(APPEND GC_COMPILE_OPTIONS
       /w /w-aus /w-ccc /w-inl /w-par /w-pro /w-rch /w-use)
elseif (MSVC)
  # All warnings but ignoring "unreferenced formal parameter" and
  # "conditional expression is constant" ones.
  list(APPEND GC_COMPILE_OPTIONS /W4 /wd4100 /wd4127)
elseif (WATCOM)
  list(APPEND GC_COMPILE_OPTIONS /wx)
  if (enable_gc_assertions)
    # Suppress "unreachable code" warning in GC_ASSERT() if some constant
    # non-zero expression is given as the argument.
    list(APPEND GC_COMPILE_OPTIONS /wcd=201)
  endif()
  if (enable_threads)
    # Suppress "missing return value" wcc warning for AO_test_and_set_full()
    # and AO_char_fetch_and_add_full() in AO msftc/x86.h file.
    list(APPEND GC_COMPILE_OPTIONS $<$<COMPILE_LANGUAGE:C>:/wcd=107>)
  endif(enable_threads)
else()
  # TODO add -[W]pedantic -Wno-long-long
  list(APPEND GC_COMPILE_OPTIONS -Wall -Wextra)
endif()

if (WIN32)
  # Disable MS crt security warnings reported e.g. for getenv, strcpy.
  list(APPEND GC_DEFINITIONS _CRT_SECURE_NO_DEPRECATE)
endif()

list(APPEND GC_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/include")

set(SRC alloc.c reclaim.c allchblk.c misc.c mach_dep.c os_dep.c
        mark_rts.c headers.c mark.c obj_map.c blacklst.c finalize.c
        new_hblk.c dbg_mlc.c malloc.c dyn_load.c typd_mlc.c ptr_chck.c
        mallocx.c)
set(THREADDLLIBS_LIST)
set(NEED_LIB_RT)

set(_HOST ${CMAKE_SYSTEM_PROCESSOR}-unknown-${CMAKE_SYSTEM})
string(TOLOWER ${_HOST} HOST)
message(STATUS "TARGET = ${HOST}")

if (enable_threads)
  find_package(Threads REQUIRED)
  message(STATUS "Thread library: ${CMAKE_THREAD_LIBS_INIT}")
  if (without_libatomic_ops OR BORLAND OR MSVC OR WATCOM)
    list(APPEND GC_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/libatomic_ops/src")
    # Note: alternatively, use CFLAGS_EXTRA to pass -I<...>/libatomic_ops/src.
  else()
    # Assume the compiler supports GCC atomic intrinsics.
    list(APPEND GC_DEFINITIONS GC_BUILTIN_ATOMIC)
  endif()
  list(APPEND GC_INCLUDE_DIRS ${Threads_INCLUDE_DIR})
  set(THREADDLLIBS_LIST ${CMAKE_THREAD_LIBS_INIT})
  if (${CMAKE_DL_LIBS} MATCHES ^[^-].*)
    # Some cmake versions have a broken non-empty CMAKE_DL_LIBS omitting "-l".
    # Assume CMAKE_DL_LIBS contains just one library.
    set(THREADDLLIBS_LIST ${THREADDLLIBS_LIST} -l${CMAKE_DL_LIBS})
  else()
    set(THREADDLLIBS_LIST ${THREADDLLIBS_LIST} ${CMAKE_DL_LIBS})
  endif()
  if (BORLAND)
    # Workaround "cannot locate assembly file" and "out of hash space"
    # compilation errors, "restarting compile using assembly" warning.
    list(APPEND GC_DEFINITIONS AO_NO_ASM_XADD AO_NO_ASM_XCHG)
  endif()
endif(enable_threads)

set(ATOMIC_OPS_LIBS "") # TODO: Assume libatomic_ops library is not needed.

# Thread support detection.
if (CMAKE_USE_PTHREADS_INIT)
  set(SRC ${SRC} gc_dlopen.c)
  if (CYGWIN OR WIN32)
    set(SRC ${SRC} win32_threads.c)
  else()
    set(SRC ${SRC} pthread_start.c pthread_stop_world.c pthread_support.c)
  endif()
  if (APPLE)
    set(SRC ${SRC} darwin_stop_world.c)
  endif()
  if (HOST MATCHES .*-.*-hpux10.*)
    message(FATAL_ERROR "HP/UX 10 POSIX threads are not supported.")
  endif()
  # Common defines for POSIX platforms.
  list(APPEND GC_DEFINITIONS GC_THREADS _REENTRANT)
  if (enable_parallel_mark)
    list(APPEND GC_DEFINITIONS PARALLEL_MARK)
  endif()
  if (enable_thread_local_alloc)
    list(APPEND GC_DEFINITIONS THREAD_LOCAL_ALLOC)
    set(SRC ${SRC} specific.c thread_local_alloc.c)
  endif()
  message("Explicit GC_INIT() calls may be required.")
  if (HOST MATCHES .*-.*-hpux11.*)
    message("Only HP/UX 11 POSIX threads are supported.")
    list(APPEND GC_DEFINITIONS _POSIX_C_SOURCE=199506L)
    set(NEED_LIB_RT ON)
  elseif (HOST MATCHES .*-.*-netbsd.*)
    message("Only on NetBSD 2.0 or later.")
    list(APPEND GC_DEFINITIONS _PTHREADS)
    set(NEED_LIB_RT ON)
  endif()
  if (WIN32) # AND NOT CYGWIN
    # Does not provide process fork functionality.
  elseif (enable_handle_fork AND NOT disable_handle_fork)
    list(APPEND GC_DEFINITIONS HANDLE_FORK)
  endif()
  if (enable_sigrt_signals)
    list(APPEND GC_DEFINITIONS GC_USESIGRT_SIGNALS)
  endif()
elseif (CMAKE_USE_WIN32_THREADS_INIT)
  list(APPEND GC_DEFINITIONS GC_THREADS)
  if (enable_parallel_mark)
    list(APPEND GC_DEFINITIONS PARALLEL_MARK)
  endif()
  if (enable_thread_local_alloc AND (enable_parallel_mark OR NOT BUILD_SHARED_LIBS))
    # Imply THREAD_LOCAL_ALLOC unless GC_DLL.
    list(APPEND GC_DEFINITIONS THREAD_LOCAL_ALLOC)
    set(SRC ${SRC} thread_local_alloc.c)
  endif()
  list(APPEND GC_DEFINITIONS EMPTY_GETENV_RESULTS)
  set(SRC ${SRC} win32_threads.c)
elseif (CMAKE_HP_PTHREADS_INIT OR CMAKE_USE_SPROC_INIT)
  message(FATAL_ERROR "Unsupported thread package")
endif()

# Check whether -lrt linker option is needed to use clock_gettime.
if (NOT NEED_LIB_RT)
  check_function_exists(clock_gettime HAVE_CLOCK_GETTIME_DIRECTLY)
  if (NOT HAVE_CLOCK_GETTIME_DIRECTLY)
    # Use of clock_gettime() probably requires linking with "rt" library.
    set(NEED_LIB_RT ON)
  endif()
endif()

# Locate and use "rt" library if needed (and the library is available).
if (NEED_LIB_RT)
  find_library(LIBRT rt)
  if (LIBRT)
    set(THREADDLLIBS_LIST ${THREADDLLIBS_LIST} ${LIBRT})
  endif()
endif(NEED_LIB_RT)

if (disable_handle_fork)
  list(APPEND GC_DEFINITIONS NO_HANDLE_FORK)
endif()

if (enable_gcj_support)
  list(APPEND GC_DEFINITIONS GC_GCJ_SUPPORT)
  if (enable_threads AND NOT (enable_thread_local_alloc AND HOST MATCHES .*-.*-kfreebsd.*-gnu))
    # FIXME: For a reason, gctest hangs up on kFreeBSD if both of
    # THREAD_LOCAL_ALLOC and GC_ENABLE_SUSPEND_THREAD are defined.
    list(APPEND GC_DEFINITIONS GC_ENABLE_SUSPEND_THREAD)
  endif()
  set(SRC ${SRC} gcj_mlc.c)
endif(enable_gcj_support)

if (enable_disclaim)
  list(APPEND GC_DEFINITIONS ENABLE_DISCLAIM)
  set(SRC ${SRC} fnlz_mlc.c)
endif()

if (enable_java_finalization)
  list(APPEND GC_DEFINITIONS JAVA_FINALIZATION)
endif()

if (enable_atomic_uncollectable)
  list(APPEND GC_DEFINITIONS GC_ATOMIC_UNCOLLECTABLE)
endif()

if (enable_gc_debug)
  list(APPEND GC_DEFINITIONS DBG_HDRS_ALL KEEP_BACK_PTRS)
  if (HOST MATCHES i.86-.*-dgux.*|ia64-.*-linux.*|i586-.*-linux.*|i686-.*-linux.*|x86-.*-linux.*|x86_64-.*-linux.*)
    list(APPEND GC_DEFINITIONS MAKE_BACK_GRAPH)
    if (HOST MATCHES .*-.*-.*linux.*)
      list(APPEND GC_DEFINITIONS SAVE_CALL_COUNT=8)
    endif()
    set(SRC ${SRC} backgraph.c)
  endif()
endif(enable_gc_debug)

if (disable_gc_debug)
  list(APPEND GC_DEFINITIONS NO_DEBUGGING)
elseif (WINCE)
  # Read environment variables from "<program>.gc.env" file.
  list(APPEND GC_DEFINITIONS GC_READ_ENV_FILE)
endif()

if (enable_redirect_malloc)
  if (enable_gc_debug)
    list(APPEND GC_DEFINITIONS REDIRECT_MALLOC=GC_debug_malloc_replacement)
    list(APPEND GC_DEFINITIONS REDIRECT_REALLOC=GC_debug_realloc_replacement)
    list(APPEND GC_DEFINITIONS REDIRECT_FREE=GC_debug_free)
  else()
    list(APPEND GC_DEFINITIONS REDIRECT_MALLOC=GC_malloc)
  endif()
  if (WIN32)
    list(APPEND GC_DEFINITIONS REDIRECT_MALLOC_IN_HEADER)
  else()
    list(APPEND GC_DEFINITIONS GC_USE_DLOPEN_WRAP)
  endif()
endif(enable_redirect_malloc)

if (enable_munmap)
  list(APPEND GC_DEFINITIONS USE_MMAP USE_MUNMAP)
elseif (enable_mmap)
  list(APPEND GC_DEFINITIONS USE_MMAP)
endif()

if (NOT enable_dynamic_loading)
  list(APPEND GC_DEFINITIONS IGNORE_DYNAMIC_LOADING)
endif()

if (NOT enable_register_main_static_data)
  list(APPEND GC_DEFINITIONS GC_DONT_REGISTER_MAIN_STATIC_DATA)
endif()

if (enable_large_config)
  list(APPEND GC_DEFINITIONS LARGE_CONFIG)
endif()

if (enable_gc_assertions)
  list(APPEND GC_DEFINITIONS GC_ASSERTIONS)
endif()

if (NOT enable_threads_discovery)
  list(APPEND GC_DEFINITIONS GC_NO_THREADS_DISCOVERY)
endif()

if (enable_checksums)
  if (enable_munmap OR enable_threads)
    message(FATAL_ERROR "CHECKSUMS not compatible with USE_MUNMAP or threads")
  endif()
  list(APPEND GC_DEFINITIONS CHECKSUMS)
  set(SRC ${SRC} checksums.c)
endif(enable_checksums)

if (enable_werror)
  if (BORLAND)
    list(APPEND GC_COMPILE_OPTIONS /w!)
  elseif (MSVC)
    list(APPEND GC_COMPILE_OPTIONS /WX)
    # Workaround "typedef ignored on left of ..." warning reported in
    # imagehlp.h of e.g. Windows Kit 8.1.
    list(APPEND GC_COMPILE_OPTIONS /wd4091)
  elseif (WATCOM)
    list(APPEND GC_COMPILE_OPTIONS /we)
  else()
    list(APPEND GC_COMPILE_OPTIONS -Werror)
  endif()
endif(enable_werror)

if (enable_single_obj_compilation OR BUILD_SHARED_LIBS)
  set(SRC extra/gc.c) # override SRC
  if (CMAKE_USE_PTHREADS_INIT AND NOT (CYGWIN OR WIN32))
    list(APPEND GC_DEFINITIONS GC_PTHREAD_START_STANDALONE)
    set(SRC ${SRC} pthread_start.c)
  endif()
endif()

# Add implementation of backtrace() and backtrace_symbols().
if (MSVC)
  set(SRC ${SRC} extra/msvc_dbg.c)
endif()

# Declare that the libraries do not refer to external symbols.
if (BUILD_SHARED_LIBS AND NOT (APPLE OR ${CMAKE_SYSTEM_NAME} MATCHES "BSD"))
  # Note: performed before CMAKE_REQUIRED_FLAGS is updated with "-c".
  if (${CMAKE_VERSION} VERSION_LESS "3.18.0")
    set(WL_NO_UNDEFINED_OPT "-Wl,--no-undefined")
    check_c_compiler_flag(${WL_NO_UNDEFINED_OPT} HAVE_FLAG_WL_NO_UNDEFINED)
  else()
    set(WL_NO_UNDEFINED_OPT "LINKER:--no-undefined")
    check_linker_flag(C "${WL_NO_UNDEFINED_OPT}" HAVE_FLAG_WL_NO_UNDEFINED)
  endif()
endif()

# Instruct check_c_source_compiles to skip linking.
# Alternatively, we could set CMAKE_REQUIRED_LIBRARIES properly.
SET(CMAKE_REQUIRED_FLAGS "-c")

# Instruct check_c_source_compiles and similar CMake checks not to ignore
# compiler warnings (like "implicit declaration of function").
if (NOT BORLAND AND NOT MSVC AND NOT WATCOM)
  check_c_compiler_flag(-Werror HAVE_FLAG_WERROR)
  if (HAVE_FLAG_WERROR)
    check_c_compiler_flag(-Wno-unused-command-line-argument
                          HAVE_FLAG_WNO_UNUSED_CMDLINE_ARG)
    SET(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -Werror")
    # Prevent "linker input unused" error in further check_c_compiler_flag.
    if (HAVE_FLAG_WNO_UNUSED_CMDLINE_ARG)
      SET(CMAKE_REQUIRED_FLAGS
          "${CMAKE_REQUIRED_FLAGS} -Wno-unused-command-line-argument")
    endif()
  endif(HAVE_FLAG_WERROR)
endif()

if (BUILD_SHARED_LIBS)
  list(APPEND GC_DEFINITIONS GC_DLL)
  # Pass -fvisibility=hidden option if supported.
  check_c_compiler_flag(-fvisibility=hidden HAVE_FLAG_F_VISIBILITY_HIDDEN)
  if (HAVE_FLAG_F_VISIBILITY_HIDDEN)
    list(APPEND GC_DEFINITIONS GC_VISIBILITY_HIDDEN_SET)
    list(APPEND GC_COMPILE_OPTIONS -fvisibility=hidden)
  else()
    list(APPEND GC_DEFINITIONS GC_NO_VISIBILITY)
  endif()
else()
  list(APPEND GC_DEFINITIONS GC_NOT_DLL)
  if (WIN32)
    # Do not require the clients to link with "user32" system library.
    list(APPEND GC_DEFINITIONS DONT_USE_USER32_DLL)
  endif(WIN32)
endif()

# Disable strict aliasing optimizations.
# It could re-enabled back by a flag passed in CFLAGS_EXTRA.
check_c_compiler_flag(-fno-strict-aliasing HAVE_FLAG_F_NO_STRICT_ALIASING)
if (HAVE_FLAG_F_NO_STRICT_ALIASING)
  list(APPEND GC_COMPILE_OPTIONS -fno-strict-aliasing)
endif()

# Prevent "__builtin_return_address with a nonzero argument is unsafe" warning.
if (NOT BORLAND AND NOT MSVC AND NOT WATCOM)
  check_c_compiler_flag(-Wno-frame-address HAVE_FLAG_WNO_FRAME_ADDRESS)
  if (HAVE_FLAG_WNO_FRAME_ADDRESS)
    list(APPEND GC_COMPILE_OPTIONS -Wno-frame-address)
  endif(HAVE_FLAG_WNO_FRAME_ADDRESS)
endif()

# Extra user-defined flags to pass both to C and C++ compilers.
if (DEFINED CFLAGS_EXTRA)
  separate_arguments(CFLAGS_EXTRA_LIST UNIX_COMMAND "${CFLAGS_EXTRA}")
  list(APPEND GC_COMPILE_OPTIONS ${CFLAGS_EXTRA_LIST})
endif()

# Check whether execinfo.h header file is present.
check_include_file(execinfo.h HAVE_EXECINFO_H)
if (NOT HAVE_EXECINFO_H)
  list(APPEND GC_DEFINITIONS GC_MISSING_EXECINFO_H)
endif()

# Check for getcontext (uClibc can be configured without it, for example).
check_function_exists(getcontext HAVE_GETCONTEXT)
if (HAVE_GETCONTEXT AND NOT APPLE)
  # Double check getcontext() is available (needed at least on OpenBSD 7.3).
  # Note: OS X is excluded here because the header filename differs.
  check_c_source_compiles("
#include <ucontext.h>\n
int main(void) { ucontext_t ctxt; (void)getcontext(&ctxt); return 0; }"
    HAVE_GETCONTEXT_FUNC)
  if (NOT HAVE_GETCONTEXT_FUNC)
    set(HAVE_GETCONTEXT OFF)
  endif()
endif()
if (NOT HAVE_GETCONTEXT)
  list(APPEND GC_DEFINITIONS NO_GETCONTEXT)
endif()

# Check whether dl_iterate_phdr exists (as a strong symbol).
check_function_exists(dl_iterate_phdr HAVE_DL_ITERATE_PHDR)
if (HAVE_DL_ITERATE_PHDR)
  list(APPEND GC_DEFINITIONS HAVE_DL_ITERATE_PHDR)
endif()

check_symbol_exists(sigsetjmp setjmp.h HAVE_SIGSETJMP)
if (NOT HAVE_SIGSETJMP)
  list(APPEND GC_DEFINITIONS GC_NO_SIGSETJMP)
endif()

# pthread_setname_np, if available, may have 1, 2 or 3 arguments.
if (CMAKE_USE_PTHREADS_INIT OR APPLE)
  check_c_source_compiles("
#define _GNU_SOURCE 1\n
#include <pthread.h>\n
int main(void) { (void)pthread_setname_np(\"thread-name\"); return 0; }"
    HAVE_PTHREAD_SETNAME_NP_WITHOUT_TID)
  if (HAVE_PTHREAD_SETNAME_NP_WITHOUT_TID)
    # Define to use 'pthread_setname_np(const char*)' function.
    list(APPEND GC_DEFINITIONS HAVE_PTHREAD_SETNAME_NP_WITHOUT_TID)
  else()
    check_c_source_compiles("
#define _GNU_SOURCE 1\n
#include <pthread.h>\n
int main(void) {\n
  (void)pthread_setname_np(pthread_self(), \"thread-name-%u\", 0); return 0; }"
      HAVE_PTHREAD_SETNAME_NP_WITH_TID_AND_ARG)
    if (HAVE_PTHREAD_SETNAME_NP_WITH_TID_AND_ARG)
      # Define to use 'pthread_setname_np(pthread_t, const char*, void *)'.
      list(APPEND GC_DEFINITIONS HAVE_PTHREAD_SETNAME_NP_WITH_TID_AND_ARG)
    else()
      check_c_source_compiles("
#define _GNU_SOURCE 1\n
#include <pthread.h>\n
int main(void) {\n
  (void)pthread_setname_np(pthread_self(), \"thread-name\"); return 0; }"
        HAVE_PTHREAD_SETNAME_NP_WITH_TID)
      if (HAVE_PTHREAD_SETNAME_NP_WITH_TID)
        # Define to use 'pthread_setname_np(pthread_t, const char*)' function.
        list(APPEND GC_DEFINITIONS HAVE_PTHREAD_SETNAME_NP_WITH_TID)
      endif()
    endif(HAVE_PTHREAD_SETNAME_NP_WITH_TID_AND_ARG)
  endif(HAVE_PTHREAD_SETNAME_NP_WITHOUT_TID)
endif()

# Check for dladdr (used for debugging).
check_c_source_compiles("
#define _GNU_SOURCE 1\n
#include <dlfcn.h>\n
int main(void) { Dl_info info; (void)dladdr(\"\", &info); return 0; }"
  HAVE_DLADDR)
if (HAVE_DLADDR)
  # Define to use 'dladdr' function.
  list(APPEND GC_DEFINITIONS HAVE_DLADDR)
endif()

# Check for emscripten; use asyncify feature if requested.
check_c_source_compiles("
#ifndef __EMSCRIPTEN__\n
# error This is not Emscripten\n
#endif\n
int main(void) { return 0; }"
  EMSCRIPTEN)
if (EMSCRIPTEN AND enable_emscripten_asyncify)
  # Use this option if your program is targeting -sASYNCIFY.  The latter is
  # required to scan the stack, ASYNCIFY_STACK_SIZE is probably needed for
  # gctest only.
  list(APPEND GC_DEFINITIONS EMSCRIPTEN_ASYNCIFY)
  set(CMAKE_EXE_LINKER_FLAGS
        "${CMAKE_EXE_LINKER_FLAGS} -sASYNCIFY -sASYNCIFY_STACK_SIZE=128000")
endif()

add_library(gc ${SRC})
add_library(bdwgc::gc ALIAS gc)
target_link_libraries(gc PRIVATE ${THREADDLLIBS_LIST} ${ATOMIC_OPS_LIBS})
target_include_directories(gc INTERFACE
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<INSTALL_INTERFACE:include>")

# Apply the compile definitions, compile options and include directories
# collected above to the exported gc library target (same for gccpp,
# gctba and cord below).
# Note: linking the exported library target with gc_build_flags is not
# possible as otherwise it would require the latter to be present in the
# BDWgcTargets export set as long as a static library could be built.
target_compile_definitions(gc PRIVATE ${GC_DEFINITIONS})
target_compile_options(gc PRIVATE ${GC_COMPILE_OPTIONS})
target_include_directories(gc PRIVATE ${GC_INCLUDE_DIRS})

if (enable_cplusplus)
  add_library(gccpp gc_badalc.cc gc_cpp.cc)
  add_library(bdwgc::gccpp ALIAS gccpp)
  target_link_libraries(gccpp PRIVATE gc)
  target_include_directories(gccpp INTERFACE
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<INSTALL_INTERFACE:include>")
  target_compile_definitions(gccpp PRIVATE ${GC_DEFINITIONS})
  target_compile_options(gccpp PRIVATE ${GC_COMPILE_OPTIONS})
  target_include_directories(gccpp PRIVATE ${GC_INCLUDE_DIRS})
  if (enable_throw_bad_alloc_library)
    # The same as gccpp but contains only gc_badalc.
    add_library(gctba gc_badalc.cc)
    add_library(bdwgc::gctba ALIAS gctba)
    target_link_libraries(gctba PRIVATE gc)
    target_include_directories(gctba INTERFACE
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<INSTALL_INTERFACE:include>")
    target_compile_definitions(gctba PRIVATE ${GC_DEFINITIONS})
    target_compile_options(gctba PRIVATE ${GC_COMPILE_OPTIONS})
    target_include_directories(gctba PRIVATE ${GC_INCLUDE_DIRS})
  endif(enable_throw_bad_alloc_library)
endif()

if (build_cord)
  set(CORD_SRC cord/cordbscs.c cord/cordprnt.c cord/cordxtra.c)
  add_library(cord ${CORD_SRC})
  add_library(bdwgc::cord ALIAS cord)
  target_link_libraries(cord PRIVATE gc)
  target_include_directories(cord INTERFACE
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<INSTALL_INTERFACE:include>")
  target_compile_definitions(cord PRIVATE ${GC_DEFINITIONS})
  target_compile_options(cord PRIVATE ${GC_COMPILE_OPTIONS})
  target_include_directories(cord PRIVATE ${GC_INCLUDE_DIRS})
  if (BUILD_SHARED_LIBS)
    set_property(TARGET cord PROPERTY VERSION ${CORD_VERSION_PROP})
    set_property(TARGET cord PROPERTY SOVERSION ${CORD_SOVERSION})
  endif()
  install(TARGETS cord EXPORT BDWgcTargets
          LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
          ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
          RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
          INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
endif(build_cord)

if (BUILD_SHARED_LIBS AND HAVE_FLAG_WL_NO_UNDEFINED)
  # Declare that the libraries do not refer to external symbols.
  if (${CMAKE_VERSION} VERSION_LESS "3.13.0")
    target_link_libraries(gc PRIVATE ${WL_NO_UNDEFINED_OPT})
    if (enable_cplusplus)
      target_link_libraries(gccpp PRIVATE ${WL_NO_UNDEFINED_OPT})
      if (enable_throw_bad_alloc_library)
        target_link_libraries(gctba PRIVATE ${WL_NO_UNDEFINED_OPT})
      endif(enable_throw_bad_alloc_library)
    endif(enable_cplusplus)
    if (build_cord)
      target_link_libraries(cord PRIVATE ${WL_NO_UNDEFINED_OPT})
    endif(build_cord)
  else()
    target_link_options(gc PRIVATE ${WL_NO_UNDEFINED_OPT})
    if (enable_cplusplus)
      target_link_options(gccpp PRIVATE ${WL_NO_UNDEFINED_OPT})
      if (enable_throw_bad_alloc_library)
        target_link_options(gctba PRIVATE ${WL_NO_UNDEFINED_OPT})
      endif(enable_throw_bad_alloc_library)
    endif(enable_cplusplus)
    if (build_cord)
      target_link_options(cord PRIVATE ${WL_NO_UNDEFINED_OPT})
    endif(build_cord)
  endif()
endif()

if (BUILD_SHARED_LIBS)
  set_property(TARGET gc PROPERTY VERSION ${GC_VERSION_PROP})
  set_property(TARGET gc PROPERTY SOVERSION ${GC_SOVERSION})
endif()
install(TARGETS gc EXPORT BDWgcTargets
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
        INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")

if (enable_cplusplus)
  if (BUILD_SHARED_LIBS)
    set_property(TARGET gccpp PROPERTY VERSION ${GCCPP_VERSION_PROP})
    set_property(TARGET gccpp PROPERTY SOVERSION ${GCCPP_SOVERSION})
  endif()
  install(TARGETS gccpp EXPORT BDWgcTargets
          LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
          ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
          RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
          INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  if (enable_throw_bad_alloc_library)
    if (BUILD_SHARED_LIBS)
      set_property(TARGET gctba PROPERTY VERSION ${GCCPP_VERSION_PROP})
      set_property(TARGET gctba PROPERTY SOVERSION ${GCCPP_SOVERSION})
    endif()
    install(TARGETS gctba EXPORT BDWgcTargets
            LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
            ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
            RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
            INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  endif(enable_throw_bad_alloc_library)
endif(enable_cplusplus)

if (install_headers)
  install(FILES include/gc.h
                include/gc_backptr.h
                include/gc_config_macros.h
                include/gc_inline.h
                include/gc_mark.h
                include/gc_tiny_fl.h
                include/gc_typed.h
                include/gc_version.h
                include/javaxfc.h
                include/leak_detector.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/gc")
  install(FILES include/extra/gc.h DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  if (enable_cplusplus)
    install(FILES include/gc_allocator.h
                  include/gc_cpp.h
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/gc")
    install(FILES include/extra/gc_cpp.h
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  endif()
  if (enable_disclaim)
    install(FILES include/gc_disclaim.h
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/gc")
  endif()
  if (enable_gcj_support)
    install(FILES include/gc_gcj.h
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/gc")
  endif()
  if (enable_threads AND CMAKE_USE_PTHREADS_INIT)
    install(FILES include/gc_pthread_redirects.h
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/gc")
  endif()
  if (build_cord)
    install(FILES include/cord.h
                  include/cord_pos.h
                  include/ec.h
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/gc")
  endif()

  # Provide pkg-config metadata.
  set(prefix "${CMAKE_INSTALL_PREFIX}")
  set(exec_prefix \${prefix})
  set(includedir "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
  set(libdir "${CMAKE_INSTALL_FULL_LIBDIR}")
  string(REPLACE ";" " " THREADDLLIBS "${THREADDLLIBS_LIST}")
  # ATOMIC_OPS_LIBS, PACKAGE_VERSION are defined above.
  configure_file(bdw-gc.pc.in bdw-gc.pc @ONLY)
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/bdw-gc.pc"
          DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
endif(install_headers)

if (build_tests)
  # The interface target that supplies the compile definitions, compile
  # options and include directories collected above.
  add_library(gc_build_flags INTERFACE)
  target_compile_definitions(gc_build_flags INTERFACE ${GC_DEFINITIONS})
  target_compile_options(gc_build_flags INTERFACE ${GC_COMPILE_OPTIONS})
  target_include_directories(gc_build_flags INTERFACE ${GC_INCLUDE_DIRS})

  if (build_cord)
    add_executable(cordtest cord/tests/cordtest.c)
    target_link_libraries(cordtest PRIVATE cord gc gc_build_flags)
    add_test(NAME cordtest COMMAND cordtest)

    if (WIN32 AND NOT CYGWIN)
      if (NOT (CMAKE_C_COMPILER_ID STREQUAL "Clang"))
        # Workaround MS Clang failure to compile a resource file.
        set(DE_WIN_RC cord/tests/de_win.rc)
      endif()
      add_executable(de cord/tests/de.c cord/tests/de_win.c
                     ${DE_WIN_RC})
      set_target_properties(de PROPERTIES WIN32_EXECUTABLE TRUE)
      target_link_libraries(de PRIVATE cord gc gc_build_flags gdi32)
    endif()
  endif(build_cord)

  # Compile some tests as C++ to test extern "C" in header files.
  if (enable_cplusplus)
    if (BORLAND OR MSVC OR WATCOM)
      # WinMain-based test hangs at startup if compiled by VC as C++ code.
      set_source_files_properties(tests/leak_test.c PROPERTIES LANGUAGE CXX)
    else()
      # The -x c++ compile option is to workaround
      # "treating c input as c++ when in C++ mode" Clang warning.
      set_source_files_properties(tests/test.c tests/leak_test.c
                                  PROPERTIES LANGUAGE CXX
                                  COMPILE_OPTIONS "-x;c++")
    endif()
  endif(enable_cplusplus)

  add_executable(gctest WIN32 tests/test.c)
  target_link_libraries(gctest PRIVATE gc gc_build_flags ${THREADDLLIBS_LIST})
  add_test(NAME gctest COMMAND gctest)
  if (WATCOM AND NOT enable_gc_assertions)
    # Suppress "unreachable code" warning in GC_MALLOC_[ATOMIC_]WORDS().
    target_compile_options(gctest PRIVATE /wcd=201)
  endif()

  add_executable(hugetest tests/huge_test.c)
  target_link_libraries(hugetest PRIVATE gc gc_build_flags)
  add_test(NAME hugetest COMMAND hugetest)

  add_executable(leaktest tests/leak_test.c)
  target_link_libraries(leaktest PRIVATE gc gc_build_flags)
  add_test(NAME leaktest COMMAND leaktest)

  add_executable(middletest tests/middle.c)
  target_link_libraries(middletest PRIVATE gc gc_build_flags)
  add_test(NAME middletest COMMAND middletest)

  add_executable(realloc_test tests/realloc_test.c)
  target_link_libraries(realloc_test PRIVATE gc gc_build_flags)
  add_test(NAME realloc_test COMMAND realloc_test)

  add_executable(smashtest tests/smash_test.c)
  target_link_libraries(smashtest PRIVATE gc gc_build_flags)
  add_test(NAME smashtest COMMAND smashtest)

  if (NOT (BUILD_SHARED_LIBS AND WIN32))
    add_library(staticrootslib_test tests/staticrootslib.c)
    target_link_libraries(staticrootslib_test PRIVATE gc gc_build_flags)
    add_library(staticrootslib2_test tests/staticrootslib.c)
    target_compile_options(staticrootslib2_test PRIVATE "-DSTATICROOTSLIB2")
    target_link_libraries(staticrootslib2_test PRIVATE gc gc_build_flags)
    add_executable(staticrootstest tests/staticrootstest.c)
    target_compile_options(staticrootstest PRIVATE "-DSTATICROOTSLIB2")
    target_link_libraries(staticrootstest PRIVATE
                          gc gc_build_flags staticrootslib_test
                          staticrootslib2_test)
    add_test(NAME staticrootstest COMMAND staticrootstest)
  endif()

  if (enable_gc_debug)
    add_executable(tracetest tests/trace_test.c)
    target_link_libraries(tracetest PRIVATE gc gc_build_flags)
    add_test(NAME tracetest COMMAND tracetest)
  endif()

  if (enable_threads)
    add_executable(test_atomic_ops tests/test_atomic_ops.c)
    target_link_libraries(test_atomic_ops PRIVATE gc gc_build_flags)
    add_test(NAME test_atomic_ops COMMAND test_atomic_ops)

    add_executable(threadleaktest tests/thread_leak_test.c)
    target_link_libraries(threadleaktest
                          PRIVATE gc gc_build_flags ${THREADDLLIBS_LIST})
    add_test(NAME threadleaktest COMMAND threadleaktest)

    if (NOT WIN32)
      add_executable(threadkey_test tests/threadkey_test.c)
      target_link_libraries(threadkey_test
                            PRIVATE gc gc_build_flags ${THREADDLLIBS_LIST})
      add_test(NAME threadkey_test COMMAND threadkey_test)
    endif()

    add_executable(subthreadcreate_test tests/subthread_create.c)
    target_link_libraries(subthreadcreate_test
                          PRIVATE gc gc_build_flags ${THREADDLLIBS_LIST})
    add_test(NAME subthreadcreate_test COMMAND subthreadcreate_test)

    add_executable(initsecondarythread_test tests/initsecondarythread.c)
    target_link_libraries(initsecondarythread_test
                          PRIVATE gc gc_build_flags ${THREADDLLIBS_LIST})
    add_test(NAME initsecondarythread_test COMMAND initsecondarythread_test)
  endif(enable_threads)

  if (enable_cplusplus)
    add_executable(test_cpp WIN32 tests/test_cpp.cc)
    target_link_libraries(test_cpp PRIVATE gc gccpp gc_build_flags)
    add_test(NAME test_cpp COMMAND test_cpp)
  endif()

  if (enable_disclaim)
    add_executable(disclaim_bench tests/disclaim_bench.c)
    target_link_libraries(disclaim_bench PRIVATE gc gc_build_flags)
    add_test(NAME disclaim_bench COMMAND disclaim_bench)

    add_executable(disclaim_test tests/disclaim_test.c)
    target_link_libraries(disclaim_test
                          PRIVATE gc gc_build_flags ${THREADDLLIBS_LIST})
    add_test(NAME disclaim_test COMMAND disclaim_test)

    add_executable(disclaim_weakmap_test tests/disclaim_weakmap_test.c)
    target_link_libraries(disclaim_weakmap_test
                          PRIVATE gc gc_build_flags ${THREADDLLIBS_LIST})
    add_test(NAME disclaim_weakmap_test COMMAND disclaim_weakmap_test)
  endif()
endif(build_tests)

if (enable_docs)
  # Note: documentation which is not installed: README.QUICK
  install(FILES AUTHORS README.md
          DESTINATION "${CMAKE_INSTALL_DOCDIR}")
  install(DIRECTORY doc/ DESTINATION "${CMAKE_INSTALL_DOCDIR}"
          FILES_MATCHING
            PATTERN "README.*"
            PATTERN "*.md")

  install(FILES doc/gc.man DESTINATION "${CMAKE_INSTALL_MANDIR}/man3"
          RENAME gc.3)
endif(enable_docs)

# CMake config/targets files.
install(EXPORT BDWgcTargets FILE BDWgcTargets.cmake
        NAMESPACE BDWgc:: DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/bdwgc")

configure_package_config_file("${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/BDWgcConfig.cmake"
        INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/bdwgc"
        NO_SET_AND_CHECK_MACRO)

write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/BDWgcConfigVersion.cmake"
        VERSION "${PACKAGE_VERSION}" COMPATIBILITY AnyNewerVersion)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/BDWgcConfig.cmake"
              "${CMAKE_CURRENT_BINARY_DIR}/BDWgcConfigVersion.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/bdwgc")

export(EXPORT BDWgcTargets
       FILE "${CMAKE_CURRENT_BINARY_DIR}/BDWgcTargets.cmake")
