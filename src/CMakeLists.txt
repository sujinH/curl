#***************************************************************************
#                                  _   _ ____  _
#  Project                     ___| | | |  _ \| |
#                             / __| | | | |_) | |
#                            | (__| |_| |  _ <| |___
#                             \___|\___/|_| \_\_____|
#
# Copyright (C) Daniel Stenberg, <daniel@haxx.se>, et al.
#
# This software is licensed as described in the file COPYING, which
# you should have received as part of this distribution. The terms
# are also available at https://curl.se/docs/copyright.html.
#
# You may opt to use, copy, modify, merge, publish, distribute and/or sell
# copies of the Software, and permit persons to whom the Software is
# furnished to do so, under the terms of the COPYING file.
#
# This software is distributed on an "AS IS" basis, WITHOUT WARRANTY OF ANY
# KIND, either express or implied.
#
# SPDX-License-Identifier: curl
#
###########################################################################
set(EXE_NAME curl)

if(USE_MANUAL)
  # Use the C locale to ensure that only ASCII characters appear in the
  # embedded text. NROFF and MANOPT are set in the parent CMakeLists.txt
  add_custom_command(
    OUTPUT tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#include \"tool_setup.h\"" > tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#ifndef HAVE_LIBZ" >> tool_hugehelp.c
    COMMAND env LC_ALL=C "${NROFF}" ${NROFF_MANOPT}
            "${CURL_BINARY_DIR}/docs/curl.1" |
            "${PERL_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl" >> tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#else" >> tool_hugehelp.c
    COMMAND env LC_ALL=C "${NROFF}" ${NROFF_MANOPT}
            "${CURL_BINARY_DIR}/docs/curl.1" |
            "${PERL_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl" -c >> tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#endif /* HAVE_LIBZ */" >> tool_hugehelp.c
    DEPENDS
      generate-curl.1
      "${CURL_BINARY_DIR}/docs/curl.1"
      "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl"
      "${CMAKE_CURRENT_SOURCE_DIR}/tool_hugehelp.h"
    VERBATIM)
else()
  add_custom_command(
    OUTPUT tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#include \"tool_hugehelp.h\"" > tool_hugehelp.c
    DEPENDS
      "${CMAKE_CURRENT_SOURCE_DIR}/tool_hugehelp.h"
    VERBATIM)

endif()

transform_makefile_inc("Makefile.inc" "${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake")
include(${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake)

if(WIN32)
  list(APPEND CURL_CFILES curl.rc)
endif()

# CURL_CFILES, CURLX_CFILES, CURL_HFILES, CURLTOOL_LIBCURL_CFILES
# come from Makefile.inc
if(BUILD_STATIC_CURL)
  set(CURLX_CFILES ${CURLTOOL_LIBCURL_CFILES})
endif()

# Compile this source separately to avoid having `stderr` overridden in this
# file. Necessary for successfully initializing stderr in unity builds.
# (Not necessary for the libcurltool target.)
set_source_files_properties(tool_stderr.c PROPERTIES SKIP_UNITY_BUILD_INCLUSION ON)

add_executable(
  ${EXE_NAME}
  ${CURL_CFILES} ${CURLX_CFILES} ${CURL_HFILES}
  )

add_executable(
  ${PROJECT_NAME}::${EXE_NAME}
  ALIAS ${EXE_NAME}
  )

add_library(
  curltool # special libcurltool library just for unittests
  STATIC
  EXCLUDE_FROM_ALL
  ${CURL_CFILES} ${CURLTOOL_LIBCURL_CFILES} ${CURL_HFILES}
)
target_compile_definitions(curltool PUBLIC UNITTESTS CURL_STATICLIB)
target_link_libraries(curltool PRIVATE ${CURL_LIBS})

if(CURL_HAS_LTO)
  set_target_properties(${EXE_NAME} PROPERTIES
    INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE
    INTERPROCEDURAL_OPTIMIZATION_RELWITHDEBINFO TRUE)
endif()

if(ENABLE_UNICODE AND MINGW)
  target_link_libraries(${EXE_NAME} -municode)
endif()

source_group("curlX source files" FILES ${CURLX_CFILES})
source_group("curl source files" FILES ${CURL_CFILES})
source_group("curl header files" FILES ${CURL_HFILES})

include_directories(
  ${CURL_SOURCE_DIR}/lib        # To be able to reach "curl_setup_once.h"
  ${CURL_BINARY_DIR}/lib        # To be able to reach "curl_config.h"
  ${CURL_BINARY_DIR}/include    # To be able to reach "curl/curl.h"
  # This is needed as tool_hugehelp.c is generated in the binary dir
  ${CURL_SOURCE_DIR}/src        # To be able to reach "tool_hugehelp.h"
  )

#Build curl executable
target_link_libraries(${EXE_NAME} ${LIB_SELECTED_FOR_EXE} ${CURL_LIBS})

################################################################################

#SET_TARGET_PROPERTIES(${EXE_NAME} ARCHIVE_OUTPUT_DIRECTORY "blah blah blah")
#SET_TARGET_PROPERTIES(${EXE_NAME} RUNTIME_OUTPUT_DIRECTORY "blah blah blah")
#SET_TARGET_PROPERTIES(${EXE_NAME} LIBRARY_OUTPUT_DIRECTORY "blah blah blah")

#INCLUDE(ModuleInstall OPTIONAL)

install(TARGETS ${EXE_NAME} EXPORT ${TARGETS_EXPORT_NAME} DESTINATION ${CMAKE_INSTALL_BINDIR})
export(TARGETS ${EXE_NAME}
       FILE ${PROJECT_BINARY_DIR}/curl-target.cmake
       NAMESPACE ${PROJECT_NAME}::
)
