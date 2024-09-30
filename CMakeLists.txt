# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# Read README.cmake before using this.

# CMAKE_MINIMUM_REQUIRED should be the first directive in the file:
# https://cmake.org/cmake/help/latest/command/cmake_minimum_required.html
CMAKE_MINIMUM_REQUIRED(VERSION 3.5)

PROJECT(APR C)

INCLUDE(CMakeDependentOption)

# Enable support for MSVC runtime library selection by abstraction
# if supported by CMake.
IF(POLICY CMP0091)
  CMAKE_POLICY(SET CMP0091 NEW)
ENDIF()

# Add simple support CMAKE_WARNING_AS_ERROR if CMake doesn't
# support it.
IF (CMAKE_VERSION VERSION_LESS 3.24)
  IF(CMAKE_WARNING_AS_ERROR AND MSVC)
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /WX")
  ENDIF()
ENDIF()

OPTION(APR_MINIMAL_BUILD    "Create minimal APR build"       OFF)
IF(NOT APR_MINIMAL_BUILD)
    OPTION(APU_HAVE_ODBC     "Build ODBC DBD driver"         ON)
ELSE()
    OPTION(APU_HAVE_ODBC     "Build ODBC DBD driver"         OFF)
ENDIF()

SET(apr_libname libapr-2)

SET(APR_XML_BACKEND "xmllite" CACHE STRING "XML library to use (expat|libxml|xmllite)")
SET_PROPERTY(CACHE APR_XML_BACKEND PROPERTY STRINGS expat libxml2 xmllite)

OPTION(APR_INSTALL_PRIVATE_H  "Install selected private .h files (for httpd)"  OFF)
OPTION(APU_HAVE_SQLITE3     "Build SQLite3 DBD driver"     OFF)
OPTION(APU_HAVE_PGSQL       "Build PostgreSQL DBD driver"  OFF)
OPTION(APU_HAVE_CRYPTO      "Crypto support"               OFF)
OPTION(APU_HAVE_ICONV       "Xlate support"                OFF)
OPTION(APR_HAVE_IPV6        "IPv6 support"                 ON)
OPTION(INSTALL_PDB          "Install .pdb files (if generated)"  ON)
OPTION(APR_BUILD_TESTAPR    "Build the test suite"         ON)
OPTION(BUILD_SHARED_LIBS    "Build using shared libraries" ON)
CMAKE_DEPENDENT_OPTION(APR_MODULAR_DSO "Use DSO build of modular components" ON "BUILD_SHARED_LIBS" OFF)
OPTION(APR_POOL_DEBUG       "Turn on pools debugging"      OFF)

SET(APR_INSTALL_BIN_DIR      "bin"                       CACHE STRING "Install subdirectory for binary files")
SET(APR_INSTALL_LIB_DIR      "lib"                       CACHE STRING "Install subdirectory for library files")
SET(APR_INSTALL_INCLUDE_DIR  "include/apr-2"             CACHE STRING "Install subdirectory for include files")

IF(APU_USE_EXPAT)
  MESSAGE(FATAL_ERROR "APU_USE_EXPAT option is not supported. Use APR_XML_BACKEND.")
ENDIF()

IF(APU_USE_LIBXML2)
  MESSAGE(FATAL_ERROR "APU_USE_LIBXML2 option is not supported. Use APR_XML_BACKEND.")
ENDIF()

IF(APU_USE_XMLLITE)
  MESSAGE(FATAL_ERROR "APU_USE_XMLLITE option is not supported. Use APR_XML_BACKEND.")
ENDIF()

IF(APR_MODULAR_DSO AND NOT BUILD_SHARED_LIBS)
  MESSAGE(FATAL_ERROR "APR_MODULAR_DSO requires shared libraries build option enabled.")
ENDIF()

IF(APU_HAVE_CRYPTO)
  FIND_PACKAGE(OpenSSL REQUIRED)
ENDIF()

IF (APR_XML_BACKEND STREQUAL "expat")
  FIND_PACKAGE(expat CONFIG REQUIRED)
  SET(XMLLIB_LIBRARIES   expat::expat)
  SET(APU_USE_EXPAT ON)
