#
#   The Clear BSD License
#
#   Copyright (c) 2022 Samsung Electronics Co., Ltd.
#   All rights reserved.
#
#   Redistribution and use in source and binary forms, with or without
#   modification, are permitted (subject to the limitations in the
#   disclaimer below) provided that the following conditions are met:
#
#   	* Redistributions of source code must retain the above copyright
#   	  notice, this list of conditions and the following disclaimer.
#   	* Redistributions in binary form must reproduce the above copyright
#   	  notice, this list of conditions and the following disclaimer in
#   	  the documentation and/or other materials provided with the distribution.
#   	* Neither the name of Samsung Electronics Co., Ltd. nor the names of its
#   	  contributors may be used to endorse or promote products derived from
#   	  this software without specific prior written permission.
#
#   NO EXPRESS OR IMPLIED LICENSES TO ANY PARTY'S PATENT RIGHTS ARE GRANTED
#   BY THIS LICENSE. THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
#   CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING,
#   BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND
#   FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
#   COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT,
#   INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT
#   NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF
#   USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON
#   ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
#   (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF
#   THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#
cmake_minimum_required(VERSION 2.6)
project(nkv-target)
enable_language(C)
enable_language(CXX)

# Prohibit a common type of an in-source build.
# Note that building in a subdirectory in the source tree is still allowed
# as it can be convenient.
string (COMPARE EQUAL "${CMAKE_SOURCE_DIR}" "${CMAKE_BINARY_DIR}" in_source)
if (in_source)
    message (FATAL_ERROR
"It is not allowed to build the project in its source directory. "
"Please delete CMakeCache.txt and CMakeFiles from ${CMAKE_SOURCE_DIR} "
"if they are there and then use some other directory to build nkv-target."
)
endif ()

add_subdirectory(mk/filelists)
#add_subdirectory(common)

set(CMAKE_C_FLAGS_DEBUG "-DDEBUG -O0 -fno-omit-frame-pointer")

add_custom_target(
  GenConfig ALL
  DEPENDS dfly_config.h
)

set(DFLY_CONFIG_FILE ${CMAKE_CURRENT_BINARY_DIR}/include/dfly_config.h)
add_custom_command ( OUTPUT dfly_config.h
                     COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/include
                     COMMAND echo "Dragonfly Config dir: `pwd`"
                     COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/scripts/genconfig.py > ${DFLY_CONFIG_FILE}
                     WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/include/version.h DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/spdk/include)

# DFLY flags
set(DFLY_LIBS ${DFLY_LIBS} -lpthread -Wl,--no-as-needed -fPIC -lrt -L. -ltbb -lm -march=native)
set(DFLY_CFLAGS -std=gnu++11 -MMD -MP -D_FILE_OFFSET_BITS=64 -fPIC -fpermissive -Wwrite-strings -march=native -include include/dfly_config.h)

#
# KVEMUL library build rules (shared)
#
SET(KVEMUL_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/core/kvemulator/src/kvemul.cpp
)
SET(KVEMUL_HEADERS
    ${CMAKE_CURRENT_SOURCE_DIR}/core/kvemulator/src/kvemul.h
    ${CMAKE_CURRENT_SOURCE_DIR}/core/inc/df_kvemul.h
)
#
# Key Distribution build rules (sources)
#
SET(KD_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/core/kd/src/murmurhash3.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/kd/src/dfly_sh.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/kd/src/dfly_rh.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/kd/src/dfly_kd.cpp
)

SET(KD_HEADERS
    ${CMAKE_CURRENT_SOURCE_DIR}/core/inc/murmurhash3.h
    ${CMAKE_CURRENT_SOURCE_DIR}/core/inc/df_kd.h
)

SET(KV_CMD_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/nvme_samsung_kv_cmd.c
)
#
# QOS library build rules (shared)
#
SET(QOS_SOURCES
#    ${CMAKE_CURRENT_SOURCE_DIR}/core/qos/src/qos.cpp
)
SET(QOS_HEADERS
#    ${CMAKE_CURRENT_SOURCE_DIR}/core/qos/src/qos.h
)

#
# WAL library build rules 
#
SET(ITER_SOURCES
    ${CMAKE_SOURCE_DIR}/core/iter/src/iter_op.cpp
    ${CMAKE_SOURCE_DIR}/core/iter/src/list_op.cpp
    ${CMAKE_SOURCE_DIR}/core/iter/src/list_module.cpp
)
SET(ITER_HEADERS
    ${CMAKE_SOURCE_DIR}/core/inc/df_iter.h
    ${CMAKE_SOURCE_DIR}/core/iter/inc/list_lib.h
)


