cmake_minimum_required(VERSION 3.25)

# Enable all policies to latest behavior
cmake_policy(VERSION ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION})

project(Sparse
    VERSION 0.6.4
    LANGUAGES C
)

option(BUILD_COMPILE "Build the 'compile' program" ON)
option(BUILD_CTAGS "Build the sparse-based 'ctags' program" ON)
option(BUILD_EXAMPLE "Build the example program" ON)
option(BUILD_GRAPH "Build the 'graph' program" ON)
option(BUILD_OBFUSCATE "Build the 'obfuscate' program" ON)
option(BUILD_CLI "Build the sparse command line tool" ON)
#option(BUILD_CGCC "Build sparse's GCC wrapper, 'cgcc'" ON)
# Optional CLIs
option(BUILD_SEMIND "Build the 'semind' program; needs SQLite" OFF)
option(BUILD_SPARSE_LLVM "Build 'sparse-llvm'" OFF)
option(BUILD_C2XML "Enable building the c2xml program" OFF)

# Global settings
set(CMAKE_C_STANDARD 99)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# This could be cleaner:
#  - Either using a .in file,
#  - not using this at all and just using a -Definition
set(VERSION_H_PATH "${CMAKE_BINARY_DIR}/version.h")
execute_process(
    COMMAND git describe --dirty
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE SPARSE_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
file(WRITE "${VERSION_H_PATH}" "#define SPARSE_VERSION \"${SPARSE_VERSION}\"\n")
add_custom_target(version_h ALL DEPENDS "${VERSION_H_PATH}")

# ---- libsparse ----
add_library(libsparse STATIC)
add_dependencies(libsparse version_h)
target_sources(libsparse
    PRIVATE
        allocate.c
        builtin.c
        char.c
        # compat-OS.c
        cse.c
        dissect.c
        dominate.c
        evaluate.c
        expand.c
        expression.c
        flow.c
        flowgraph.c
        inline.c
        ir.c
        lib.c
        linearize.c
        liveness.c
        memops.c
        opcode.c
        optimize.c
        options.c
        parse.c
        predefine.c
        pre-process.c
        ptrlist.c
        ptrmap.c
        scope.c
        show-parse.c
        simplify.c
        sort.c
        ssa.c
        stats.c
        storage.c
        symbol.c
        target.c
        target-alpha.c
        target-arm.c
        target-arm64.c
        target-bfin.c
        target-default.c
        target-h8300.c
        target-m68k.c
        target-microblaze.c
        target-mips.c
        target-nds32.c
        target-nios2.c
        target-openrisc.c
        target-ppc.c
        target-riscv.c
        target-s390.c
        target-sh.c
        target-sparc.c
        target-x86.c
        target-xtensa.c
        tokenize.c
        unssa.c
        utils.c
        version.c
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS .
        FILES
            ${VERSION_H_PATH}
            allocate.h
            bitmap.h
            bits.h
            builtin.h
            char.h
            compat.h
            cse.h
            dissect.h
            dominate.h
            evaluate.h
            expand.h
            expression.h
            flowgraph.h
            flow.h
            gcc-attr-list.h
            ident-list.h
            ir.h
            lib.h
            linearize.h
            liveness.h
            machine.h
            opcode.h
            optimize.h
            options.h
            parse.h
            ptrlist.h
            ptrmap.h
            scope.h
            simplify.h
            ssa.h
            storage.h
            symbol.h
            target.h
            token.h
            utils.h
)

# Add the platform-dependent compat's
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_sources(libsparse PRIVATE compat-linux.c)
elseif(CMAKE_SYSTEM_NAME MATCHES ".*BSD$" OR CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    target_sources(libsparse PRIVATE compat-bsd.c)
elseif(CYGWIN)
    target_sources(libsparse PRIVATE compat-cygwin.c)
elseif(MINGW)
    target_sources(libsparse PRIVATE compat-mingw.c)
elseif(WIN32) # TODO
    target_sources(libsparse PRIVATE compat-win32.c)
elseif(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
    target_sources(libsparse PRIVATE compat-solaris.c)
else()
    message(FATAL_ERROR "No compat-*.c implementation available for ${CMAKE_SYSTEM_NAME}")
endif()

# Sparse wants `GCC_ROOT`; which is effectively the sysroot.
# So, we feed it something that... works.
if(CMAKE_C_COMPILER_ID STREQUAL "GNU" OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
    # Makefile's method
    execute_process(
        COMMAND ${CMAKE_C_COMPILER} --print-file-name=
        OUTPUT_VARIABLE COMPILER_SYSROOTISH
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
elseif(CMAKE_C_COMPILER_ID STREQUAL "MSVC")
    # We lie on Windows. Windows is a lie, anyway.
    set(COMPILER_SYSROOTISH "${VCINSTALLDIR}/")
else()
    message(WARNING "Unknown compiler, cannot determine sysroot path")
endif()
# Private, so peers do not pick it up.
target_compile_definitions(libsparse PRIVATE GCC_BASE="${COMPILER_SYSROOTISH}")

# Sparse wants to know the target triplet; again, we kinda fake it.
if(CMAKE_C_COMPILER_ID STREQUAL "GNU" OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
    execute_process(
        COMMAND ${CMAKE_C_COMPILER} -print-multiarch
        OUTPUT_VARIABLE MULTIARCH
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
else()
    # Construct a target triplet manually
    set(MULTIARCH "${CMAKE_SYSTEM_PROCESSOR}-${CMAKE_SYSTEM_NAME}-probablynotgnu")
endif()
target_compile_definitions(libsparse PRIVATE MULTIARCH_TRIPLET="${MULTIARCH}")

# Take care of MSVC warnings
if(MSVC)
    target_compile_options(libsparse PRIVATE /wd4311 /w4312)
endif()


target_include_directories(libsparse
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include/sparse>
)

# Provide an export name for downstream
add_library(Sparse::libsparse ALIAS libsparse)

# CLI programs

## Defaults
if(BUILD_COMPILE)
    add_executable(compile compile.c compile-i386.c)
    target_link_libraries(compile PRIVATE libsparse)
endif()

if(BUILD_CTAGS)
    add_executable(sparse-ctags ctags.c)
    target_link_libraries(sparse-ctags PRIVATE libsparse)
endif()

if(BUILD_EXAMPLE)
    add_executable(example example.c)
    target_link_libraries(example PRIVATE libsparse)
endif()

if(BUILD_GRAPH)
    add_executable(graph graph.c)
    target_link_libraries(graph PRIVATE libsparse)
endif()

if(BUILD_OBFUSCATE)
    add_executable(obfuscate obfuscate.c)
    target_link_libraries(obfuscate PRIVATE libsparse)
endif()

if(BUILD_CLI)
    add_executable(sparse sparse.c)
    target_link_libraries(sparse PRIVATE libsparse)
endif()

#if(BUILD_CGCC)
#    add_executable(cgcc cgcc.c)
#    target_link_libraries(cgcc PRIVATE libsparse)
#endif()

# SKIP: What is "boolector"? O.o
#if(BUILD_SCHECK)
#    add_executable(scheck scheck.c)
#    target_link_libraries(scheck PRIVATE libsparse)
#endif()

# I admit using ChatGPT here. These checks are slightly annoying and I...just want to be done.
if(BUILD_SEMIND)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(SQLITE3 sqlite3)
    if(SQLITE3_FOUND)
        string(REPLACE "." ";" SQLITE_VERSION_LIST ${SQLITE3_VERSION})
        list(GET SQLITE_VERSION_LIST 0 SQLITE_MAJOR)
        list(GET SQLITE_VERSION_LIST 1 SQLITE_MINOR)
        list(GET SQLITE_VERSION_LIST 2 SQLITE_PATCH)
        math(EXPR SQLITE_VNUMBER "${SQLITE_MAJOR}*10000 + ${SQLITE_MINOR}*100 + ${SQLITE_PATCH}")
        if(SQLITE_VNUMBER GREATER_EQUAL 32400)
            add_executable(semind semind.c)
            target_link_libraries(semind PRIVATE libsparse ${SQLITE3_LIBRARIES})
            target_include_directories(semind PRIVATE ${SQLITE3_INCLUDE_DIRS})
            target_compile_options(semind PRIVATE ${SQLITE3_CFLAGS_OTHER} -std=gnu99)
        else()
            message(STATUS "SQLite3 version is too old (${SQLITE3_VERSION}), skipping semind")
        endif()
    else()
        message(STATUS "SQLite3 not found, skipping semind")
    endif()
endif()


if(BUILD_SPARSE_LLVM)
    find_program(LLVM_CONFIG llvm-config)
    if(LLVM_CONFIG)
        execute_process(COMMAND ${LLVM_CONFIG} --version
                        OUTPUT_VARIABLE LLVM_VERSION
                        OUTPUT_STRIP_TRAILING_WHITESPACE
                        ERROR_QUIET)
        if(LLVM_VERSION)
            string(REGEX MATCH "^[0-9]+" LLVM_VERSION_MAJOR "${LLVM_VERSION}")
            if(LLVM_VERSION_MAJOR GREATER_EQUAL 3)
                if(NOT DEFINED MULTIARCH_TRIPLET)
                    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
                        set(HOST_ARCH x86_64)
                    else()
                        set(HOST_ARCH i386)
                    endif()
                else()
                    set(HOST_ARCH ${MULTIARCH_TRIPLET})
                endif()
                # Limit ach
                set(ALLOWED_ARCHS i386 i486 i586 i686 x86_64 amd64 aarch64 arm64)
                list(FIND ALLOWED_ARCHS ${HOST_ARCH} ARCH_INDEX)
                # This could be optimized by directly grabbing them
                # I am sure LLVM provides CMake bindings - but I will check them later.
                if(ARCH_INDEX GREATER -1)
                    execute_process(COMMAND ${LLVM_CONFIG} --cppflags
                                    OUTPUT_VARIABLE SPARSE_LLVM_CFLAGS
                                    OUTPUT_STRIP_TRAILING_WHITESPACE)
                    execute_process(COMMAND ${LLVM_CONFIG} --ldflags
                                    OUTPUT_VARIABLE SPARSE_LLVM_LDFLAGS
                                    OUTPUT_STRIP_TRAILING_WHITESPACE)
                    execute_process(COMMAND ${LLVM_CONFIG} --libs
                                    OUTPUT_VARIABLE SPARSE_LLVM_LIBS
                                    OUTPUT_STRIP_TRAILING_WHITESPACE)
                    execute_process(COMMAND ${LLVM_CONFIG} --system-libs
                                    OUTPUT_VARIABLE SPARSE_LLVM_SYSTEM_LIBS
                                    OUTPUT_STRIP_TRAILING_WHITESPACE)
                    set(SPARSE_LLVM_LIBS "${SPARSE_LLVM_LIBS} ${SPARSE_LLVM_SYSTEM_LIBS}")

                    # Special case: LLVM 14
                    if(LLVM_VERSION_MAJOR STREQUAL "14")
                        set(SPARSE_LLVM_CFLAGS "${SPARSE_LLVM_CFLAGS} -Wno-deprecated-declarations")
                    endif()

                    # Add executable
                    add_executable(sparse-llvm sparse-llvm.c)
                    target_link_libraries(sparse-llvm PRIVATE libsparse)
                    target_compile_options(sparse-llvm PRIVATE ${SPARSE_LLVM_CFLAGS})
                    target_link_options(sparse-llvm PRIVATE ${SPARSE_LLVM_LDFLAGS})
                else()
                    message(WARNING "sparse-llvm disabled on architecture ${HOST_ARCH}")
                endif()
            else()
                message(WARNING "LLVM 3.0 or later required. Your system has version ${LLVM_VERSION}")
            endif()
        endif()
    else()
        message(WARNING "Your system does not have llvm-config, disabling sparse-llvm")
    endif()
endif()

## Test programs
# List of test names
set(SPARSE_TESTS dissect lexing linearize parsing show-type unssa)
foreach(test_name IN LISTS SPARSE_TESTS)
    add_executable(test-${test_name} test-${test_name}.c)
    target_link_libraries(test-${test_name} PRIVATE libsparse)
    add_test(NAME ${test_name} COMMAND test-${test_name})
endforeach()

# test-inspect is a GTK app
# Ignore on Windows; might be doable on msys though.
if(NOT WIN32)
    set(GTK_VERSIONS 3.0 2.0)
    set(GTK_FOUND FALSE)
    foreach(GTK_VERSION IN LISTS GTK_VERSIONS)
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(GTK gtk+-${GTK_VERSION})
        if(GTK_FOUND)
            message(STATUS "Found GTK version ${GTK_VERSION}")
            break()
        endif()
    endforeach()
    if(GTK_FOUND)
        add_executable(test-inspect ast-model.c ast-view.c ast-inspect.c test-inspect.c)
        target_link_libraries(test-inspect PRIVATE libsparse ${GTK_LIBRARIES})
        target_include_directories(test-inspect PRIVATE ${GTK_INCLUDE_DIRS})
        target_compile_options(test-inspect PRIVATE ${GTK_CFLAGS_OTHER})
    else()
        message(WARNING "Your system does not have GTK2/GTK3, disabling test-inspect")
    endif()
endif()


## Extra CLIs:
if(BUILD_C2XML)
    find_package(LibXml2 REQUIRED)
    add_executable(c2xml c2xml.c)
    target_link_libraries(c2xml PRIVATE libsparse LibXml2::LibXml2)
endif()


# ---- Install rules ----
include(GNUInstallDirs)

install(
    TARGETS libsparse
    EXPORT SparseTargets
    FILE_SET HEADERS
)

#install(
#    TARGETS sparse c2xml
#    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
#)

install(
    EXPORT SparseTargets
    NAMESPACE Sparse::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Sparse
)