ELSEIF (APR_XML_BACKEND STREQUAL "libxml2")
  FIND_PACKAGE(LibXml2 REQUIRED)
  SET(XMLLIB_LIBRARIES   LibXml2::LibXml2)
  SET(APU_USE_LIBXML2 ON)
ELSEIF (APR_XML_BACKEND STREQUAL "xmllite")
  SET(XMLLIB_LIBRARIES   "xmllite.lib")
  SET(APU_USE_XMLLITE ON)
ELSE()
  MESSAGE(FATAL_ERROR "Unsupported APR_XML_BACKEND value (${APR_XML_BACKEND}).")
ENDIF()

IF(APU_HAVE_ICONV)
  FIND_PACKAGE(Iconv REQUIRED)
ENDIF()

IF(APU_HAVE_SQLITE3)
  FIND_PACKAGE(SQLite3 REQUIRED)
ENDIF()

IF(APU_HAVE_PGSQL)
  FIND_PACKAGE(PostgreSQL REQUIRED)
ENDIF()

IF(APR_POOL_DEBUG)
  ADD_COMPILE_DEFINITIONS(APR_POOL_DEBUG=1)
ENDIF()

IF(APR_MODULAR_DSO)
  ADD_COMPILE_DEFINITIONS(APR_HAVE_MODULAR_DSO=1)
ELSE()
  ADD_COMPILE_DEFINITIONS(APR_HAVE_MODULAR_DSO=0)
ENDIF()

# Read current version.
FILE(STRINGS include/apr_version.h VERSION_STRINGS REGEX "#define (APR_MAJOR_VERSION|APR_MINOR_VERSION|APR_PATCH_VERSION)")

STRING(REGEX REPLACE ".*#define APR_MAJOR_VERSION[ \t]+([0-9]+).*" "\\1" APR_MAJOR_VERSION ${VERSION_STRINGS})
STRING(REGEX REPLACE ".*#define APR_MINOR_VERSION[ \t]+([0-9]+).*" "\\1" APR_MINOR_VERSION ${VERSION_STRINGS})
STRING(REGEX REPLACE ".*#define APR_PATCH_VERSION[ \t]+([0-9]+).*" "\\1" APR_PATCH_VERSION ${VERSION_STRINGS})

CONFIGURE_FILE(include/apr.hwc
               ${PROJECT_BINARY_DIR}/apr.h)
# "COPYONLY" just because anything else isn't implemented ;)
CONFIGURE_FILE(include/private/apu_select_dbm.hw
               ${PROJECT_BINARY_DIR}/apu_select_dbm.h
               COPYONLY)
CONFIGURE_FILE(include/apu_want.hw
               ${PROJECT_BINARY_DIR}/apu_want.h
               COPYONLY)

ADD_EXECUTABLE(gen_test_char tools/gen_test_char.c)

ADD_CUSTOM_COMMAND(
  COMMENT "Generating character tables, apr_escape_test_char.h, for current locale"
  DEPENDS gen_test_char
  COMMAND $<TARGET_FILE:gen_test_char> > ${PROJECT_BINARY_DIR}/apr_escape_test_char.h
  OUTPUT ${PROJECT_BINARY_DIR}/apr_escape_test_char.h
)
ADD_CUSTOM_TARGET(
  test_char_header ALL
  DEPENDS ${PROJECT_BINARY_DIR}/apr_escape_test_char.h
)

IF(APU_HAVE_ICONV)
  SET(XLATE_LIBRARIES    Iconv::Iconv)
ELSE()
  SET(XLATE_LIBRARIES   "")
ENDIF()
# Generated .h files are stored in PROJECT_BINARY_DIR, not the
# source tree.
#
# BROKEN: not searching PROJECT_BINARY_DIR first, so you have to
#         manually delete apr.h in PROJECT_SOURCE_DIR/include if
#         you've generated apr.h before using a different build

SET(APR_INCLUDE_DIRECTORIES
  $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/arch/win32>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/arch/unix>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/private>
)

SET(APR_SYSTEM_LIBS
  ws2_32
  rpcrt4
  bcrypt
)