SET(WAL_SOURCES
    ${CMAKE_SOURCE_DIR}/core/wal/src/wal_hash.cpp
    ${CMAKE_SOURCE_DIR}/core/wal/src/wal_map.cpp
    ${CMAKE_SOURCE_DIR}/core/wal/src/wal_zone_buffer.cpp
    ${CMAKE_SOURCE_DIR}/core/wal/src/wal_log.cpp
    ${CMAKE_SOURCE_DIR}/core/wal/src/wal_module.cpp
)
SET(WAL_HEADERS
    ${CMAKE_SOURCE_DIR}/core/wal/inc/wal_lib.h
    ${CMAKE_SOURCE_DIR}/core/wal/inc/wal_map.h
    ${CMAKE_SOURCE_DIR}/core/wal/inc/wal_zone_buffer.h
    ${CMAKE_SOURCE_DIR}/core/inc/module_hash.h
)


# FUSE library build rules
#
SET(FUSE_SOURCES
    ${CMAKE_SOURCE_DIR}/core/fuse/src/fuse_lib.cpp
    ${CMAKE_SOURCE_DIR}/core/fuse/src/fuse_map.cpp
    ${CMAKE_SOURCE_DIR}/core/fuse/src/fuse_module.cpp
    ${CMAKE_SOURCE_DIR}/core/wal/src/wal_hash.cpp
)
SET(FUSE_HEADERS
    ${CMAKE_SOURCE_DIR}/core/fuse/inc/fuse_lib.h
    ${CMAKE_SOURCE_DIR}/core/fuse/inc/fuse_map.h
    ${CMAKE_SOURCE_DIR}/core/fuse/inc/fuse_module.h
    ${CMAKE_SOURCE_DIR}/core/inc/module_hash.h
)



file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/oss/dssd DESTINATION ${CMAKE_BINARY_DIR})
ADD_CUSTOM_TARGET(dssd
    COMMENT "Building open source DSSD Library"
    COMMAND make -C dssd/lib
    COMMAND make -C dssd/cmd/ustat
    COMMAND export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${CMAKE_BINARY_DIR}/dssd/lib
    COMMAND export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${CMAKE_BINARY_DIR}/dssd/lib/libustat
    COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/include
    COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/lib
    COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/dssd/lib/libdssd.a ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/lib/
)

#
# FRAMEWORK library build rules (shared)
#
SET(FW_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_log.c
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_latency.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_spdk.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_stats.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_subsystem.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_conf.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_namespace.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_request.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_numa.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_qpair.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_ctrl.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_poller.c
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_dev.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_bdev.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_kv_pool_dev.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_mm.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_module.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dss_tpool.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_io_thread.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_lock_service.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_req_handler.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_ops.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_counters.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dfly_blk_stats.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/core/framework/src/dragonfly.cpp
)
SET(FW_HEADERS
    ${CMAKE_CURRENT_SOURCE_DIR}/core/inc/uthash.h
    ${CMAKE_CURRENT_SOURCE_DIR}/core/inc/dragonfly.h
    ${CMAKE_CURRENT_SOURCE_DIR}/oss/dssd/lib/libustat/ustat.h
)

include_directories (${CMAKE_SOURCE_DIR}/include)
include_directories (${CMAKE_BINARY_DIR}/oss/spdk_tcp/include)
include_directories (${CMAKE_BINARY_DIR}/oss/spdk_tcp/lib/nvmf/)
include_directories (${CMAKE_BINARY_DIR}/oss/spdk_tcp/lib/nvme/)
include_directories (${CMAKE_BINARY_DIR}/oss/spdk_tcp/app/nvmf_tgt)
include_directories (${CMAKE_SOURCE_DIR}/stats)
include_directories (${CMAKE_SOURCE_DIR}/core/inc)
include_directories (${CMAKE_SOURCE_DIR}/core/wal/inc)
include_directories (${CMAKE_SOURCE_DIR}/core/iter/inc)
include_directories (${CMAKE_SOURCE_DIR}/core/fuse/inc)
include_directories (${CMAKE_SOURCE_DIR}/oss/dssd/include/)
include_directories (${CMAKE_SOURCE_DIR}/oss/dssd/lib/libustat/)
include_directories (${CMAKE_SOURCE_DIR}/oss/rocksdb/include)
include_directories (${CMAKE_SOURCE_DIR}/include)


