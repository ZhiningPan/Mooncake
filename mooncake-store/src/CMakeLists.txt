# Find Python package
add_subdirectory(cachelib_memory_allocator)

set(MOONCAKE_STORE_SOURCES
    allocator.cpp
    master_service.cpp
    master_snapshot_manager.cpp
    master_snapshot_repository.cpp
    client_service.cpp
    client_metric.cpp
    types.cpp
    master_client.cpp
    utils.cpp
    mmap_arena.cpp
    master_metric_manager.cpp
    storage_backend.cpp
    thread_pool.cpp
    etcd_helper.cpp
    segment.cpp
    transfer_task.cpp
    tenant_quota.cpp
    tenant_quota_policy_store.cpp
    rpc_service.cpp
    master_admin_service.cpp
    offset_allocator.cpp
    posix_file.cpp
    client_buffer.cpp
    aligned_client_buffer.cpp
    real_client.cpp
    registered_pinned_memory.cpp
    dummy_client.cpp
    uds_transport.cpp
    shm_helper.cpp
    http_metadata_server.cpp
    file_storage.cpp
    serialize/serializer.cpp
    storage/distributed/distributed_storage_backend.cpp
    device/accelerator_device.cpp
    device/accelerator_registry.cpp
    device/runtime_accelerator.cpp
    device/cuda_like_accelerator_device.cpp
    device/hip_accelerator_device.cpp
    device/ascend_accelerator_device.cpp
    device/sunrise_accelerator_device.cpp
    ha/leadership/leader_coordinator_factory.cpp
    ha/leadership/backends/etcd/etcd_leader_coordinator.cpp
    ha/common/redis/redis_connection.cpp
    ha/leadership/backends/redis/redis_leader_coordinator.cpp
    ha/leadership/master_service_supervisor.cpp
    ha/master_metrics_reporter.cpp
    ha/kv/etcd_ha_kv_backend.cpp
    ha/standby_controller.cpp
    ha/snapshot/catalog_backed_snapshot_provider.cpp
    ha/snapshot/master_snapshot_codec.cpp
    ha/snapshot/object/snapshot_object_store.cpp
    ha/snapshot/object/backends/local/local_file_snapshot_object_store.cpp
    ha/snapshot/object/backends/s3/s3_snapshot_object_store.cpp
    ha/snapshot/catalog/backends/embedded/embedded_snapshot_catalog_store.cpp
    ha/snapshot/catalog/backends/redis/redis_snapshot_catalog_store.cpp
    utils/type_util.cpp
    utils/file_util.cpp
    task_manager.cpp
    local_hot_cache.cpp
    ha/oplog/oplog_types.cpp
    ha/oplog/oplog_batch_codec.cpp
    ha/oplog/oplog_batch_storage.cpp
    ha/oplog/oplog_batch_standby_reader.cpp
    ha/oplog/oplog_batch_types.cpp
    ha/oplog/ordered_oplog_writer.cpp
    ha/oplog/oplog_test_failpoint.cpp
    ha/oplog/oplog_applier.cpp
    hot_standby_service.cpp
    metadata_store.cpp
    standby_state_machine.cpp
    ha_metric_manager.cpp
    store_c.cpp
    memory_alloc.cpp
    ssd_register_client.cpp
    engram/engram_store.cpp)

set(EXTRA_LIBS "")
set(SPDK_STATIC_LIBS "")

# Find AWS SDK
find_package(AWSSDK QUIET COMPONENTS s3)
if(AWSSDK_FOUND)
  message(STATUS "AWS SDK found: ${AWSSDK_VERSION}")
  set(HAVE_AWS_SDK TRUE)
  # Add S3 related source files
  list(APPEND MOONCAKE_STORE_SOURCES utils/s3_helper.cpp)
else()
  message(STATUS "AWS SDK not found, S3 functionality will be disabled")
  set(HAVE_AWS_SDK FALSE)
endif()

# Find zstd library
find_library(ZSTD_LIBRARY NAMES zstd)
if(NOT ZSTD_LIBRARY)
  message(FATAL_ERROR "zstd library not found")
endif()

set(EXTRA_LIBS ${ZSTD_LIBRARY})

