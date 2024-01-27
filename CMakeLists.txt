cmake_minimum_required(VERSION 3.20)
project(venok)

set(CMAKE_CXX_STANDARD 17)



if(NOT EXISTS ${VCPKG_ROOT})
    message("VCPKG_ROOT not found. Please, install vcpkg or provide VCPKG_ROOT ${VCPKG_ROOT}")
    set(VCPKG_ROOT "$ENV{VCPKG_ROOT}")
    message("VCPKG_ROOT not found. Please, install vcpkg or provide VCPKG_ROOT ${VCPKG_ROOT}")
    if(NOT EXISTS ${VCPKG_ROOT})
        message(FATAL_ERROR "VCPKG_ROOT not found. Please, install vcpkg or provide VCPKG_ROOT")
    endif ()
endif ()

message("Found VCPKG_ROOT: ${VCPKG_ROOT}")

# Определение поддерживаемых архитектур для каждой операционной системы
set(SUPPORTED_ARCHITECTURES_WINDOWS "x64")
set(SUPPORTED_ARCHITECTURES_LINUX "x64")
set(SUPPORTED_ARCHITECTURES_MACOS "x64" "arm64")

# Определение архитектуры для текущей системы
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(CURRENT_ARCHITECTURES ${SUPPORTED_ARCHITECTURES_WINDOWS})
    set(CURRENT_OS "windows")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(CURRENT_ARCHITECTURES ${SUPPORTED_ARCHITECTURES_LINUX})
    set(CURRENT_OS "linux")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin") # MacOS
    set(CURRENT_ARCHITECTURES ${SUPPORTED_ARCHITECTURES_MACOS})
    set(CURRENT_OS "osx")
else()
    message(FATAL_ERROR "Unsupported operating system: ${CMAKE_SYSTEM_NAME}")
endif()

set(NODE_VERSIONS "v16.0.0" "v18.0.0" "v20.0.0" "v21.0.0")
set(NODE_ABIS "93" "108" "115" "120")
set(ITERATE_ABI 0)

find_program(CURL_EXECUTABLE NAMES curl)
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/targets)

foreach (CURRENT_ARCHITECTURE IN LISTS CURRENT_ARCHITECTURES)
    set(VCPKG_INSTALLED_DIR ${VCPKG_ROOT}/installed)
    message(${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/share/boringssl)
    if(NOT EXISTS ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/share/boringssl)
        message(FATAL_ERROR "BoringSSL not found. Please, install BoringSSL")
    endif ()
    message("Found BoringSSL")

    if(NOT EXISTS ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/share/lsquic)
        message(FATAL_ERROR "lsquic not found. Please, install lsquic")
    endif ()
    message("Found lsquic")

    set(CMAKE_GENERATOR_PLATFORM ${CURRENT_ARCHITECTURE})

    # Build addons
    foreach (NODE_VERSION IN LISTS NODE_VERSIONS)
        list(GET NODE_ABIS ${ITERATE_ABI} NODE_ABI)
        MATH(EXPR ITERATE_ABI "${ITERATE_ABI}+1")

        # Get node headers and lib
        set(HEADERS_URL "https://nodejs.org/dist/${NODE_VERSION}/node-${NODE_VERSION}-headers.tar.gz")
        set(LIB_URL "https://nodejs.org/dist/${NODE_VERSION}/win-x64/node.lib")

        execute_process(COMMAND ${CURL_EXECUTABLE} ${HEADERS_URL} -o "node-${NODE_VERSION}-headers.tar.gz")
        execute_process(COMMAND tar xzf node-${NODE_VERSION}-headers.tar.gz -C targets)
        execute_process(COMMAND ${CURL_EXECUTABLE} ${LIB_URL} -o "targets/node-${NODE_VERSION}/node.lib")

        # Build libs
        add_library("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" SHARED library.cpp)

        set_target_properties("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PROPERTIES
                OUTPUT_NAME "node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}"
                PREFIX ""
                SUFFIX ".node"
        )

        # Include dirs
        # target_include_directories("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${CMAKE_SOURCE_DIR}/include)
        target_include_directories("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/targets/node-${NODE_VERSION}/include/node)
        target_include_directories("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/include)
        target_include_directories("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/include/openssl)
        target_include_directories("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/include/lsquic)

        message(${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/include/openssl)

        # Link libs
        if(WIN32)
            target_link_libraries("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/lib/crypto.lib)
            target_link_libraries("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/lib/ssl.lib)
            target_link_libraries("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/lib/lsquic.lib)
            target_link_libraries("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/targets/node-${NODE_VERSION}/node.lib)
        else ()
            target_link_libraries("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/lib/libcrypto.a)
            target_link_libraries("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/lib/libssl.a)
            target_link_libraries("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${VCPKG_INSTALLED_DIR}/${CURRENT_ARCHITECTURE}-${CURRENT_OS}/lib/liblsquic.a)
            target_link_libraries("node-${CURRENT_OS}-${CURRENT_ARCHITECTURE}-${NODE_ABI}" PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/targets/node-${NODE_VERSION}/node.lib)
        endif ()
    endforeach ()

endforeach ()
