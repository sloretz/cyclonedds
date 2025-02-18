#
# Copyright(c) 2021 ADLINK Technology Limited and others
#
# This program and the accompanying materials are made available under the
# terms of the Eclipse Public License v. 2.0 which is available at
# http://www.eclipse.org/legal/epl-2.0, or the Eclipse Distribution License
# v. 1.0 which is available at
# http://www.eclipse.org/org/documents/edl-v10.php.
#
# SPDX-License-Identifier: EPL-2.0 OR BSD-3-Clause
#
include(GenerateExportHeader)

find_package(BISON 3.0.4 REQUIRED)
find_package(Threads QUIET REQUIRED)

set(binary_dir "${CMAKE_CURRENT_BINARY_DIR}")
set(source_dir "${CMAKE_CURRENT_SOURCE_DIR}")

bison_target(parser src/parser.y "${binary_dir}/parser.c")

configure_file(include/idl/version.h.in include/idl/version.h @ONLY)

set(headers
  include/idl/expression.h
  include/idl/file.h
  include/idl/print.h
  include/idl/processor.h
  include/idl/retcode.h
  include/idl/scope.h
  include/idl/stream.h
  include/idl/string.h
  include/idl/symbol.h
  include/idl/tree.h
  include/idl/visit.h
  ${binary_dir}/include/idl/version.h)

set(sources
  src/symbol.c
  src/directive.c
  src/expression.c
  src/file.c
  src/processor.c
  src/scanner.c
  src/string.c
  src/annotation.c
  src/scope.c
  src/string.c
  src/tree.c
  src/visit.c
  src/print.c
  src/keylist.c
  src/hashid.c
  ${BISON_parser_OUTPUT_SOURCE}
  src/annotation.h
  src/directive.h
  src/expression.h
  src/file.h
  src/hashid.h
  src/scanner.h
  src/scope.h
  src/symbol.h
  src/tree.h
  src/keylist.h)

# import generic utility sources from ddsrt to avoid duplication
# NOTE: an additional INTERFACE library defined in ddsrt would be ideal, but
#       the GENERATED property is not propagated before 3.20. using an OBJECT
#       library and propagating the object files through an INTERFACE library
#       can be done, but a proper export header can only be generated from the
#       final library.
#       see: https://gitlab.kitware.com/cmake/cmake/-/issues/18399
set(script "${PROJECT_SOURCE_DIR}/src/ddsrt/rename.cmake")
set(templates
  include/dds/ddsrt/misc.h
  include/dds/ddsrt/attributes.h
  include/dds/ddsrt/endian.h
  include/dds/ddsrt/md5.h
  src/md5.c)
# FIXME: move general functionality to ddsrt

foreach(source ${templates})
  set(input "${PROJECT_SOURCE_DIR}/src/ddsrt/${source}")
  # map include path. i.e. map include/dds/ddsrt to include/idl
  string(REGEX REPLACE "(/dds(rt)?)+" "/idl" source "${source}")
  set(output "${binary_dir}/${source}")
  # process source file. i.e. replace dds and ddsrt with idl
  add_custom_command(
    OUTPUT "${output}"
    COMMAND "${CMAKE_COMMAND}" -DNAMESPACE=idl
                               -DINPUT_FILE="${input}"
                               -DOUTPUT_DIRECTORY="${binary_dir}"
                               -DOUTPUT_FILE="${output}"
                               -P "${script}"
    DEPENDS "${input}" "${script}")
  if(source MATCHES "^include/.*\\.h")
    list(APPEND headers ${output})
  else()
    list(APPEND sources ${output})
  endif()
endforeach()

add_library(idl SHARED ${headers} ${sources})

set_target_properties(
  idl PROPERTIES
  OUTPUT_NAME "cycloneddsidl"
  VERSION ${PROJECT_VERSION}
  SOVERSION ${PROJECT_VERSION_MAJOR})

generate_export_header(idl EXPORT_FILE_NAME include/idl/export.h)


target_include_directories(
  idl
  PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>"
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>"
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>"
  INTERFACE
    "$<INSTALL_INTERFACE:include>")

target_link_libraries(idl PRIVATE Threads::Threads)

install(
  DIRECTORY
    "${source_dir}/include/idl"
    "${binary_dir}/include/idl"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  COMPONENT dev
  FILES_MATCHING PATTERN "*.h")

install(
  TARGETS idl
  EXPORT "${CMAKE_PROJECT_NAME}"
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT lib
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT lib
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT lib)

if (MSVC)
  install(FILES $<TARGET_PDB_FILE:idl>
    DESTINATION "${CMAKE_INSTALL_BINDIR}"
    COMPONENT dev
    OPTIONAL)
endif()

if(BUILD_TESTING)
  add_subdirectory(tests)
endif()