# If AWS SDK is found, add it to EXTRA_LIBS
if(HAVE_AWS_SDK)
  list(APPEND EXTRA_LIBS ${AWSSDK_LINK_LIBRARIES})
  add_definitions(-DHAVE_AWS_SDK)
  if(AWS_S3_LIB AND AWS_CORE_LIB)
    list(APPEND EXTRA_LIBS ${AWS_S3_LIB} ${AWS_CORE_LIB})
    add_definitions(-DHAVE_AWS_SDK)
  endif()
endif()

# Find xxHash (required for ComputeChecksum)
find_path(
  XXHASH_INCLUDE_DIR
  NAMES xxhash.h
  PATHS /usr/include /usr/local/include)
find_library(
  XXHASH_LIBRARY
  NAMES xxhash libxxhash
  PATHS /usr/lib /usr/local/lib /usr/lib64)
set(KV_EVENTS_ZMQ_FOUND FALSE)
if(ENABLE_KV_EVENTS)
  find_library(
    ZMQ_LIBRARY
    NAMES zmq libzmq
    PATHS /usr/lib /usr/local/lib /usr/lib64)
  find_path(
    ZMQ_INCLUDE_DIR
    NAMES zmq.h
    PATHS /usr/include /usr/local/include)
  if(ZMQ_INCLUDE_DIR AND ZMQ_LIBRARY)
    message(STATUS "Found ZMQ: include=${ZMQ_INCLUDE_DIR} lib=${ZMQ_LIBRARY}")
    list(APPEND MASTER_EXTRA_INCS ${ZMQ_INCLUDE_DIR})
    list(APPEND EXTRA_LIBS ${ZMQ_LIBRARY})
    list(APPEND MOONCAKE_STORE_SOURCES kv_event/kv_event_publisher.cpp)
    set(KV_EVENTS_ZMQ_FOUND TRUE)
  else()
    message(
      FATAL_ERROR
        "ENABLE_KV_EVENTS is ON but libzmq was not found. "
        "Install libzmq3-dev or pass -DENABLE_KV_EVENTS=OFF to configure without ZMQ."
    )
  endif()
endif()

if(XXHASH_INCLUDE_DIR AND XXHASH_LIBRARY)
  message(
    STATUS "Found xxHash: include=${XXHASH_INCLUDE_DIR} lib=${XXHASH_LIBRARY}")
  list(APPEND MASTER_EXTRA_INCS ${XXHASH_INCLUDE_DIR})
else()
  message(
    FATAL_ERROR
      "xxHash library/header not found. Please install xxhash (development headers) and try again."
  )
endif()

if(USE_3FS)
  add_subdirectory(hf3fs)
  list(APPEND MOONCAKE_STORE_SOURCES ${HF3FS_SOURCES})
  list(APPEND MOONCAKE_STORE_SOURCES storage/distributed/hf3fs_adapter.cpp)
  find_library(
    HF3FS_API_LIB hf3fs_api_shared
    PATHS /usr/lib
    NO_DEFAULT_PATH)
  if(NOT HF3FS_API_LIB)
    message(FATAL_ERROR "hf3fs_api_shared library not found in /usr/lib")
  endif()
  list(APPEND EXTRA_LIBS ${HF3FS_API_LIB})
endif()

# io_uring support (auto-detected)
find_library(URING_LIB uring PATHS /usr/lib /usr/lib64 /usr/local/lib
                                   /usr/local/lib64)
find_path(URING_INCLUDE liburing.h PATHS /usr/include /usr/local/include)
if(URING_LIB AND URING_INCLUDE)
  message(STATUS "io_uring: Enabled for mooncake_store")
  list(APPEND MOONCAKE_STORE_SOURCES uring_file.cpp)
  list(APPEND EXTRA_LIBS ${URING_LIB})
else()
  message(STATUS "io_uring: Disabled (liburing not found)")
endif()

if(STORE_USE_REDIS)
  list(APPEND EXTRA_LIBS ${MOONCAKE_STORE_HIREDIS_LIBRARY})
endif()