option(WITH_KV_STATS "Enable KV Stats library" OFF)
option(WITH_KV_LOG   "Enable KV Logging" OFF)
if(WITH_KV_STATS)

SET(KVV_LIBS    ${KVV_LIBS}    -lkv_stats)
SET(KVSH_LIBS   ${KVSH_LIBS}   -lkv_stats)
SET(SIRIUS_LIBS ${SIRIUS_LIBS} -lkv_stats -L.)

add_definitions(-DCONFIG_KVV_STATS=y)
add_definitions(-DCONFIG_KV_STATS=y)

if(WITH_KV_LOG)
add_definitions(-DCONFIG_KVV_LOG=y)
endif(WITH_KV_LOG)

message("KV Stats is enabled")

# KVV STATS flags
set(KV_STATS_LIBS -lpthread -lconfig -Wl,--no-as-needed -fPIC -lrt -L. -march=native)
set(KV_STATS_CFLAGS  -fPIC -O3 -Wall -Werror -march=native)

SET(KV_STATS_SOURCES
    ${CMAKE_SOURCE_DIR}/stats/kv_stats_lib.c
)

add_library(kv_stats SHARED ${KV_STATS_SOURCES})
target_link_libraries (kv_stats ${KV_STATS_LIBS})
target_compile_options(kv_stats PRIVATE ${KV_STATS_CFLAGS})

endif(WITH_KV_STATS)

option(WITH_DSS_OPEN_SOURCE_RELEASE "Open source release build" ON)

#SPDK tcp
set(OSS_LIBS ${OSS_LIBS} -lpthread -Wl,--no-as-needed -fPIC -lrt -L. -ltbb -lm -march=native)
set(OSS_CFLAGS -g -std=gnu++11 -MMD -MP -D_FILE_OFFSET_BITS=64 -fPIC -fpermissive -Wwrite-strings -march=native)

add_library(oss STATIC ${FW_SOURCES} ${FW_HEADERS} ${ITER_SOURCES} ${ITER_HEADERS} ${WAL_SOURCES} ${WAL_HEADERS} ${FUSE_SOURCES} ${FUSE_HEADERS} ${QOS_SOURCES} ${QOS_HEADERS} ${KD_SOURCES} ${KD_HEADERS})
target_link_libraries (oss ${OSS_LIBS})
target_compile_options(oss PRIVATE ${OSS_CFLAGS})
add_dependencies(oss spdk_tcp_configure)
add_dependencies(oss GenConfig)

add_custom_command(
        TARGET oss POST_BUILD
        COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/lib
        COMMAND cp liboss.a ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/lib
        COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/build/lib/
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/liboss.a ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/build/lib/libspdk_oss.a
        COMMAND export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${CMAKE_BINARY_DIR}/nkv-target/lib
)


ADD_CUSTOM_TARGET(spdk_tcp_configure
  COMMENT "Configuring SPDK with NVMEoTCP Support"
  WORKING_DIRECTORY oss/spdk_tcp
  COMMAND ./configure --disable-unit-tests --disable-tests --enable-debug --with-rdma
  COMMAND make include/spdk/config.h
)

if(WITH_DSS_OPEN_SOURCE_RELEASE)

target_compile_definitions(oss PRIVATE -DDSS_OPEN_SOURCE_RELEASE)
set(DSS_RDB_OSS_EXTRACXXFLAGS EXTRA_CXXFLAGS=-DDSS_OPEN_SOURCE_RELEASE)

ADD_CUSTOM_TARGET(spdk_tcp
  COMMENT "Building SPDK with NVMEoTCP Support"
  WORKING_DIRECTORY oss/spdk_tcp
  COMMAND make V=2 -j16 CONFIG_DSS_OSS=y
  COMMAND make -C test/blobfs/mkfs CONFIG_DSS_OSS=y
  DEPENDS dssd
  DEPENDS spdk_tcp_configure
  DEPENDS oss
  DEPENDS rocksdb_lib
)

else(WITH_DSS_OPEN_SOURCE_RELEASE)

ADD_CUSTOM_TARGET(spdk_tcp
  COMMENT "Building SPDK with NVMEoTCP Support"
  WORKING_DIRECTORY oss/spdk_tcp
  COMMAND make V=2 -j16
  COMMAND make -C test/blobfs/mkfs
  DEPENDS dssd
  DEPENDS spdk_tcp_configure
  DEPENDS oss
  DEPENDS dss_lat
  DEPENDS dss_hsl
  DEPENDS rocksdb_lib
)
endif(WITH_DSS_OPEN_SOURCE_RELEASE)

