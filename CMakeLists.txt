#   Copyright (C) 2018 by Till Hofmann

#   This program is free software; you can redistribute it and/or modify it
#   under the terms of the GNU General Public License version 2 as published
#   by the Free Software Foundation.

cmake_minimum_required (VERSION 3.1.0)

# projectname is the same as the main-executable
project(memleax)

find_package(PkgConfig)

if (NOT PKG_CONFIG_FOUND)
  message(FATAL_ERROR "Could not find PkgConfig")
endif()

add_executable(
  memleax
  addr_maps.c
  breakpoint.c
  callstack.c
  debug_file.c
  debug_line.c
  memblock.c
  memleax.c
  proc_info.c
  ptr_backtrace.c
  symtab.c
)

target_compile_options(memleax PRIVATE -g -Wall)
if(${CMAKE_SYSTEM_NAME} STREQUAL Linux)
  target_compile_options(memleax PRIVATE -DMLX_LINUX)
elseif(${CMAKE_SYSTEM_NAME} STREQUAL FreeBSD)
  target_compile_options(memleax PRIVATE -DMLX_FREEBSD)
  target_link_libraries(memleax PRIVATE procstat util z)
else()
  message(FATAL_ERROR "Unsupported OS: ${CMAKE_SYSTEM_NAME}. Only GNU/Linux and FreeBSD are supported.")
endif()

set(TARGET_ARCH "autodetect" CACHE STRING "Target Architecture")

if (TARGET_ARCH STREQUAL "autodetect")
	execute_process(COMMAND uname -m
	  RESULT_VARIABLE UNAME_RES OUTPUT_VARIABLE
	  MACHINE_TYPE OUTPUT_STRIP_TRAILING_WHITESPACE)
	if (UNAME_RES)
	  message(FATAL_ERROR "Error running uname, result: ${UNAME_RES}")
	endif()
else()
	set(MACHINE_TYPE ${TARGET_ARCH})
endif()

message("Machine type is: ${MACHINE_TYPE}")


if (MACHINE_TYPE STREQUAL "x86" OR MACHINE_TYPE STREQUAL "i386" OR MACHINE_TYPE STREQUAL "i686")
  target_compile_options(memleax PRIVATE -DMLX_X86)
  target_link_libraries(memleax PRIVATE unwind-x86)
elseif (MACHINE_TYPE STREQUAL "x86_64" OR MACHINE_TYPE STREQUAL "amd64")
  target_compile_options(memleax PRIVATE -DMLX_X86_64)
  target_link_libraries(memleax PRIVATE unwind-x86_64)
elseif (MACHINE_TYPE MATCHES "armv7.*" OR MACHINE_TYPE STREQUAL "armv7")
  target_compile_options(memleax PRIVATE -DMLX_ARMv7)
  target_link_libraries(memleax PRIVATE unwind-arm)
elseif (MACHINE_TYPE STREQUAL "aarch64")
  target_compile_options(memleax PRIVATE -DMLX_AARCH64)
  target_link_libraries(memleax PRIVATE unwind-aarch64)
else ()
  message(FATAL_ERROR "Unsupported machine type: ${MACHINE_TYPE}")
endif()

pkg_search_module(LIBUNWIND REQUIRED libunwind-ptrace)
pkg_search_module(LIBELF REQUIRED libelf)
target_compile_options(memleax PRIVATE ${LIBUNWIND_CFLAGS})
target_compile_options(memleax PRIVATE ${LIBELF_CFLAGS})
target_link_libraries(memleax PRIVATE ${LIBUNWIND_LIBRARIES} ${LIBELF_LIBRARIES})
pkg_search_module(LIBDW libdw)
if (LIBDW_FOUND)
  target_compile_options(memleax PRIVATE ${LIBDW_CFLAGS})
  target_link_libraries(memleax PRIVATE ${LIBDW_LIBRARIES})
else()
  pkg_search_module(LIBDWARF libdwarf)
  if (LIBDWARF_FOUND)
    target_compile_options(memleax PRIVATE ${LIBDWARF_CFLAGS})
    target_link_libraries(memleax PRIVATE ${LIBDWARF_LIBRARIES})
  else()
    message(WARNING "Could not find libdw or libdwarf. File name and line number will not be shown in backtrace.")
  endif()
endif()

install(TARGETS memleax RUNTIME DESTINATION bin)
install(FILES memleax.1 DESTINATION share/man/man1/)