list(APPEND MOONCAKE_STORE_SOURCES k8s_lease_helper.cpp)
if(STORE_USE_K8S_LEASE)
  list(APPEND MOONCAKE_STORE_SOURCES
       ha/leadership/backends/k8s/k8s_leader_coordinator.cpp)
  list(APPEND EXTRA_LIBS ${K8S_LEASE_WRAPPER_LIB})
  set_source_files_properties(
    k8s_lease_helper.cpp
    PROPERTIES
      OBJECT_DEPENDS
      "${CMAKE_BINARY_DIR}/mooncake-common/k8s-lease/libk8s_lease_wrapper.h")
endif()

if(USE_NOF)
  add_subdirectory(spdk)
  list(APPEND MOONCAKE_STORE_SOURCES ${SPDK_WRAPPER_SOURCES})

  set(SPDK_INCLUDE_DIR
      "/usr/local/include"
      CACHE PATH "SPDK include directory")
  set(DPDK_INCLUDE_DIR
      "/usr/local/include"
      CACHE PATH "DPDK include directory")
  set(SPDK_LIB_DIR
      "/usr/local/lib"
      CACHE PATH "SPDK static library directory")
  set(DPDK_LIB_DIR
      "/usr/local/lib"
      CACHE PATH "DPDK static library directory")

  # Modified By Yida (v3): SPDK 版本兼容链接清单（v23.01 上游基线 / 26.x 本项目 fork）。
  # - 核心库 event/nvme 任何版本都必须存在，缺失 = make install 没跑完，直接报错
  # - 版本差异库按存在与否静默引入：26.x 将 lib/rdma 拆为 rdma_cm/rdma_provider/
  #   rdma_utils（未开 --with-rdma 则无）、sock_posix 并入 libspdk_sock.a、
  #   blobfs 已移除；v23.01 为单库 rdma
  # - 其余库任何版本都应存在，缺失给 WARNING（多半是 install 不完整）
  foreach(_lib event nvme)
    if(NOT EXISTS "${SPDK_LIB_DIR}/libspdk_${_lib}.a")
      message(FATAL_ERROR
        "SPDK lib missing: ${SPDK_LIB_DIR}/libspdk_${_lib}.a — "
        "回 SPDK 目录执行 sudo make install（部署文档阶段 2）")
    endif()
  endforeach()

  set(SPDK_CORE_LIBS)
  foreach(_lib event nvme sock_posix rdma rdma_cm rdma_provider rdma_utils)
    if(EXISTS "${SPDK_LIB_DIR}/libspdk_${_lib}.a")
      list(APPEND SPDK_CORE_LIBS ${SPDK_LIB_DIR}/libspdk_${_lib}.a)
    endif()
  endforeach()

  set(SPDK_OPTIONAL_LIBS)
  # Modified By Yida (v3): keyring 系列是按需编译的子库（v26.09 新增），
  # 旧版 SPDK 没有这些库——放进 OPTIONAL 列表，存在则链接，缺失静默跳过
  foreach(_lib vfio_user dma init env_dpdk log util blob bdev conf json jsonrpc sock thread trace vmd rpc keyring keyring_file keyring_linux)
    if(EXISTS "${SPDK_LIB_DIR}/libspdk_${_lib}.a")
      list(APPEND SPDK_OPTIONAL_LIBS ${SPDK_LIB_DIR}/libspdk_${_lib}.a)
    else()
      message(WARNING "SPDK lib not found (skipped): libspdk_${_lib}.a in ${SPDK_LIB_DIR}")
    endif()
  endforeach()

  set(SPDK_STATIC_LIBS
      -Wl,--whole-archive
      ${SPDK_CORE_LIBS}
      -Wl,--no-whole-archive
      ${SPDK_OPTIONAL_LIBS}
      ${DPDK_LIB_DIR}/librte_eal.a
      ${DPDK_LIB_DIR}/librte_mempool.a
      ${DPDK_LIB_DIR}/librte_ring.a
      ${DPDK_LIB_DIR}/librte_bus_pci.a
      ${DPDK_LIB_DIR}/librte_pci.a
      ${DPDK_LIB_DIR}/librte_timer.a
      ${DPDK_LIB_DIR}/librte_telemetry.a
      ${DPDK_LIB_DIR}/librte_kvargs.a
      ${DPDK_LIB_DIR}/librte_mbuf.a
      -Wl,--whole-archive
      ${DPDK_LIB_DIR}/librte_mempool_ring.a
      -Wl,--no-whole-archive
      ${DPDK_LIB_DIR}/librte_power.a
      ${DPDK_LIB_DIR}/librte_ethdev.a
      ${DPDK_LIB_DIR}/librte_net.a
      ${DPDK_LIB_DIR}/librte_vhost.a
      ${DPDK_LIB_DIR}/librte_cryptodev.a
      ${DPDK_LIB_DIR}/librte_hash.a
      ${DPDK_LIB_DIR}/librte_rcu.a
      # Modified By Yida (v3): DPDK 26.x 将 log/argparse/cmdline 从 librte_eal.a
      # 拆分为独立库（rte_log/rte_argparse_* 符号缺失的根因），必须显式链接
      ${DPDK_LIB_DIR}/librte_log.a
      ${DPDK_LIB_DIR}/librte_argparse.a
      ${DPDK_LIB_DIR}/librte_cmdline.a
      pthread
      dl
      numa
      rt
      m
      uuid
      crypto
      aio
      z
      isal
      elf
      ibverbs
      rdmacm
      ssl
      bsd
      # Modified By Yida (v3): fuse_reply_*/fuse_session_* 等符号需要 fuse3；
      # 必须加在 SPDK_STATIC_LIBS 里（而不是 mooncake_master 的 target_link_libraries），
      # 否则 PUBLIC 传播顺序导致符号解析失败（部署文档第三节问题三）
      /usr/lib64/libfuse3.so
      # Modified By Yida (v3): keyring 三件套显式加在末尾（实测写法；
      # 与上面 OPTIONAL foreach 重复无害，静态库按需取 object）
      ${SPDK_LIB_DIR}/libspdk_keyring.a
      ${SPDK_LIB_DIR}/libspdk_keyring_file.a
      ${SPDK_LIB_DIR}/libspdk_keyring_linux.a)

  # Modified By Yida (v3): URMA transport (UMDK)。libspdk_nvme.a 中的 URMA
  # object 引用 liburma 符号，必须出现在链接列表（此处 SPDK_STATIC_LIBS 整体
  # 挂到 mooncake_store 的 PUBLIC libs，位置在 nvme 之后满足符号解析）。
  # 通过 -DUMDK_ROOT=/path/to/umdk-prefix 指定 UMDK 安装前缀。
  if(USE_NOF_URMA)
    # Modified By Yida (v6): /usr/local 装的是不含 URMA transport 的 SPDK
    # （如上游 v23.01 残留）时，缺符号/缺声明会拖到编译或链接期且难定位
    # （系统遮蔽——报错路径含 /usr/local/include 即旧版遮蔽的指纹）。
    # nvme_urma.h 是本项目 fork SPDK 的标志头文件，在此提前校验。
    if(NOT EXISTS "${SPDK_INCLUDE_DIR}/spdk/nvme_urma.h")
      message(
        FATAL_ERROR
          "USE_NOF_URMA=ON but ${SPDK_INCLUDE_DIR}/spdk/nvme_urma.h not found — "
          "installed SPDK has no URMA transport. 确认 urma_modified_v6 分支产物"
          "已 make install 到 /usr/local（部署文档阶段 2），或用 -DSPDK_INCLUDE_DIR="
          "指向正确前缀")
    endif()
    set(UMDK_ROOT
        ""
        CACHE PATH "UMDK install prefix for the SPDK URMA transport")
    find_path(
      URMA_INCLUDE_DIR
      NAMES urma_api.h
      HINTS ${UMDK_ROOT}
      PATHS /usr /usr/local
      PATH_SUFFIXES include/ub/umdk/urma include/ub/umdk include)
    find_library(
      URMA_LIBRARY
      NAMES urma
      HINTS ${UMDK_ROOT}
      PATHS /usr /usr/local
      PATH_SUFFIXES lib64 lib build/lib)
    if(NOT URMA_INCLUDE_DIR OR NOT URMA_LIBRARY)
      message(
        FATAL_ERROR
          "USE_NOF_URMA=ON but UMDK (urma_api.h / liburma) not found; \
pass -DUMDK_ROOT=/path/to/umdk-prefix")
    endif()
    message(STATUS "NoF URMA: include=${URMA_INCLUDE_DIR} lib=${URMA_LIBRARY}")
    list(APPEND SPDK_STATIC_LIBS ${URMA_LIBRARY})
  endif()

  # Modified By Yida (v3): NoF GPU Direct —— provider 使用 CUDA driver API
  # (cuMemGetAddressRange / DMA-BUF export)。提前探测 CUDAToolkit，供后段
  # USE_CUDA 块复用（后续会统一加 USE_CUDA 宏、cudart 与 include 目录）。
  if(USE_NOF_GPU_DIRECT)
    find_package(CUDAToolkit REQUIRED)
    message(STATUS "NoF GPU Direct: CUDA driver library required for provider")
    list(APPEND SPDK_STATIC_LIBS CUDA::cuda_driver)
  endif()
