# Copyright (c) 2023-present The Bitcoin Core developers
# Distributed under the MIT software license, see the accompanying
# file COPYING or https://opensource.org/license/mit/.

configure_file(${PROJECT_SOURCE_DIR}/cmake/bitcoin-build-config.h.in bitcoin-build-config.h USE_SOURCE_PERMISSIONS @ONLY)
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_SOURCE_DIR})

#=============================
# Subprojects
#=============================
# Subprojects include subdirectories that do or could have tests
# and/or benchmark binaries, such as all subtrees and univalue.
# These need to be included before CMAKE_*_OUTPUT_DIRECTORY variables
# are set, so output locations of subproject tests and libraries are
# not overridden.
include(../cmake/crc32c.cmake)
include(../cmake/leveldb.cmake)

#=============================
# secp256k1 subtree
#=============================
message("")
message("Configuring secp256k1 subtree...")
set(SECP256K1_DISABLE_SHARED ON CACHE BOOL "" FORCE)
set(SECP256K1_ENABLE_MODULE_ECDH OFF CACHE BOOL "" FORCE)
set(SECP256K1_ENABLE_MODULE_RECOVERY ON CACHE BOOL "" FORCE)
set(SECP256K1_ENABLE_MODULE_MUSIG OFF CACHE BOOL "" FORCE)
set(SECP256K1_BUILD_BENCHMARK OFF CACHE BOOL "" FORCE)
set(SECP256K1_BUILD_TESTS ${BUILD_TESTS} CACHE BOOL "" FORCE)
set(SECP256K1_BUILD_EXHAUSTIVE_TESTS ${BUILD_TESTS} CACHE BOOL "" FORCE)
if(NOT BUILD_TESTS)
  # Always skip the ctime tests, if we are building no other tests.
  # Otherwise, they are built if Valgrind is available. See SECP256K1_VALGRIND.
  set(SECP256K1_BUILD_CTIME_TESTS ${BUILD_TESTS} CACHE BOOL "" FORCE)