SET(APR_PUBLIC_HEADERS_STATIC
  include/apr_allocator.h
  include/apr_anylock.h
  include/apr_atomic.h
  include/apr_base64.h
  include/apr_buckets.h
  include/apr_buffer.h
  include/apr_crypto.h
  include/apr_cstr.h
  include/apr_date.h
  include/apr_dbd.h
  include/apr_dbm.h
  include/apr_dso.h
  include/apr_encode.h
  include/apr_env.h
  include/apr_errno.h
  include/apr_escape.h
  include/apr_file_info.h
  include/apr_file_io.h
  include/apr_fnmatch.h
  include/apr_general.h
  include/apr_getopt.h
  include/apr_global_mutex.h
  include/apr_hash.h
  include/apr_hooks.h
  include/apr_inherit.h
  include/apr_jose.h
  include/apr_json.h
  include/apr_ldap.h
  include/apr_lib.h
  include/apr_md4.h
  include/apr_md5.h
  include/apr_memcache.h
  include/apr_mmap.h
  include/apr_network_io.h
  include/apr_optional.h
  include/apr_optional_hooks.h
  include/apr_perms_set.h
  include/apr_poll.h
  include/apr_pools.h
  include/apr_portable.h
  include/apr_proc_mutex.h
  include/apr_queue.h
  include/apr_random.h
  include/apr_redis.h
  include/apr_reslist.h
  include/apr_ring.h
  include/apr_rmm.h
  include/apr_sdbm.h
  include/apr_sha1.h
  include/apr_shm.h
  include/apr_signal.h
  include/apr_siphash.h
  include/apr_skiplist.h
  include/apr_strings.h
  include/apr_strmatch.h
  include/apr_tables.h
  include/apr_thread_cond.h
  include/apr_thread_mutex.h
  include/apr_thread_pool.h
  include/apr_thread_proc.h
  include/apr_thread_rwlock.h
  include/apr_time.h
  include/apr_uri.h
  include/apr_user.h
  include/apr_uuid.h
  include/apr_version.h
  include/apr_want.h
  include/apr_xlate.h
  include/apr_xml.h
  include/apu.h
  include/apu_errno.h
  include/apu_version.h
)
SET(APR_PUBLIC_HEADERS_GENERATED
  ${PROJECT_BINARY_DIR}/apr.h
  ${PROJECT_BINARY_DIR}/apu_want.h
)