endif()

# The cache_allocator library
include_directories(${Python3_INCLUDE_DIRS})
add_library(mooncake_store ${MOONCAKE_STORE_SOURCES})
if(KV_EVENTS_ZMQ_FOUND)
  target_compile_definitions(mooncake_store PRIVATE MOONCAKE_ENABLE_KV_EVENTS=1)
endif()
if(MOONCAKE_ENABLE_OPLOG_PERF_METRICS)
  target_compile_definitions(mooncake_store
                             PRIVATE MOONCAKE_ENABLE_OPLOG_PERF_METRICS)
endif()
target_include_directories(mooncake_store PUBLIC ${XXHASH_INCLUDE_DIR})
if(USE_NOF)
  target_include_directories(mooncake_store PRIVATE ${SPDK_INCLUDE_DIR}
                                                    ${DPDK_INCLUDE_DIR})
  if(USE_NOF_URMA)
    target_include_directories(mooncake_store PRIVATE ${URMA_INCLUDE_DIR})
  endif()
endif()
target_link_libraries(mooncake_store PUBLIC ${XXHASH_LIBRARY})
if(STORE_USE_REDIS)
  target_include_directories(mooncake_store
                             PRIVATE ${MOONCAKE_STORE_HIREDIS_INCLUDE_DIR})