endif()
set(SECP256K1_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
include(GetTargetInterface)
# -fsanitize and related flags apply to both C++ and C,
# so we can pass them down to libsecp256k1 as CFLAGS and LDFLAGS.
get_target_interface(SECP256K1_APPEND_CFLAGS "" sanitize_interface COMPILE_OPTIONS)
string(STRIP "${SECP256K1_APPEND_CFLAGS} ${APPEND_CPPFLAGS}" SECP256K1_APPEND_CFLAGS)
string(STRIP "${SECP256K1_APPEND_CFLAGS} ${APPEND_CFLAGS}" SECP256K1_APPEND_CFLAGS)
set(SECP256K1_APPEND_CFLAGS ${SECP256K1_APPEND_CFLAGS} CACHE STRING "" FORCE)
get_target_interface(SECP256K1_APPEND_LDFLAGS "" sanitize_interface LINK_OPTIONS)
string(STRIP "${SECP256K1_APPEND_LDFLAGS} ${APPEND_LDFLAGS}" SECP256K1_APPEND_LDFLAGS)
set(SECP256K1_APPEND_LDFLAGS ${SECP256K1_APPEND_LDFLAGS} CACHE STRING "" FORCE)
# We want to build libsecp256k1 with the most tested RelWithDebInfo configuration.
enable_language(C)
foreach(config IN LISTS CMAKE_BUILD_TYPE CMAKE_CONFIGURATION_TYPES)
  if(config STREQUAL "")
    continue()
  endif()
  string(TOUPPER "${config}" config)
  set(CMAKE_C_FLAGS_${config} "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
endforeach()
# If the CFLAGS environment variable is defined during building depends
# and configuring this build system, its content might be duplicated.
if(DEFINED ENV{CFLAGS})
  deduplicate_flags(CMAKE_C_FLAGS)
endif()
set(CMAKE_EXPORT_COMPILE_COMMANDS OFF)
add_subdirectory(secp256k1)
set_target_properties(secp256k1 PROPERTIES
  EXCLUDE_FROM_ALL TRUE
)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Set top-level target output locations.
if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
endif()
if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
endif()
if(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
endif()

add_subdirectory(crypto)

add_library(bitcoin_kernel_util STATIC EXCLUDE_FROM_ALL
  util/chaintype.cpp
  util/check.cpp
  util/feefrac.cpp
  util/fs.cpp
  util/fs_helpers.cpp
  util/hasher.cpp
  util/moneystr.cpp
  util/rbf.cpp
  util/readwritefile.cpp
  util/syserror.cpp
  util/signalinterrupt.cpp
  util/strencodings.cpp
  util/threadnames.cpp
  util/time.cpp
  util/tokenpipe.cpp
  logging.cpp
  random.cpp
  randomenv.cpp
  streams.cpp
  support/lockedpool.cpp
  sync.cpp
)

target_link_libraries(bitcoin_kernel_util
  PRIVATE
    core_interface
    bitcoin_crypto
    $<$<PLATFORM_ID:Windows>:ws2_32>
    $<$<PLATFORM_ID:Windows>:iphlpapi>
    $<$<PLATFORM_ID:Windows>:bcrypt>
)

add_library(bitcoin_consensus STATIC EXCLUDE_FROM_ALL
  arith_uint256.cpp
  consensus/merkle.cpp
  consensus/tx_check.cpp
  hash.cpp
  primitives/block.cpp
  primitives/transaction.cpp
  pubkey.cpp
  script/interpreter.cpp
  script/script.cpp
  script/script_error.cpp
  uint256.cpp
)
target_link_libraries(bitcoin_consensus
  PRIVATE
    core_interface
    bitcoin_crypto
    secp256k1
)

add_library(bitcoin_kernel STATIC EXCLUDE_FROM_ALL
  chain.cpp
  coins.cpp
  compressor.cpp
  consensus/tx_verify.cpp
  core_read.cpp
  dbwrapper.cpp
  deploymentinfo.cpp
  deploymentstatus.cpp
  flatfile.cpp
  kernel/chain.cpp
  kernel/chainparams.cpp
  kernel/checks.cpp
  kernel/coinstats.cpp
  kernel/context.cpp
  kernel/cs_main.cpp
  kernel/disconnected_transactions.cpp
  kernel/mempool_removal_reason.cpp
  node/blockstorage.cpp
  node/chainstate.cpp
  node/utxo_snapshot.cpp
  policy/ephemeral_policy.cpp
  policy/feerate.cpp
  policy/packages.cpp
  policy/policy.cpp
  policy/rbf.cpp
  policy/settings.cpp
  policy/truc_policy.cpp
  pow.cpp
  script/sigcache.cpp
  script/solver.cpp
  signet.cpp
  txdb.cpp
  txgraph.cpp
  txmempool.cpp
  validation.cpp
  validationinterface.cpp
  versionbits.cpp
)

target_link_libraries(bitcoin_kernel
  PRIVATE
    core_interface
    bitcoin_consensus
    bitcoin_kernel_util
    leveldb
    secp256k1
    Boost::headers
    $<$<PLATFORM_ID:Windows>:ws2_32>
)

add_library(bitcoinkernel
  bitcoinkernel.cpp
)

get_target_property(kernel_sources bitcoin_kernel SOURCES)
foreach(src ${kernel_sources})
  target_sources(bitcoinkernel PRIVATE "${CMAKE_SOURCE_DIR}/src/${src}")
endforeach()

get_target_property(util_sources bitcoin_kernel_util SOURCES)
foreach(src ${util_sources})
  target_sources(bitcoinkernel PRIVATE "${CMAKE_SOURCE_DIR}/src/${src}")
endforeach()

get_target_property(consensus_sources bitcoin_consensus SOURCES)
foreach(src ${consensus_sources})
  target_sources(bitcoinkernel PRIVATE "${CMAKE_SOURCE_DIR}/src/${src}")
endforeach()

target_link_libraries(bitcoinkernel
  PRIVATE
    core_interface
    bitcoin_crypto
    leveldb
    secp256k1
    $<$<PLATFORM_ID:Windows>:bcrypt>
  PUBLIC
    Boost::headers
)

# libbitcoinkernel requires default symbol visibility, explicitly
# specify that here so that things still work even when user
# configures with -DREDUCE_EXPORTS=ON
#
# Note this is a quick hack that will be removed as we
# incrementally define what to export from the library.
set_target_properties(bitcoinkernel PROPERTIES
  CXX_VISIBILITY_PRESET default
)

# Add a convenience libbitcoinkernel target as a synonym for bitcoinkernel.
add_custom_target(libbitcoinkernel)
add_dependencies(libbitcoinkernel bitcoinkernel)

# When building the static library, install all static libraries the
# bitcoinkernel depends on.
if(NOT BUILD_SHARED_LIBS)
  # Recursively get all the static libraries a target depends on and put them in libs_out
  function(get_target_static_link_libs target libs_out)
    get_target_property(linked_libraries ${target} LINK_LIBRARIES)
    foreach(dep ${linked_libraries})
      if(TARGET ${dep})
        add_dependencies(libbitcoinkernel ${dep})
        get_target_property(dep_type ${dep} TYPE)
        if(dep_type STREQUAL "STATIC_LIBRARY")
          list(APPEND ${libs_out} ${dep})
          get_target_static_link_libs(${dep} ${libs_out})
        endif()
      endif()
    endforeach()
    set(${libs_out} ${${libs_out}} PARENT_SCOPE)
  endfunction()

  set(all_kernel_static_link_libs "")
  get_target_static_link_libs(bitcoinkernel all_kernel_static_link_libs)

  install(TARGETS ${all_kernel_static_link_libs} ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT libbitcoinkernel)
  list(TRANSFORM all_kernel_static_link_libs PREPEND "-l")
  # LIBS_PRIVATE is substituted in the pkg-config file.
  list(JOIN all_kernel_static_link_libs " " LIBS_PRIVATE)
endif()

configure_file(${PROJECT_SOURCE_DIR}/libbitcoinkernel.pc.in ${PROJECT_BINARY_DIR}/libbitcoinkernel.pc @ONLY)
install(FILES ${PROJECT_BINARY_DIR}/libbitcoinkernel.pc DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig" COMPONENT libbitcoinkernel)

install(TARGETS bitcoinkernel
  RUNTIME
    DESTINATION ${CMAKE_INSTALL_BINDIR}
    COMPONENT libbitcoinkernel
  LIBRARY
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    COMPONENT libbitcoinkernel
  ARCHIVE
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    COMPONENT libbitcoinkernel
)

if(BUILD_UTIL_CHAINSTATE)
  add_executable(bitcoin-chainstate
    bitcoin-chainstate.cpp
  )
  # TODO: The `SKIP_BUILD_RPATH` property setting can be deleted
  #       in the future after reordering Guix script commands to
  #       perform binary checks after the installation step.
  # Relevant discussions:
  # - https://github.com/hebasto/bitcoin/pull/236#issuecomment-2183120953
  # - https://github.com/bitcoin/bitcoin/pull/30312#issuecomment-2191235833
  set_target_properties(bitcoin-chainstate PROPERTIES
    SKIP_BUILD_RPATH OFF
  )
  target_link_libraries(bitcoin-chainstate
    PRIVATE
      core_interface
      bitcoinkernel
  )
endif()