SET(APR_SOURCES
  atomic/win32/apr_atomic.c
  atomic/win32/apr_atomic64.c
  buckets/apr_brigade.c
  buckets/apr_buckets.c
  buckets/apr_buckets_alloc.c
  buckets/apr_buckets_eos.c
  buckets/apr_buckets_file.c
  buckets/apr_buckets_flush.c
  buckets/apr_buckets_heap.c
  buckets/apr_buckets_mmap.c
  buckets/apr_buckets_pipe.c
  buckets/apr_buckets_pool.c
  buckets/apr_buckets_refcount.c
  buckets/apr_buckets_simple.c
  buckets/apr_buckets_socket.c
  buffer/apr_buffer.c
  crypto/apr_crypto.c
  crypto/apr_crypto_prng.c
  crypto/apr_md4.c
  crypto/apr_md5.c
  crypto/apr_passwd.c
  crypto/apr_sha1.c
  crypto/apr_siphash.c
  crypto/crypt_blowfish.c
  crypto/getuuid.c
  crypto/uuid.c
  dbd/apr_dbd.c
  dbm/apr_dbm.c
  dbm/apr_dbm_sdbm.c
  dbm/sdbm/sdbm.c
  dbm/sdbm/sdbm_hash.c
  dbm/sdbm/sdbm_lock.c
  dbm/sdbm/sdbm_pair.c
  dso/win32/dso.c
  encoding/apr_base64.c
  encoding/apr_encode.c
  encoding/apr_escape.c
  file_io/unix/copy.c
  file_io/unix/fileacc.c
  file_io/unix/filepath_util.c
  file_io/unix/fullrw.c
  file_io/unix/mktemp.c
  file_io/unix/tempdir.c
  file_io/win32/buffer.c
  file_io/win32/dir.c
  file_io/win32/filedup.c
  file_io/win32/filepath.c
  file_io/win32/filestat.c
  file_io/win32/filesys.c
  file_io/win32/flock.c
  file_io/win32/open.c
  file_io/win32/pipe.c
  file_io/win32/readwrite.c
  file_io/win32/seek.c
  jose/apr_jose.c
  jose/apr_jose_decode.c
  jose/apr_jose_encode.c
  json/apr_json.c
  json/apr_json_decode.c
  json/apr_json_encode.c
  hooks/apr_hooks.c
  locks/win32/proc_mutex.c
  locks/win32/thread_cond.c
  locks/win32/thread_mutex.c
  locks/win32/thread_rwlock.c
  memcache/apr_memcache.c
  memory/unix/apr_pools.c
  misc/unix/errorcodes.c
  misc/unix/getopt.c
  misc/unix/otherchild.c
  misc/unix/version.c
  misc/win32/charset.c
  misc/win32/env.c
  misc/win32/internal.c
  misc/win32/misc.c
  misc/win32/rand.c
  misc/win32/start.c
  misc/win32/utf8.c
  mmap/unix/common.c
  mmap/win32/mmap.c
  network_io/unix/inet_ntop.c
  network_io/unix/inet_pton.c
  network_io/unix/multicast.c
  network_io/unix/sockaddr.c
  network_io/unix/socket_util.c
  network_io/win32/sendrecv.c
  network_io/win32/socket_pipe.c
  network_io/win32/sockets.c
  network_io/win32/sockopt.c
  passwd/apr_getpass.c
  poll/unix/poll.c
  poll/unix/pollcb.c
  poll/unix/pollset.c
  poll/unix/select.c
  poll/unix/wakeup.c
  random/unix/apr_random.c
  random/unix/sha2.c
  random/unix/sha2_glue.c
  redis/apr_redis.c
  shmem/win32/shm.c
  strings/apr_cpystrn.c
  strings/apr_cstr.c
  strings/apr_fnmatch.c
  strings/apr_snprintf.c
  strings/apr_strings.c
  strings/apr_strnatcmp.c
  strings/apr_strtok.c
  strmatch/apr_strmatch.c
  tables/apr_hash.c
  tables/apr_skiplist.c
  tables/apr_tables.c
  threadproc/win32/proc.c
  threadproc/win32/signals.c
  threadproc/win32/thread.c
  threadproc/win32/threadpriv.c
  time/win32/time.c
  time/win32/timestr.c
  uri/apr_uri.c
  user/win32/groupinfo.c
  user/win32/userinfo.c
  util-misc/apr_date.c
  util-misc/apr_error.c
  util-misc/apr_queue.c
  util-misc/apr_reslist.c
  util-misc/apr_rmm.c
  util-misc/apr_thread_pool.c
  util-misc/apu_dso.c
  xlate/xlate.c
  xml/apr_xml.c
  xml/apr_xml_expat.c
  xml/apr_xml_libxml2.c
  xml/apr_xml_xmllite.c
)

# Sources and libraries for APR builtin drivers. Used when APR_MODULAR_DSO=OFF.
SET(APR_EXTRA_SOURCES)
SET(APR_EXTRA_LIBRARIES)

SET(APR_TEST_SUITES
  testargs
  testatomic
  testbase64
  testbuckets
  testbuffer
  testcond
  testcrypto
  testdate
  testdbd
  testdbm
  testdir
  testdso
  testdup
  testenv
  testencode
  testescape
  testfile
  testfilecopy
  testfileinfo
  testflock
  testfmt
  testfnmatch
  testglobalmutex
  testhash
  testhooks
  testjson
  testjose
  testipsub
  testldap
  testlfs
  testlfsabi
  testlock
  testmd4
  testmd5
  testmemcache
  testmmap
  testnames
  testoc
  testpass
  testpath
  testpipe
  testpoll
  testpools
  testproc
  testprocmutex
  testqueue
  testrand
  testredis
  testreslist
  testrmm
  testshm
  testsiphash
  testskiplist
  testsleep
  testsock
  testsockets
  testsockopt
  teststr
  teststrmatch
  teststrnatcmp
  testtable
  testtemp
  testthread
  testtime
  testud
  testuri
  testuser
  testuuid
  testvsn
  testxlate
  testxml
)

SET(install_targets)
SET(install_modules)
SET(install_bin_pdb)
SET(dbd_drivers)