endif()
# Note: transfer_engine is PRIVATE to avoid propagating its dependencies (e.g.,
# vendor-specific hardware) to targets that don't need it (e.g.,
# mooncake_master). Targets that need transfer_engine should link it explicitly.
target_link_libraries(
  mooncake_store
  PUBLIC cachelib_memory_allocator
         ${ETCD_WRAPPER_LIB}
         glog::glog
         gflags::gflags
         ${EXTRA_LIBS}
         ${SPDK_STATIC_LIBS}
         mooncake_common
         yaml-cpp
         asio_shared
  PRIVATE transfer_engine)

# SpDiag instrumentation
include(${CMAKE_SOURCE_DIR}/mooncake-common/FindSpDiag.cmake)
target_link_libraries(mooncake_store PRIVATE SpDiag::spdiag_lib)
target_include_directories(mooncake_store PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/../../mooncake-integration/store)

if(STORE_USE_ETCD)
  add_dependencies(mooncake_store build_etcd_wrapper)
endif()
if(STORE_USE_K8S_LEASE)
  add_dependencies(mooncake_store build_k8s_lease_wrapper)
endif()

if(URING_LIB AND URING_INCLUDE)
  target_compile_definitions(mooncake_store PUBLIC USE_URING)
  target_include_directories(mooncake_store PRIVATE ${URING_INCLUDE})
endif()

if(BUILD_SHARED_LIBS)
  install(TARGETS mooncake_store DESTINATION lib)
endif()

# Master binary
include(CheckPIESupported)
check_pie_supported(LANGUAGES CXX)

add_executable(mooncake_master master.cpp)

set(MASTER_EXTRA_INCS)
set(MASTER_EXTRA_LIBS)

