if (USE_CLANG_TIDY)
    set (CMAKE_CXX_CLANG_TIDY "${CLANG_TIDY_PATH}")
endif ()

# The `clickhouse` binary is a multi purpose tool that contains multiple execution modes (client, server, etc.),
# each of them may be built and linked as a separate library.
# If you do not know what modes you need, turn this option OFF and enable SERVER and CLIENT only.
option (ENABLE_CLICKHOUSE_ALL "Enable all ClickHouse modes by default" ON)

option (ENABLE_CLICKHOUSE_SERVER "Server mode (main mode)" ${ENABLE_CLICKHOUSE_ALL})
option (ENABLE_CLICKHOUSE_CLIENT "Client mode (interactive tui/shell that connects to the server)"
    ${ENABLE_CLICKHOUSE_ALL})

# https://clickhouse.tech/docs/en/operations/utilities/clickhouse-local/
option (ENABLE_CLICKHOUSE_LOCAL "Local files fast processing mode" ${ENABLE_CLICKHOUSE_ALL})

# https://clickhouse.tech/docs/en/operations/utilities/clickhouse-benchmark/
option (ENABLE_CLICKHOUSE_BENCHMARK "Queries benchmarking mode" ${ENABLE_CLICKHOUSE_ALL})

option (ENABLE_CLICKHOUSE_EXTRACT_FROM_CONFIG "Configs processor (extract values etc.)" ${ENABLE_CLICKHOUSE_ALL})

# https://clickhouse.tech/docs/en/operations/utilities/clickhouse-compressor/
option (ENABLE_CLICKHOUSE_COMPRESSOR "Data compressor and decompressor" ${ENABLE_CLICKHOUSE_ALL})

# https://clickhouse.tech/docs/en/operations/utilities/clickhouse-copier/
option (ENABLE_CLICKHOUSE_COPIER "Inter-cluster data copying mode" ${ENABLE_CLICKHOUSE_ALL})

option (ENABLE_CLICKHOUSE_FORMAT "Queries pretty-printer and formatter with syntax highlighting"
    ${ENABLE_CLICKHOUSE_ALL})

# https://clickhouse.tech/docs/en/operations/utilities/clickhouse-obfuscator/
option (ENABLE_CLICKHOUSE_OBFUSCATOR "Table data obfuscator (convert real data to benchmark-ready one)"
    ${ENABLE_CLICKHOUSE_ALL})

# https://clickhouse.tech/docs/en/operations/utilities/odbc-bridge/
if (ENABLE_ODBC)
    option (ENABLE_CLICKHOUSE_ODBC_BRIDGE "HTTP-server working like a proxy to ODBC driver"
        ${ENABLE_CLICKHOUSE_ALL})
else ()
    option (ENABLE_CLICKHOUSE_ODBC_BRIDGE "HTTP-server working like a proxy to ODBC driver" OFF)
endif ()

option (ENABLE_CLICKHOUSE_LIBRARY_BRIDGE "HTTP-server working like a proxy to Library dictionary source"
    ${ENABLE_CLICKHOUSE_ALL})

# https://presentations.clickhouse.tech/matemarketing_2020/
option (ENABLE_CLICKHOUSE_GIT_IMPORT "A tool to analyze Git repositories"
    ${ENABLE_CLICKHOUSE_ALL})


option (ENABLE_CLICKHOUSE_KEEPER "ClickHouse alternative to ZooKeeper" ${ENABLE_CLICKHOUSE_ALL})

if (CLICKHOUSE_SPLIT_BINARY)
    option(ENABLE_CLICKHOUSE_INSTALL "Install ClickHouse without .deb/.rpm/.tgz packages (having the binary only)" OFF)
else ()
    option(ENABLE_CLICKHOUSE_INSTALL "Install ClickHouse without .deb/.rpm/.tgz packages (having the binary only)"
        ${ENABLE_CLICKHOUSE_ALL})
endif ()

message(STATUS "ClickHouse modes:")

if (NOT ENABLE_CLICKHOUSE_SERVER)
    message(WARNING "ClickHouse server mode is not going to be built.")
else()
    message(STATUS "Server mode: ON")
endif()

if (NOT ENABLE_CLICKHOUSE_CLIENT)
    message(WARNING "ClickHouse client mode is not going to be built. You won't be able to connect to the server and run tests")
else()
    message(STATUS "Client mode: ON")
endif()

if (ENABLE_CLICKHOUSE_LOCAL)
    message(STATUS "Local mode: ON")