IF (MSVC)
  # Ignore Microsoft's interpretation of secure development
  # and the POSIX string handling API
  ADD_COMPILE_DEFINITIONS(_CRT_SECURE_NO_DEPRECATE _CRT_NONSTDC_NO_WARNINGS)
ENDIF()

MACRO(ADD_APR_MODULE name dllname sources libraries)
  IF(APR_MODULAR_DSO)
    ADD_LIBRARY(${name} SHARED ${sources})
    LIST(APPEND install_modules ${name})
    LIST(APPEND install_bin_pdb $<TARGET_PDB_FILE:${name}>)

    TARGET_SOURCES(${name} PRIVATE libapr.rc)
    TARGET_COMPILE_DEFINITIONS(${name}
                               PRIVATE DLL_NAME=${dllname})

    TARGET_LINK_LIBRARIES(${name}
                          PRIVATE ${apr_libname} ${APR_SYSTEM_LIBS})
    TARGET_LINK_LIBRARIES(${name}
                          PRIVATE ${libraries})
  ELSE()
    LIST(APPEND APR_EXTRA_SOURCES ${sources})
    LIST(APPEND APR_EXTRA_LIBRARIES ${libraries})
  ENDIF()
ENDMACRO()

IF(APU_HAVE_CRYPTO)
  IF(NOT OPENSSL_FOUND)
    MESSAGE(FATAL_ERROR "Only OpenSSL-based crypto is currently implemented in the cmake build")
  ENDIF()

  ADD_APR_MODULE(
    apr_crypto_openssl-2 "apr_crypto_openssl"
    "crypto/apr_crypto_openssl.c"
    "OpenSSL::Crypto")
ENDIF()

IF(APU_HAVE_ODBC)
  LIST(APPEND dbd_drivers odbc)

  ADD_APR_MODULE(
    apr_dbd_odbc-2 "apr_dbd_odbc"
    "dbd/apr_dbd_odbc.c"
    "odbc32;odbccp32"
  )
ENDIF()

IF(APU_HAVE_SQLITE3)
  LIST(APPEND dbd_drivers sqlite3)

  ADD_APR_MODULE(
    apr_dbd_sqlite3-2 "apr_dbd_sqlite3-1"
    "dbd/apr_dbd_sqlite3.c"
    "SQLite::SQLite3"
  )
ENDIF()

IF(APU_HAVE_PGSQL)
  LIST(APPEND dbd_drivers pgsql)

  ADD_APR_MODULE(apr_dbd_pgsql-2 "apr_dbd_pgsql"
                 "dbd/apr_dbd_pgsql.c"
                 "PostgreSQL::PostgreSQL")

  ADD_COMPILE_DEFINITIONS(HAVE_LIBPQ_FE_H)
ENDIF()

IF (BUILD_SHARED_LIBS)
  ADD_LIBRARY(${apr_libname} SHARED ${APR_SOURCES} ${APR_EXTRA_SOURCES} ${APR_PUBLIC_HEADERS_GENERATED} libapr.rc)
  LIST(APPEND install_targets ${apr_libname})
  LIST(APPEND install_bin_pdb $<TARGET_PDB_FILE:${apr_libname}>)
  TARGET_LINK_LIBRARIES(${apr_libname}
                        PRIVATE ${XMLLIB_LIBRARIES} ${XLATE_LIBRARIES} ${APR_SYSTEM_LIBS} ${APR_EXTRA_LIBRARIES})
  TARGET_COMPILE_DEFINITIONS(${apr_libname}
                             PRIVATE "APR_DECLARE_EXPORT"
                             INTERFACE "APR_DECLARE_IMPORT")
  TARGET_INCLUDE_DIRECTORIES(${apr_libname}
    PRIVATE ${APR_INCLUDE_DIRECTORIES}
    INTERFACE $<INSTALL_INTERFACE:${APR_INSTALL_INCLUDE_DIR}> ${APR_INCLUDE_DIRECTORIES}
  )
  ADD_DEPENDENCIES(${apr_libname} test_char_header)