if(STORE_USE_JEMALLOC)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(JEMALLOC REQUIRED jemalloc)
  list(APPEND MASTER_EXTRA_INCS ${JEMALLOC_INCLUDE_DIRS})
  list(APPEND MASTER_EXTRA_LIBS ${JEMALLOC_STATIC_LIBRARIES})
endif()

target_include_directories(mooncake_master PRIVATE ${MASTER_EXTRA_INCS})
target_link_libraries(
  mooncake_master
  PRIVATE mooncake_store
          cachelib_memory_allocator
          pthread
          ibverbs
          mooncake_common
          JsonCpp::JsonCpp
          ${ETCD_WRAPPER_LIB}
          ${MASTER_EXTRA_LIBS}
          asio_shared
          SpDiag::spdiag_lib)
# mooncake_store is static and propagates optional runtime dependencies to its
# consumers. The metadata-only master does not use accelerator staging, so drop
# those dependencies when they do not satisfy any symbols in the final binary.
target_link_options(mooncake_master PRIVATE "LINKER:--as-needed")
if(USE_SUNRISE)
  target_link_directories(mooncake_master PRIVATE ${MC_TANGRT_ROOT}/lib)
endif()

if(STORE_USE_ETCD)
  add_dependencies(mooncake_master build_etcd_wrapper)
endif()

# Client server binary
add_executable(mooncake_client real_client_main.cpp)
# Client needs transfer_engine for data transfer operations
target_link_libraries(mooncake_client PRIVATE mooncake_store transfer_engine
                                              asio_shared
                                              SpDiag::spdiag_lib)
set_target_properties(mooncake_master mooncake_client
                      PROPERTIES POSITION_INDEPENDENT_CODE ON)

# Optimize binary sizes only in Release mode
string(TOUPPER "${CMAKE_BUILD_TYPE}" CMAKE_BUILD_TYPE_UPPER)
if(CMAKE_BUILD_TYPE_UPPER STREQUAL "RELEASE")
  target_compile_options(mooncake_master PRIVATE -Os)
  target_link_options(mooncake_master PRIVATE -Os -s)
  target_compile_options(mooncake_client PRIVATE -Os)
  target_link_options(mooncake_client PRIVATE -Os -s)
endif()

# GPU runtime library for D2H staging in PutToLocalFile / OffloadObjects.
# transfer_engine is PRIVATE-linked, so its CUDA/HIP/Ascend dependencies are not
# propagated; we must detect and link them independently.
#
# Auto-detect each toolkit regardless of global USE_CUDA/USE_HIP flags, because
# USE_CUDA may be OFF even when GPU pointers are present (e.g.
# WITH_NVIDIA_PEERMEM env var uses nvidia-peermem for RDMA without cudart). Each
# detected toolkit gets both link libraries AND compile definitions, so that
# AcceleratorDevice implementations and pinned_buffer_pool.h pick the correct
# backend.
#
# NOTE: mooncake_store is a static library (.a). External consumers (Go CGo,
# Python pybind) that link it must also link the GPU runtime (e.g. -lcudart).
# Go's build.sh already does this; CI workflows must do the same.
if(USE_CUDA)
    find_package(CUDAToolkit QUIET)
endif()
if(CUDAToolkit_FOUND)
  message(STATUS "mooncake_store: CUDAToolkit detected, enabling D2H staging")
  target_compile_definitions(mooncake_store PRIVATE USE_CUDA)
  target_compile_definitions(mooncake_client PRIVATE USE_CUDA)
  target_include_directories(mooncake_store PRIVATE ${CUDAToolkit_INCLUDE_DIRS})
  target_include_directories(mooncake_client
                             PRIVATE ${CUDAToolkit_INCLUDE_DIRS})
  target_link_libraries(mooncake_store PRIVATE CUDA::cudart)
  target_link_libraries(mooncake_client PRIVATE CUDA::cudart)
endif()

if(NOT CUDAToolkit_FOUND)
  find_package(hip QUIET)
  if(hip_FOUND)
    message(STATUS "mooncake_store: HIP detected, enabling D2H staging")
    target_compile_definitions(mooncake_store PRIVATE USE_HIP)
    target_compile_definitions(mooncake_client PRIVATE USE_HIP)
    target_link_libraries(mooncake_store PRIVATE hip::host)
    target_link_libraries(mooncake_client PRIVATE hip::host)
  endif()