else()
    message(STATUS "Local mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_BENCHMARK)
    message(STATUS "Benchmark mode: ON")
else()
    message(STATUS "Benchmark mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_EXTRACT_FROM_CONFIG)
    message(STATUS "Extract from config mode: ON")
else()
    message(STATUS "Extract from config mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_COMPRESSOR)
    message(STATUS "Compressor mode: ON")
else()
    message(STATUS "Compressor mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_COPIER)
    message(STATUS "Copier mode: ON")
else()
    message(STATUS "Copier mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_FORMAT)
    message(STATUS "Format mode: ON")
else()
    message(STATUS "Format mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_OBFUSCATOR)
    message(STATUS "Obfuscator mode: ON")
else()
    message(STATUS "Obfuscator mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_ODBC_BRIDGE)
    message(STATUS "ODBC bridge mode: ON")
else()
    message(STATUS "ODBC bridge mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_LIBRARY_BRIDGE)
    message(STATUS "Library bridge mode: ON")
else()
    message(STATUS "Library bridge mode: OFF")
endif()

if (ENABLE_CLICKHOUSE_INSTALL)
    message(STATUS "ClickHouse install: ON")
else()
    message(STATUS "ClickHouse install: OFF")
endif()

if (ENABLE_CLICKHOUSE_GIT_IMPORT)
    message(STATUS "ClickHouse git-import: ON")
else()
    message(STATUS "ClickHouse git-import: OFF")
endif()

if (ENABLE_CLICKHOUSE_KEEPER)
    message(STATUS "ClickHouse keeper mode: ON")
else()
    message(STATUS "ClickHouse keeper mode: OFF")
endif()

if(NOT (MAKE_STATIC_LIBRARIES OR SPLIT_SHARED_LIBRARIES))
    set(CLICKHOUSE_ONE_SHARED ON)
endif()

configure_file (config_tools.h.in ${ConfigIncludePath}/config_tools.h)

macro(clickhouse_target_link_split_lib target name)
    if(NOT CLICKHOUSE_ONE_SHARED)
        target_link_libraries(${target} PRIVATE clickhouse-${name}-lib)
    else()
        target_link_libraries(${target} PRIVATE clickhouse-lib)
    endif()
endmacro()

macro(clickhouse_program_link_split_binary name)
    clickhouse_target_link_split_lib(clickhouse-${name} ${name})
endmacro()

macro(clickhouse_program_add_library name)
    string(TOUPPER ${name} name_uc)
    string(REPLACE "-" "_" name_uc ${name_uc})

    # Some dark magic
    set(CLICKHOUSE_${name_uc}_SOURCES ${CLICKHOUSE_${name_uc}_SOURCES} PARENT_SCOPE)
    set(CLICKHOUSE_${name_uc}_LINK ${CLICKHOUSE_${name_uc}_LINK} PARENT_SCOPE)
    set(CLICKHOUSE_${name_uc}_INCLUDE ${CLICKHOUSE_${name_uc}_INCLUDE} PARENT_SCOPE)

    if(NOT CLICKHOUSE_ONE_SHARED)
        add_library(clickhouse-${name}-lib ${CLICKHOUSE_${name_uc}_SOURCES})

        set(_link ${CLICKHOUSE_${name_uc}_LINK}) # can't use ${} in if()
        if(_link)
            target_link_libraries(clickhouse-${name}-lib ${CLICKHOUSE_${name_uc}_LINK})
        endif()

        set(_include ${CLICKHOUSE_${name_uc}_INCLUDE}) # can't use ${} in if()
        if (_include)
            target_include_directories(clickhouse-${name}-lib ${CLICKHOUSE_${name_uc}_INCLUDE})
        endif()
    endif()
endmacro()

macro(clickhouse_program_add_executable name)
    if(CLICKHOUSE_SPLIT_BINARY)
        add_executable(clickhouse-${name} clickhouse-${name}.cpp)
        clickhouse_program_link_split_binary(${name})
        install(TARGETS clickhouse-${name} ${CLICKHOUSE_ALL_TARGETS} RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
    endif()
endmacro()

macro(clickhouse_program_add name)
    clickhouse_program_add_library(${name})
    clickhouse_program_add_executable(${name})
endmacro()

# Embed default config files as a resource into the binary.
# This is needed for two purposes:
# 1. Allow to run the binary without download of any other files.
# 2. Allow to implement "sudo clickhouse install" tool.
#
# Arguments: target (server, client, keeper, etc.) and list of files
#
# Also dependency on TARGET_FILE is required, look at examples in programs/server and programs/keeper
macro(clickhouse_embed_binaries)
    # TODO We actually need this on Mac, FreeBSD.
    if (OS_LINUX)

        set(arguments_list "${ARGN}")
        list(GET arguments_list 0 target)

        # for some reason cmake iterates loop including <stop>
        math(EXPR arguments_count "${ARGC}-1")

        foreach(RESOURCE_POS RANGE 1 "${arguments_count}")
            list(GET arguments_list "${RESOURCE_POS}" RESOURCE_FILE)
            set(RESOURCE_OBJ ${RESOURCE_FILE}.o)
            set(RESOURCE_OBJS ${RESOURCE_OBJS} ${RESOURCE_OBJ})

            # https://stackoverflow.com/questions/14776463/compile-and-add-an-object-file-from-a-binary-with-cmake
            # PPC64LE fails to do this with objcopy, use ld or lld instead
            if (ARCH_PPC64LE)
                add_custom_command(OUTPUT ${RESOURCE_OBJ}
                    COMMAND cd ${CMAKE_CURRENT_SOURCE_DIR} && ${CMAKE_LINKER} -m elf64lppc -r -b binary -o "${CMAKE_CURRENT_BINARY_DIR}/${RESOURCE_OBJ}" ${RESOURCE_FILE})
            else()
                add_custom_command(OUTPUT ${RESOURCE_OBJ}
                    COMMAND cd ${CMAKE_CURRENT_SOURCE_DIR} && ${OBJCOPY_PATH} -I binary ${OBJCOPY_ARCH_OPTIONS} ${RESOURCE_FILE} "${CMAKE_CURRENT_BINARY_DIR}/${RESOURCE_OBJ}"
                    COMMAND ${OBJCOPY_PATH} --rename-section .data=.rodata,alloc,load,readonly,data,contents
                        "${CMAKE_CURRENT_BINARY_DIR}/${RESOURCE_OBJ}" "${CMAKE_CURRENT_BINARY_DIR}/${RESOURCE_OBJ}")
            endif()
            set_source_files_properties(${RESOURCE_OBJ} PROPERTIES EXTERNAL_OBJECT true GENERATED true)
        endforeach()

        add_library(clickhouse_${target}_configs STATIC ${RESOURCE_OBJS})
        set_target_properties(clickhouse_${target}_configs PROPERTIES LINKER_LANGUAGE C)

        # whole-archive prevents symbols from being discarded for unknown reason
        # CMake can shuffle each of target_link_libraries arguments with other
        # libraries in linker command. To avoid this we hardcode whole-archive
        # library into single string.
        add_dependencies(clickhouse-${target}-lib clickhouse_${target}_configs)
    endif ()
endmacro()


add_subdirectory (server)
add_subdirectory (client)
add_subdirectory (local)
add_subdirectory (benchmark)
add_subdirectory (extract-from-config)
add_subdirectory (compressor)
add_subdirectory (copier)
add_subdirectory (format)
add_subdirectory (obfuscator)
add_subdirectory (install)
add_subdirectory (git-import)
add_subdirectory (bash-completion)
add_subdirectory (keeper)

if (ENABLE_CLICKHOUSE_ODBC_BRIDGE)
    add_subdirectory (odbc-bridge)
endif ()

if (ENABLE_CLICKHOUSE_LIBRARY_BRIDGE)
    add_subdirectory (library-bridge)
endif ()

if (CLICKHOUSE_ONE_SHARED)
    add_library(clickhouse-lib SHARED ${CLICKHOUSE_SERVER_SOURCES} ${CLICKHOUSE_CLIENT_SOURCES} ${CLICKHOUSE_LOCAL_SOURCES} ${CLICKHOUSE_BENCHMARK_SOURCES} ${CLICKHOUSE_COPIER_SOURCES} ${CLICKHOUSE_EXTRACT_FROM_CONFIG_SOURCES} ${CLICKHOUSE_COMPRESSOR_SOURCES} ${CLICKHOUSE_FORMAT_SOURCES} ${CLICKHOUSE_OBFUSCATOR_SOURCES} ${CLICKHOUSE_GIT_IMPORT_SOURCES} ${CLICKHOUSE_ODBC_BRIDGE_SOURCES} ${CLICKHOUSE_KEEPER_SOURCES})
    target_link_libraries(clickhouse-lib ${CLICKHOUSE_SERVER_LINK} ${CLICKHOUSE_CLIENT_LINK} ${CLICKHOUSE_LOCAL_LINK} ${CLICKHOUSE_BENCHMARK_LINK} ${CLICKHOUSE_COPIER_LINK} ${CLICKHOUSE_EXTRACT_FROM_CONFIG_LINK} ${CLICKHOUSE_COMPRESSOR_LINK} ${CLICKHOUSE_FORMAT_LINK} ${CLICKHOUSE_OBFUSCATOR_LINK} ${CLICKHOUSE_GIT_IMPORT_LINK} ${CLICKHOUSE_ODBC_BRIDGE_LINK} ${CLICKHOUSE_KEEPER_LINK})
    target_include_directories(clickhouse-lib ${CLICKHOUSE_SERVER_INCLUDE} ${CLICKHOUSE_CLIENT_INCLUDE} ${CLICKHOUSE_LOCAL_INCLUDE} ${CLICKHOUSE_BENCHMARK_INCLUDE} ${CLICKHOUSE_COPIER_INCLUDE} ${CLICKHOUSE_EXTRACT_FROM_CONFIG_INCLUDE} ${CLICKHOUSE_COMPRESSOR_INCLUDE} ${CLICKHOUSE_FORMAT_INCLUDE} ${CLICKHOUSE_OBFUSCATOR_INCLUDE} ${CLICKHOUSE_GIT_IMPORT_INCLUDE} ${CLICKHOUSE_ODBC_BRIDGE_INCLUDE} ${CLICKHOUSE_KEEPER_INCLUDE})
    set_target_properties(clickhouse-lib PROPERTIES SOVERSION ${VERSION_MAJOR}.${VERSION_MINOR} VERSION ${VERSION_SO} OUTPUT_NAME clickhouse DEBUG_POSTFIX "")
    install (TARGETS clickhouse-lib LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT clickhouse)
endif()

if (CLICKHOUSE_SPLIT_BINARY)
    set (CLICKHOUSE_ALL_TARGETS clickhouse-server clickhouse-client clickhouse-local clickhouse-benchmark clickhouse-extract-from-config clickhouse-compressor clickhouse-format clickhouse-obfuscator clickhouse-git-import clickhouse-copier clickhouse-keeper)

    if (ENABLE_CLICKHOUSE_ODBC_BRIDGE)
        list (APPEND CLICKHOUSE_ALL_TARGETS clickhouse-odbc-bridge)
    endif ()

    if (ENABLE_CLICKHOUSE_LIBRARY_BRIDGE)
        list (APPEND CLICKHOUSE_ALL_TARGETS clickhouse-library-bridge)
    endif ()

    set_target_properties(${CLICKHOUSE_ALL_TARGETS} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ..)

    add_custom_target (clickhouse-bundle ALL DEPENDS ${CLICKHOUSE_ALL_TARGETS})
    add_custom_target (clickhouse ALL DEPENDS clickhouse-bundle)

    install(PROGRAMS clickhouse-split-helper DESTINATION ${CMAKE_INSTALL_BINDIR} RENAME clickhouse COMPONENT clickhouse)
else ()
    add_executable (clickhouse main.cpp)

    # A library that prevent usage of several functions from libc.
    if (ARCH_AMD64 AND OS_LINUX AND NOT OS_ANDROID)
        set (HARMFUL_LIB harmful)
    endif ()

    target_link_libraries (clickhouse PRIVATE clickhouse_common_io string_utils ${HARMFUL_LIB})
    target_include_directories (clickhouse PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

    if (ENABLE_CLICKHOUSE_SERVER)
        clickhouse_target_link_split_lib(clickhouse server)
    endif ()
    if (ENABLE_CLICKHOUSE_CLIENT)
        clickhouse_target_link_split_lib(clickhouse client)
    endif ()
    if (ENABLE_CLICKHOUSE_LOCAL)
        clickhouse_target_link_split_lib(clickhouse local)
    endif ()
    if (ENABLE_CLICKHOUSE_BENCHMARK)
        clickhouse_target_link_split_lib(clickhouse benchmark)
    endif ()
    if (ENABLE_CLICKHOUSE_COPIER)
        clickhouse_target_link_split_lib(clickhouse copier)
    endif ()
    if (ENABLE_CLICKHOUSE_EXTRACT_FROM_CONFIG)
        clickhouse_target_link_split_lib(clickhouse extract-from-config)
    endif ()
    if (ENABLE_CLICKHOUSE_COMPRESSOR)
        clickhouse_target_link_split_lib(clickhouse compressor)
    endif ()
    if (ENABLE_CLICKHOUSE_FORMAT)
        clickhouse_target_link_split_lib(clickhouse format)
    endif ()
    if (ENABLE_CLICKHOUSE_OBFUSCATOR)
        clickhouse_target_link_split_lib(clickhouse obfuscator)
    endif ()
    if (ENABLE_CLICKHOUSE_GIT_IMPORT)
        clickhouse_target_link_split_lib(clickhouse git-import)
    endif ()
    if (ENABLE_CLICKHOUSE_KEEPER)
        clickhouse_target_link_split_lib(clickhouse keeper)
    endif()
    if (ENABLE_CLICKHOUSE_INSTALL)
        clickhouse_target_link_split_lib(clickhouse install)
    endif ()

    set (CLICKHOUSE_BUNDLE)
    if (ENABLE_CLICKHOUSE_SERVER)
        add_custom_target (clickhouse-server ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-server DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-server" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-server)
    endif ()
    if (ENABLE_CLICKHOUSE_CLIENT)
        add_custom_target (clickhouse-client ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-client DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-client" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-client)
    endif ()
    if (ENABLE_CLICKHOUSE_LOCAL)
        add_custom_target (clickhouse-local ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-local DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-local" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-local)
    endif ()
    if (ENABLE_CLICKHOUSE_BENCHMARK)
        add_custom_target (clickhouse-benchmark ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-benchmark DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-benchmark" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-benchmark)
    endif ()
    if (ENABLE_CLICKHOUSE_COPIER)
        add_custom_target (clickhouse-copier ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-copier DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-copier" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-copier)
    endif ()
    if (ENABLE_CLICKHOUSE_EXTRACT_FROM_CONFIG)
        add_custom_target (clickhouse-extract-from-config ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-extract-from-config DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-extract-from-config" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-extract-from-config)
    endif ()
    if (ENABLE_CLICKHOUSE_COMPRESSOR)
        add_custom_target (clickhouse-compressor ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-compressor DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-compressor" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-compressor)
    endif ()
    if (ENABLE_CLICKHOUSE_FORMAT)
        add_custom_target (clickhouse-format ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-format DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-format" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-format)
    endif ()
    if (ENABLE_CLICKHOUSE_OBFUSCATOR)
        add_custom_target (clickhouse-obfuscator ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-obfuscator DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-obfuscator" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-obfuscator)
    endif ()
    if (ENABLE_CLICKHOUSE_GIT_IMPORT)
        add_custom_target (clickhouse-git-import ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-git-import DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-git-import" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-git-import)
    endif ()
    if (ENABLE_CLICKHOUSE_KEEPER)
        add_custom_target (clickhouse-keeper ALL COMMAND ${CMAKE_COMMAND} -E create_symlink clickhouse clickhouse-keeper DEPENDS clickhouse)
        install (FILES "${CMAKE_CURRENT_BINARY_DIR}/clickhouse-keeper" DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)
        list(APPEND CLICKHOUSE_BUNDLE clickhouse-keeper)
    endif ()

    install (TARGETS clickhouse RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT clickhouse)

    add_custom_target (clickhouse-bundle ALL DEPENDS ${CLICKHOUSE_BUNDLE})

    if (USE_GDB_ADD_INDEX)
        add_custom_command(TARGET clickhouse POST_BUILD COMMAND ${GDB_ADD_INDEX_EXE} clickhouse COMMENT "Adding .gdb-index to clickhouse" VERBATIM)
    endif()

    if (USE_BINARY_HASH)
        add_custom_command(TARGET clickhouse POST_BUILD COMMAND ./clickhouse hash-binary > hash && ${OBJCOPY_PATH} --add-section .note.ClickHouse.hash=hash clickhouse COMMENT "Adding .note.ClickHouse.hash to clickhouse" VERBATIM)
    endif()
endif ()

if (ENABLE_TESTS AND USE_GTEST)
    set (CLICKHOUSE_UNIT_TESTS_TARGETS unit_tests_dbms)
    add_custom_target (clickhouse-tests ALL DEPENDS ${CLICKHOUSE_UNIT_TESTS_TARGETS})
    add_dependencies(clickhouse-bundle clickhouse-tests)
endif()