ELSE()
  ADD_LIBRARY(${apr_libname} STATIC ${APR_SOURCES} ${APR_EXTRA_SOURCES} ${APR_PUBLIC_HEADERS_GENERATED})
  LIST(APPEND install_targets ${apr_libname})
  # no .pdb file generated for static libraries
  TARGET_LINK_LIBRARIES(${apr_libname}
                        PRIVATE ${XMLLIB_LIBRARIES} ${XLATE_LIBRARIES} ${APR_SYSTEM_LIBS} ${APR_EXTRA_LIBRARIES})
  TARGET_COMPILE_DEFINITIONS(${apr_libname}
                             PRIVATE "APR_DECLARE_STATIC"
                             INTERFACE "APR_DECLARE_STATIC")
  TARGET_INCLUDE_DIRECTORIES(${apr_libname}
    PRIVATE ${APR_INCLUDE_DIRECTORIES}
    INTERFACE $<INSTALL_INTERFACE:${APR_INSTALL_INCLUDE_DIR}> ${APR_INCLUDE_DIRECTORIES}
  )
  ADD_DEPENDENCIES(${apr_libname} test_char_header)
ENDIF()

IF (BUILD_SHARED_LIBS)
  ADD_LIBRARY(libaprapp-2 STATIC misc/win32/apr_app.c misc/win32/internal.c ${APR_PUBLIC_HEADERS_GENERATED})
  LIST(APPEND install_targets libaprapp-2)
  SET_TARGET_PROPERTIES(libaprapp-2 PROPERTIES COMPILE_DEFINITIONS APR_APP)
  TARGET_INCLUDE_DIRECTORIES(libaprapp-2
    PRIVATE ${APR_INCLUDE_DIRECTORIES}
  )
ELSE()
  ADD_LIBRARY(libaprapp-2 STATIC misc/win32/apr_app.c misc/win32/internal.c ${APR_PUBLIC_HEADERS_GENERATED})
  LIST(APPEND install_targets libaprapp-2)
  SET_TARGET_PROPERTIES(libaprapp-2 PROPERTIES COMPILE_DEFINITIONS "APR_DECLARE_STATIC;APR_APP")
  TARGET_INCLUDE_DIRECTORIES(libaprapp-2
    PRIVATE ${APR_INCLUDE_DIRECTORIES}
  )
ENDIF()

