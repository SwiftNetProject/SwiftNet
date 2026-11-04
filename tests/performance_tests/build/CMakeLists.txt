cmake_minimum_required(VERSION 3.15)
project(SwiftNetTests C)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/../output)

if(SANITIZER STREQUAL "address")
    message(STATUS "Using AddressSanitizer")
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address)
elseif(SANITIZER STREQUAL "thread")
    message(STATUS "Using ThreadSanitizer")
    add_compile_options(-fsanitize=thread -fno-omit-frame-pointer)
    add_link_options(-fsanitize=thread)
elseif(SANITIZER STREQUAL "undefined")
    message(STATUS "Using UndefinedBehaviorSanitizer")
    add_compile_options(-fsanitize=undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=undefined)
else()
    message(STATUS "No sanitizer enabled")
endif()

link_directories(${CMAKE_SOURCE_DIR}/../../../build/output)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O0")

file(GLOB SRC_FILES "${CMAKE_SOURCE_DIR}/../src/*.c")

execute_process(
  COMMAND ${CMAKE_C_COMPILER} -dumpmachine
  OUTPUT_VARIABLE TARGET_TRIPLE
  OUTPUT_STRIP_TRAILING_WHITESPACE
  ERROR_QUIET
)

message(STATUS "Compiler target: ${TARGET_TRIPLE}")

if(TARGET_TRIPLE MATCHES "^(aarch64|arm64)")
  add_compile_options(-march=armv8-a)
  set(SN_ARCH aarch64)
elseif(TARGET_TRIPLE MATCHES "^(x86_64|amd64|i[3-6]86)")
  add_compile_options(-march=x86-64 -msse3 -mssse3)
  set(SN_ARCH x86)
endif()

add_executable(run_tests ${SRC_FILES})

if(BACKEND STREQUAL "pcap")
    add_compile_definitions(SWIFT_NET_BACKEND_PCAP)

    if(WIN32)
        find_path(NPCAP_INCLUDE_DIR
            NAMES pcap.h
            PATH_SUFFIXES Include include
            PATHS
                "C:/Npcap"
                "$ENV{NPCAP_SDK}"
        )

        if(SN_ARCH MATCHES aarch64)
        find_library(NPCAP_WPCAP_LIBRARY
            NAMES wpcap
            PATHS
                "C:/Npcap/Lib/ARM64"
                "$ENV{NPCAP_SDK}/Lib/ARM64"
        )

        find_library(NPCAP_PACKET_LIBRARY
            NAMES Packet
            PATHS
                "C:/Npcap/Lib/ARM64"
                "$ENV{NPCAP_SDK}/Lib/ARM64"
        )
        elseif(SN_ARCH MATCHES x86)
        find_library(NPCAP_WPCAP_LIBRARY
            NAMES wpcap
            PATHS
                "C:/Npcap/Lib/x64"
                "$ENV{NPCAP_SDK}/Lib/x64"
        )

        find_library(NPCAP_PACKET_LIBRARY
            NAMES Packet
            PATHS
                "C:/Npcap/Lib/x64"
                "$ENV{NPCAP_SDK}/Lib/x64"
        )
        endif()


        if(NOT NPCAP_INCLUDE_DIR OR
        NOT NPCAP_WPCAP_LIBRARY OR
        NOT NPCAP_PACKET_LIBRARY)
            message(FATAL_ERROR "Npcap SDK not found")
        endif()

        target_include_directories(run_tests PRIVATE
            "${NPCAP_INCLUDE_DIR}"
        )

        target_link_libraries(run_tests PRIVATE
            "${NPCAP_WPCAP_LIBRARY}"
            "${NPCAP_PACKET_LIBRARY}"
        )

        target_link_libraries(run_tests PRIVATE ${CMAKE_SOURCE_DIR}/../../../build/output/libswiftnet.a)
    else()
        target_link_libraries(run_tests PRIVATE -lpcap)

        target_link_libraries(run_tests PRIVATE ${CMAKE_SOURCE_DIR}/../../../build/output/libswiftnet.a)
    endif()
elseif(BACKEND STREQUAL "dpdk")
    add_compile_definitions(SWIFT_NET_BACKEND_DPDK)

    add_library(swiftnet STATIC IMPORTED)

    set_target_properties(swiftnet PROPERTIES
        IMPORTED_LOCATION "${CMAKE_SOURCE_DIR}/../../../build/output/libswiftnet.a"
    )

    target_link_libraries(swiftnet INTERFACE
        rte_eal
        rte_ethdev
        rte_mbuf
        rte_mempool
        rte_ring
        rte_net
    )

    target_link_libraries(run_tests PRIVATE swiftnet)
endif()

if(WIN32)
    target_link_libraries(run_tests PRIVATE
        ws2_32
        iphlpapi
        pthread
    )
endif()


target_link_options(run_tests PRIVATE -O3)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