endif()

if(USE_ASCEND
   OR USE_ASCEND_DIRECT
   OR USE_UBSHMEM)
  target_include_directories(mooncake_store PRIVATE $ENV{ASCEND_PATH}/include)
  target_link_libraries(mooncake_store PRIVATE ascendcl)
  target_include_directories(mooncake_client PRIVATE $ENV{ASCEND_PATH}/include)
  target_link_libraries(mooncake_client PRIVATE ascendcl)
endif()

if(USE_SUNRISE)
  target_link_libraries(
    mooncake_store PRIVATE ${MC_TANGRT_ROOT}/lib/libtangrt_shared.so
                           ${MC_TANGRT_ROOT}/lib/libptml_shared.so dl)
  target_link_libraries(
    mooncake_client PRIVATE ${MC_TANGRT_ROOT}/lib/libtangrt_shared.so
                            ${MC_TANGRT_ROOT}/lib/libptml_shared.so dl)
endif()

install(TARGETS mooncake_master mooncake_client DESTINATION bin)

# ---------------------------------------------------------------------------
# Optional: a self-contained libmooncake_store.so exporting only the store_c.h
# C ABI, for consumers that dlopen the store (e.g. the Rust crate's `dlopen`
# feature). Whole-archives the static mooncake_store + asio_static (objects are
# -fPIC, see mooncake-common/common.cmake) and localizes everything else via the
# version script; only system/toolkit libs (and the Go etcd/k8s backends when
# enabled) stay DT_NEEDED.
# ---------------------------------------------------------------------------
if(WITH_STORE_C_SHARED)
  # ELF/GNU-ld features (--whole-archive, --as-needed, version script, $ORIGIN);
  # supported by GNU ld, gold, and lld. Linux only.
  if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux")
    message(
      FATAL_ERROR
        "WITH_STORE_C_SHARED currently supports Linux/ELF only "
        "(got CMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}).")
  endif()

  # We whole-archive mooncake_store and reuse its name, so require it to be static
  # (add_library(mooncake_store) follows BUILD_SHARED_LIBS, which some backends
  # force ON).
  get_target_property(_mooncake_store_type mooncake_store TYPE)
  if(NOT _mooncake_store_type STREQUAL "STATIC_LIBRARY")
    message(
      FATAL_ERROR
        "WITH_STORE_C_SHARED requires mooncake_store to be a STATIC_LIBRARY "
        "(got ${_mooncake_store_type}); configure with BUILD_SHARED_LIBS=OFF.")
  endif()

  add_library(mooncake_store_c SHARED store_c_shared.cpp)
  set_target_properties(
    mooncake_store_c
    PROPERTIES OUTPUT_NAME mooncake_store
               # Separate dir so this .so does not shadow the static
               # libmooncake_store.a on the Rust `link` backend's -L path.
               LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/shared"
               # $ORIGIN so the installed .so finds co-installed deps (CMake
               # strips the build RUNPATH on install).
               INSTALL_RPATH "$ORIGIN"
               # Relink when the version script (a bare link flag) changes.
               LINK_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/store_c.map")
  add_dependencies(mooncake_store_c mooncake_store asio_static)
  # Whole-archive the static lib + asio_static via file paths (no interface-dep
  # propagation); re-list the mooncake_store target for its transitive deps.
  # --as-needed drops the now-redundant transitive libasio.so.
  target_link_libraries(
    mooncake_store_c
    PRIVATE -Wl,--whole-archive "$<TARGET_FILE:mooncake_store>"
            "$<TARGET_FILE:asio_static>" -Wl,--no-whole-archive -Wl,--as-needed
            mooncake_store)
  # Export only the C ABI; the version script localizes every other symbol
  # pulled from the static archives. Strip debug info in Release (it dominates
  # the .so size).
  target_link_options(
    mooncake_store_c PRIVATE
    "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/store_c.map"
    "$<$<CONFIG:Release>:-Wl,-s>")
  install(TARGETS mooncake_store_c DESTINATION lib)
endif()