IF(APR_BUILD_TESTAPR)
  ENABLE_TESTING()
  # Create a "check" target that displays test program output to the console.
  ADD_CUSTOM_TARGET(check COMMAND ${CMAKE_CTEST_COMMAND} --verbose)

  # copy data files to build directory so that we can run programs from there
  EXECUTE_PROCESS(COMMAND ${CMAKE_COMMAND} -E make_directory 
                  ${PROJECT_BINARY_DIR}/data)
  EXECUTE_PROCESS(COMMAND ${CMAKE_COMMAND} -E copy_if_different 
                  ${PROJECT_SOURCE_DIR}/test/data/billion-laughs.xml
                  ${PROJECT_BINARY_DIR}/data/billion-laughs.xml)
  EXECUTE_PROCESS(COMMAND ${CMAKE_COMMAND} -E copy_if_different 
                  ${PROJECT_SOURCE_DIR}/test/data/file_datafile.txt
                  ${PROJECT_BINARY_DIR}/data/file_datafile.txt)
  EXECUTE_PROCESS(COMMAND ${CMAKE_COMMAND} -E copy_if_different
                  ${PROJECT_SOURCE_DIR}/test/data/mmap_datafile.txt
                  ${PROJECT_BINARY_DIR}/data/mmap_datafile.txt)
  EXECUTE_PROCESS(COMMAND ${CMAKE_COMMAND} -E copy_if_different
                  ${PROJECT_SOURCE_DIR}/test/data/mmap_large_datafile.txt
                  ${PROJECT_BINARY_DIR}/data/mmap_large_datafile.txt)

  EXECUTE_PROCESS(COMMAND ${CMAKE_COMMAND} -E copy_if_different
                  ${PROJECT_SOURCE_DIR}/test/echoargs.bat
                  ${PROJECT_BINARY_DIR}/echoargs.bat)

  ADD_EXECUTABLE(testapp test/testapp.c)
  TARGET_LINK_LIBRARIES(testapp ${apr_libname} libaprapp-2)
  SET_TARGET_PROPERTIES(testapp PROPERTIES LINK_FLAGS /entry:wmainCRTStartup)
  ADD_TEST(NAME testapp COMMAND testapp)

  SET (APR_TEST_SOURCES
    test/abts.c
    test/testlfsabi32.c
    test/testlfsabi64.c
    test/testlfsabi_include.c
    test/testutil.c
    )

  FOREACH(testsuite ${APR_TEST_SUITES})
    LIST(APPEND APR_TEST_SOURCES test/${testsuite}.c)
  ENDFOREACH()

  ADD_EXECUTABLE(testall ${APR_TEST_SOURCES})
  TARGET_LINK_LIBRARIES(testall ${apr_libname} ws2_32.lib)
  SET_TARGET_PROPERTIES(testall PROPERTIES COMPILE_DEFINITIONS "BINPATH=$<TARGET_FILE_DIR:testall>")

  FOREACH(test ${APR_TEST_SUITES})
    ADD_TEST(NAME ${test} COMMAND testall -v ${test})
  ENDFOREACH()

  ADD_LIBRARY(mod_test MODULE test/mod_test.c)
  TARGET_LINK_LIBRARIES(mod_test ${apr_libname})
  SET_PROPERTY(TARGET mod_test APPEND PROPERTY LINK_FLAGS /export:print_hello)
  # nasty work-around for difficulties adding more than one additional flag
  # (they get joined in a bad way behind the scenes)
  GET_PROPERTY(link_flags TARGET mod_test PROPERTY LINK_FLAGS)
  SET(link_flags "${link_flags} /export:count_reps")
  SET_TARGET_PROPERTIES(mod_test PROPERTIES LINK_FLAGS ${link_flags})

  # Build all the single-source executable files with no special build
  # requirements.
  SET(single_source_programs
    test/dbd.c
    test/echoargs.c
    test/echod.c
    test/sendfile.c
    test/sockperf.c
    test/testlockperf.c
    test/testmutexscope.c
    test/globalmutexchild.c
    test/occhild.c
    test/proc_child.c
    test/readchild.c
    test/sockchild.c
    test/memcachedmock.c
    test/testshmproducer.c
    test/testshmconsumer.c
    test/tryread.c
    test/internal/testutf.c
  )

  FOREACH(sourcefile ${single_source_programs})
    STRING(REGEX REPLACE ".*/([^\\]+)\\.c" "\\1" proggie ${sourcefile})
    ADD_EXECUTABLE(${proggie} ${sourcefile})
    TARGET_LINK_LIBRARIES(${proggie} ${apr_libname})
    SET_TARGET_PROPERTIES(${proggie} PROPERTIES COMPILE_DEFINITIONS "BINPATH=$<TARGET_FILE_DIR:${proggie}>")

    ADD_DEPENDENCIES(testall ${proggie})
  ENDFOREACH()

  # Add tests for programs that run by themselves with no arguments.
  SET(simple_tests
    testmutexscope
    testutf
  )

  FOREACH(simple ${simple_tests})
    ADD_TEST(NAME ${simple} COMMAND ${simple})
  ENDFOREACH()

  # testlockperf takes forever on Windows with default counter limit
  ADD_TEST(NAME testlockperf COMMAND testlockperf -c 50000)

  # dbd and sendfile are run multiple times with different parameters.
  FOREACH(somedbd ${dbd_drivers})
    ADD_TEST(NAME dbd-${somedbd} COMMAND dbd ${somedbd})
  ENDFOREACH()

  FOREACH(sendfile_mode blocking nonblocking timeout)
    ADD_TEST(NAME sendfile-${sendfile_mode} COMMAND sendfile client ${sendfile_mode} startserver)
  ENDFOREACH()

  # No test is added for echod+sockperf.  Those will have to be run manually.

ENDIF (APR_BUILD_TESTAPR)

# Installation
INCLUDE(CMakePackageConfigHelpers)

string(TOLOWER "${PROJECT_NAME}" project_name_lower)

write_basic_package_version_file("${CMAKE_CURRENT_BINARY_DIR}/${project_name_lower}-config-version.cmake"
  VERSION ${APR_MAJOR_VERSION}.${APR_MINOR_VERSION}.${APR_PATCH_VERSION}
  COMPATIBILITY SameMajorVersion
)

