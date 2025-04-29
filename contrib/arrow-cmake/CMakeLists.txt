# We dont use arrow's cmakefiles because they uses too many depends and download some libs in compile time
# But you can update auto-generated parquet files manually:
# cd {BUILD_DIR}/contrib/arrow/cpp/src/parquet && mkdir -p build && cd build
# cmake .. -DARROW_COMPUTE=ON -DARROW_PARQUET=ON -DARROW_SIMD_LEVEL=NONE -DARROW_VERBOSE_THIRDPARTY_BUILD=ON
#          -DARROW_BUILD_SHARED=1 -DARROW_BUILD_UTILITIES=OFF -DARROW_BUILD_INTEGRATION=OFF
#          -DBoost_FOUND=1 -DARROW_TEST_LINKAGE="shared"
# make -j8
# copy {BUILD_DIR}/contrib/arrow/cpp/src/parquet/*.cpp,*.h -> {BUILD_DIR}/contrib/arrow-cmake/cpp/src/parquet/

# Also useful parquet reader:
# cd {BUILD_DIR}/contrib/arrow/cpp && mkdir -p build && cd build
# cmake .. -DARROW_PARQUET=1 -DARROW_WITH_SNAPPY=1 -DPARQUET_BUILD_EXECUTABLES=1
# make -j8
# {BUILD_DIR}/contrib/arrow/cpp/build/release/parquet-reader some_file.parquet

set (ENABLE_PARQUET_DEFAULT ${ENABLE_LIBRARIES})
if (OS_FREEBSD)
    set (ENABLE_PARQUET_DEFAULT OFF)
endif()
option (ENABLE_PARQUET "Enable parquet" ${ENABLE_PARQUET_DEFAULT})

if (NOT ENABLE_PARQUET)
    message(STATUS "Not using parquet")
    return()
endif()

# Freebsd: ../contrib/arrow/cpp/src/arrow/util/bit-util.h:27:10: fatal error: endian.h: No such file or directory
if (OS_FREEBSD)
    message (FATAL_ERROR "Using internal parquet library on FreeBSD is not supported")
endif()

# Remove jemalloc usage for Arrow; always use system allocator

set (CMAKE_CXX_STANDARD 17)

set(ARROW_VERSION "11.0.0")
string(REGEX MATCH "^[0-9]+\\.[0-9]+\\.[0-9]+" ARROW_BASE_VERSION "${ARROW_VERSION}")

set(ARROW_VERSION_MAJOR "11")
set(ARROW_VERSION_MINOR "0")
set(ARROW_VERSION_PATCH "0")

if(ARROW_VERSION_MAJOR STREQUAL "0")
    # Arrow 0.x.y => SO version is "x", full SO version is "x.y.0"
    set(ARROW_SO_VERSION "${ARROW_VERSION_MINOR}")
    set(ARROW_FULL_SO_VERSION "${ARROW_SO_VERSION}.${ARROW_VERSION_PATCH}.0")
else()
    # Arrow 1.x.y => SO version is "10x", full SO version is "10x.y.0"
    math(EXPR ARROW_SO_VERSION "${ARROW_VERSION_MAJOR} * 100 + ${ARROW_VERSION_MINOR}")
    set(ARROW_FULL_SO_VERSION "${ARROW_SO_VERSION}.${ARROW_VERSION_PATCH}.0")
endif()


# === orc
set(ORC_SOURCE_DIR "${ClickHouse_SOURCE_DIR}/contrib/orc/c++")
set(ORC_INCLUDE_DIR "${ORC_SOURCE_DIR}/include")
set(ORC_SOURCE_SRC_DIR "${ORC_SOURCE_DIR}/src")
# set(ORC_SOURCE_WRAP_DIR "${ORC_SOURCE_DIR}/wrap")