# cmake file is missing in this directory - ER
# add_subdirectory(test)


execute_process(COMMAND git describe --abbrev=4 --always --tags OUTPUT_VARIABLE DSS_VER OUTPUT_STRIP_TRAILING_WHITESPACE)
add_custom_command(
        TARGET spdk_tcp POST_BUILD
        COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/nkv-target
        COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/bin
        COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/scripts
        COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/include
        COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/include/spdk
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/build/bin/nvmf_tgt ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/bin/
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/test/blobfs/mkfs/mkfs ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/bin/mkfs_blobfs
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/scripts/setup.sh ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/scripts/
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/scripts/common.sh ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/scripts/
        COMMAND cp -r ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/scripts/rpc/ ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/scripts/
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/scripts/rpc.py ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/scripts/
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/scripts/dss_rpc.py ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/scripts/
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/oss/spdk_tcp/include/spdk/pci_ids.h ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/include/spdk/
        COMMAND cp ${CMAKE_SOURCE_DIR}/dss_target.py ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/bin/
        COMMAND cp ${CMAKE_CURRENT_BINARY_DIR}/dssd/cmd/ustat/ustat ${CMAKE_CURRENT_BINARY_DIR}/nkv-target/bin/
        COMMAND cd ${CMAKE_CURRENT_BINARY_DIR}
        COMMAND tar cvzf nkv-target-${DSS_VER}.tgz nkv-target
)

ADD_CUSTOM_TARGET(rocksdb
  COMMENT "Building rocksdb async"
  WORKING_DIRECTORY oss/rocksdb
  COMMAND DISABLE_WARNING_AS_ERROR=1 make -j16 db_bench SPDK_DIR=../spdk_tcp/
  DEPENDS spdk_tcp
)

if(NOT WITH_DSS_OPEN_SOURCE_RELEASE)

add_library(dss_lat STATIC	utils/dss_count_latency.c)
target_link_libraries(dss_lat -ljudyL)
target_compile_options(dss_lat PRIVATE -g -I${CMAKE_SOURCE_DIR}/oss/judy-1.0.5/src -I${CMAKE_SOURCE_DIR}/include/)
add_dependencies(dss_lat judyL)
add_custom_command(
        TARGET dss_lat PRE_BUILD
        COMMENT ${CMAKE_SOURCE_DIR}
        COMMAND cp oss/judy-1.0.5/libjudyL.a ${CMAKE_CURRENT_BINARY_DIR}/
)

ADD_CUSTOM_TARGET(test_latency_counter
    COMMENT "Building test latency counter"
    WORKING_DIRECTORY test/test_latency_counter
    COMMAND make
    DEPENDS dss_lat
)

add_library(dss_hsl STATIC	utils/dss_list_tree.c
							utils/keygen.cc
							utils/crc32.cc
							utils/dss_keygen.c)
target_link_libraries(dss_hsl -ljudyL)
target_compile_options(dss_hsl PRIVATE -g -I${CMAKE_SOURCE_DIR}/oss/rocksdb/include -I${CMAKE_SOURCE_DIR}/oss/judy-1.0.5/src -I${CMAKE_SOURCE_DIR}/include/)
add_dependencies(dss_hsl judyL)

add_custom_command(
        TARGET dss_hsl PRE_BUILD
        COMMENT ${CMAKE_SOURCE_DIR}
        COMMAND cp oss/judy-1.0.5/libjudyL.a ${CMAKE_CURRENT_BINARY_DIR}/
)

ADD_CUSTOM_TARGET(test_hsl
    COMMENT "Building test hsl"
    WORKING_DIRECTORY test/test_hsl
    COMMAND make
    DEPENDS dss_hsl
)

endif(NOT WITH_DSS_OPEN_SOURCE_RELEASE)

ADD_CUSTOM_TARGET(rocksdb_lib
  COMMENT "Building rocksdb async static library"
  WORKING_DIRECTORY oss/rocksdb
  COMMAND make clean SPDK_DIR=../spdk_tcp
  COMMAND DISABLE_WARNING_AS_ERROR=1 make -j16 static_lib SPDK_DIR=../spdk_tcp/ -I../../core/iter/inc/ ${DSS_RDB_OSS_EXTRACXXFLAGS}
  DEPENDS spdk_tcp_configure
)