INSTALL(TARGETS ${install_targets} EXPORT apr-config
        RUNTIME DESTINATION ${APR_INSTALL_BIN_DIR}
        LIBRARY DESTINATION ${APR_INSTALL_LIB_DIR}
        ARCHIVE DESTINATION ${APR_INSTALL_LIB_DIR}
)

INSTALL(EXPORT apr-config
        NAMESPACE apr::
        DESTINATION "lib/cmake/${project_name_lower}"
        FILE "${project_name_lower}-config.cmake"
)

INSTALL(FILES "${CMAKE_CURRENT_BINARY_DIR}/${project_name_lower}-config-version.cmake"
        DESTINATION "lib/cmake/${project_name_lower}"
)

# For DSO modules, install only binaries
INSTALL(TARGETS ${install_modules}
        COMPONENT RUNTIME
        DESTINATION ${APR_INSTALL_BIN_DIR})

IF(INSTALL_PDB)
  INSTALL(FILES ${install_bin_pdb}
          DESTINATION ${APR_INSTALL_BIN_DIR}
          CONFIGURATIONS RelWithDebInfo Debug)
ENDIF()

INSTALL(FILES ${APR_PUBLIC_HEADERS_STATIC} ${APR_PUBLIC_HEADERS_GENERATED} DESTINATION ${APR_INSTALL_INCLUDE_DIR})
IF(APR_INSTALL_PRIVATE_H)
  # Kludges for unexpected dependencies of httpd 2.x, not installed by default
  SET(APR_PRIVATE_H_FOR_HTTPD
    include/arch/win32/apr_arch_file_io.h
    include/arch/win32/apr_arch_misc.h
    include/arch/win32/apr_arch_utf8.h
    include/arch/win32/apr_private.h
    )
  INSTALL(FILES ${APR_PRIVATE_H_FOR_HTTPD} DESTINATION ${APR_INSTALL_INCLUDE_DIR}/arch/win32)
ENDIF()

STRING(TOUPPER "${CMAKE_BUILD_TYPE}" buildtype)
MESSAGE(STATUS "")
MESSAGE(STATUS "")
MESSAGE(STATUS "APR configuration summary:")
MESSAGE(STATUS "")

MESSAGE(STATUS "  Build type ...................... : ${CMAKE_BUILD_TYPE}")
MESSAGE(STATUS "  Install .pdb (if available)...... : ${INSTALL_PDB}")
MESSAGE(STATUS "  Install prefix .................. : ${CMAKE_INSTALL_PREFIX}")
MESSAGE(STATUS "    Directory for binary files .... : PREFIX/${APR_INSTALL_BIN_DIR}")
MESSAGE(STATUS "    Directory for library files ... : PREFIX/${APR_INSTALL_LIB_DIR}")
MESSAGE(STATUS "    Directory for include files ... : PREFIX/${APR_INSTALL_INCLUDE_DIR}")
MESSAGE(STATUS "  C compiler ...................... : ${CMAKE_C_COMPILER}")
MESSAGE(STATUS "  IPv6 ............................ : ${APR_HAVE_IPV6}")
MESSAGE(STATUS "  DBD ODBC driver ................. : ${APU_HAVE_ODBC}")
MESSAGE(STATUS "  DBD SQLite3 driver .............. : ${APU_HAVE_SQLITE3}")
MESSAGE(STATUS "  DBD PostgreSQL .................. : ${APU_HAVE_PGSQL}")
MESSAGE(STATUS "  XML backend ..................... : ${APR_XML_BACKEND}")
MESSAGE(STATUS "  Have Crypto ..................... : ${APU_HAVE_CRYPTO}")
MESSAGE(STATUS "  Have Iconv ...................... : ${APU_HAVE_ICONV}")
MESSAGE(STATUS "  Library files for XML ........... : ${XMLLIB_LIBRARIES}")
MESSAGE(STATUS "  DSO build of modular components.. : ${APR_MODULAR_DSO}")
MESSAGE(STATUS "  Turn on pools debugging ..........: ${APR_POOL_DEBUG}")
MESSAGE(STATUS "  Build shared libraries .......... : ${BUILD_SHARED_LIBS}")
MESSAGE(STATUS "  Build test suite ................ : ${APR_BUILD_TESTAPR}")
MESSAGE(STATUS "  Install private .h for httpd .... : ${APR_INSTALL_PRIVATE_H}")