set(ORC_BUILD_SRC_DIR "${CMAKE_CURRENT_BINARY_DIR}/../orc/c++/src")
set(ORC_BUILD_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/../orc/c++/include")

set(ORC_ADDITION_SOURCE_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(ARROW_SRC_DIR "${ClickHouse_SOURCE_DIR}/contrib/arrow/cpp/src")

set(PROTOBUF_EXECUTABLE $<TARGET_FILE:protoc>)
set(PROTO_DIR "${ORC_SOURCE_DIR}/../proto")


add_custom_command(OUTPUT orc_proto.pb.h orc_proto.pb.cc
        COMMAND ${PROTOBUF_EXECUTABLE}
        -I ${PROTO_DIR}
        --cpp_out="${CMAKE_CURRENT_BINARY_DIR}"
        "${PROTO_DIR}/orc_proto.proto")


# === flatbuffers
set(FLATBUFFERS_SRC_DIR "${ClickHouse_SOURCE_DIR}/contrib/flatbuffers")
set(FLATBUFFERS_BINARY_DIR "${ClickHouse_BINARY_DIR}/contrib/flatbuffers")
set(FLATBUFFERS_INCLUDE_DIR "${FLATBUFFERS_SRC_DIR}/include")

set(FLATBUFFERS_SRCS
  ${FLATBUFFERS_SRC_DIR}/src/idl_parser.cpp
  ${FLATBUFFERS_SRC_DIR}/src/idl_gen_text.cpp
  ${FLATBUFFERS_SRC_DIR}/src/reflection.cpp
  ${FLATBUFFERS_SRC_DIR}/src/util.cpp)

add_library(_flatbuffers STATIC ${FLATBUFFERS_SRCS})
target_include_directories(_flatbuffers PUBLIC ${FLATBUFFERS_INCLUDE_DIR})
target_compile_definitions(_flatbuffers PRIVATE -DFLATBUFFERS_LOCALE_INDEPENDENT=0)


# ===  hdfs
# NOTE: cannot use ch_contrib::hdfs since it's INCLUDE_DIRECTORIES does not includes trailing "hdfs/"
set(HDFS_INCLUDE_DIR "${ClickHouse_SOURCE_DIR}/contrib/libhdfs3/include/hdfs/")

# arrow-cmake cmake file calling orc cmake subroutine which detects certain compiler features.
# Apple Clang compiler failed to compile this code without specifying c++11 standard.
# As result these compiler features detected as absent. In result it failed to compile orc itself.
# In orc makefile there is code that sets flags, but arrow-cmake ignores these flags.
if (CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    set(CXX11_FLAGS "-std=c++0x")
endif ()

set (ORC_CXX_HAS_INITIALIZER_LIST 1)
set (ORC_CXX_HAS_NOEXCEPT 1)
set (ORC_CXX_HAS_NULLPTR 1)
set (ORC_CXX_HAS_OVERRIDE 1)
set (ORC_CXX_HAS_UNIQUE_PTR 1)
set (ORC_CXX_HAS_CSTDINT 1)
set (ORC_CXX_HAS_THREAD_LOCAL 1)

include(orc_check.cmake)
configure_file("${ORC_SOURCE_SRC_DIR}/Adaptor.hh.in" "${ORC_BUILD_INCLUDE_DIR}/Adaptor.hh")


set(ORC_SRCS
        "${CMAKE_CURRENT_BINARY_DIR}/orc_proto.pb.h"
        "${ORC_ADDITION_SOURCE_DIR}/orc_proto.pb.cc"
        "${ORC_SOURCE_SRC_DIR}/Adaptor.cc"
        # ... rest of ORC_SRCS ...
        )

add_library(_orc STATIC ${ORC_SRCS})
target_link_libraries(_orc PRIVATE
    ch_contrib::protobuf
    ch_contrib::lz4
    ch_contrib::snappy
    ch_contrib::zlib
    ch_contrib::zstd)
target_include_directories(_orc SYSTEM BEFORE PUBLIC
    ${ORC_INCLUDE_DIR}
    "${ClickHouse_SOURCE_DIR}/contrib/arrow-cmake/cpp/src/orc/c++/include")
# Removed jemalloc link
target_include_directories(_orc SYSTEM BEFORE PUBLIC ${ORC_BUILD_INCLUDE_DIR})
target_include_directories(_orc SYSTEM PRIVATE
    ${ORC_SOURCE_SRC_DIR}
    ${ORC_SOURCE_WRAP_DIR}
    ${ORC_BUILD_SRC_DIR}
    ${ORC_ADDITION_SOURCE_DIR}
    ${ARROW_SRC_DIR})


# === arrow

set(LIBRARY_DIR "${ClickHouse_SOURCE_DIR}/contrib/arrow/cpp/src/arrow")

set(ARROW_SRCS
        "${LIBRARY_DIR}/array/array_base.cc"
        # ... rest of ARROW_SRCS ...
        "${LIBRARY_DIR}/util/compression_brotli.cc"
)

# Removed memory_pool_jemalloc.cc from ARROW_SRCS

add_library(_arrow STATIC ${ARROW_SRCS})

target_link_libraries(_arrow PRIVATE
    boost::filesystem
    _flatbuffers
    ch_contrib::double_conversion
    ch_contrib::lz4
    ch_contrib::snappy
    ch_contrib::zlib
    ch_contrib::zstd
    ch_contrib::brotli
)
# Removed jemalloc link
target_link_libraries(_arrow PUBLIC _orc)

add_dependencies(_arrow protoc)

target_include_directories(_arrow SYSTEM BEFORE PUBLIC ${ARROW_SRC_DIR})
target_include_directories(_arrow SYSTEM BEFORE PUBLIC "${ClickHouse_SOURCE_DIR}/contrib/arrow-cmake/cpp/src")

target_include_directories(_arrow SYSTEM PRIVATE ${ARROW_SRC_DIR})
target_include_directories(_arrow SYSTEM PRIVATE ${HDFS_INCLUDE_DIR})

# === parquet

set(LIBRARY_DIR "${ClickHouse_SOURCE_DIR}/contrib/arrow/cpp/src/parquet")
set(GEN_LIBRARY_DIR "${ClickHouse_SOURCE_DIR}/contrib/arrow/cpp/src/generated")

set(PARQUET_SRCS
        "${LIBRARY_DIR}/arrow/path_internal.cc"
        # ... rest of PARQUET_SRCS ...
)

add_library(_parquet STATIC ${PARQUET_SRCS})
add_library(ch_contrib::parquet ALIAS _parquet)
target_include_directories(_parquet SYSTEM BEFORE
    PUBLIC
        "${ClickHouse_SOURCE_DIR}/contrib/arrow/cpp/src"
        "${CMAKE_CURRENT_SOURCE_DIR}/cpp/src")
target_link_libraries(_parquet
    PUBLIC
        _arrow
        ch_contrib::thrift
    PRIVATE
        boost::headers_only
        boost::regex
        OpenSSL::Crypto OpenSSL::SSL
)

# Removed jemalloc link

if (SANITIZE STREQUAL "undefined")
    target_compile_options(_parquet PRIVATE -fno-sanitize=undefined)
    target_compile_options(_arrow PRIVATE -fno-sanitize=undefined)
endif ()

# Define Thrift version for parquet (we use 0.16.0)
add_definitions(-DPARQUET_THRIFT_VERSION_MAJOR=0)
add_definitions(-DPARQUET_THRIFT_VERSION_MINOR=16)

# === tools

set(TOOLS_DIR "${ClickHouse_SOURCE_DIR}/contrib/arrow/cpp/tools/parquet")
set(PARQUET_TOOLS parquet_dump_schema parquet_reader parquet_scan)
foreach (TOOL ${PARQUET_TOOLS})
    add_executable(${TOOL} "${TOOLS_DIR}/${TOOL}.cc")
    target_link_libraries(${TOOL} PRIVATE _parquet)
endforeach ()

# The library is large - avoid bloat.
if (OMIT_HEAVY_DEBUG_SYMBOLS)
    target_compile_options (_arrow PRIVATE -g0)
    target_compile_options (_parquet PRIVATE -g0)
endif()
