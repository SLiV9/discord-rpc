include_directories(${PROJECT_SOURCE_DIR}/include)

option(ENABLE_IO_THREAD "Start up a separate I/O thread, otherwise I'd need to call an update function" ON)
option(USE_STATIC_CRT "Use /MT[d] for dynamic library" OFF)
option(WARNINGS_AS_ERRORS "When enabled, compiles with `-Werror` (on *nix platforms)." OFF)

set(CMAKE_CXX_STANDARD 14)

set(BASE_RPC_SRC
    ${PROJECT_SOURCE_DIR}/include/discord_rpc.h
    discord_rpc.cpp
    ${PROJECT_SOURCE_DIR}/include/discord_register.h
    rpc_connection.h
    rpc_connection.cpp
    serialization.h
    serialization.cpp
    connection.h
    backoff.h
    msg_queue.h
)

if (WIN32)
    if (CMAKE_SIZEOF_VOID_P EQUAL 8)
        MESSAGE("64 bits compiler detected")
        set(LIBRARY_EXTENSION "x64")
    else(CMAKE_SIZEOF_VOID_P EQUAL 8)
        MESSAGE("32 bits compiler detected")
        set(LIBRARY_EXTENSION "x86")
    endif(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(LIBRARY_NAME "discord-rpc.${LIBRARY_EXTENSION}")
else(WIN32)
    set(LIBRARY_NAME "discord-rpc")
endif(WIN32)
MESSAGE("LIBRARY_NAME is ${LIBRARY_NAME}")


if (${BUILD_SHARED_LIBS})
    if(WIN32)
        set(BASE_RPC_SRC ${BASE_RPC_SRC} dllmain.cpp)
    endif(WIN32)
endif(${BUILD_SHARED_LIBS})

if(WIN32)
    add_definitions(-DDISCORD_WINDOWS)
    set(BASE_RPC_SRC ${BASE_RPC_SRC} connection_win.cpp discord_register_win.cpp)
    add_library(${LIBRARY_NAME} ${BASE_RPC_SRC})
    if (MSVC)
        if(USE_STATIC_CRT)
            foreach(CompilerFlag
                    CMAKE_CXX_FLAGS
                    CMAKE_CXX_FLAGS_DEBUG
                    CMAKE_CXX_FLAGS_RELEASE
                    CMAKE_C_FLAGS
                    CMAKE_C_FLAGS_DEBUG
                    CMAKE_C_FLAGS_RELEASE)
                string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
            endforeach()
        endif(USE_STATIC_CRT)
        target_compile_options(${LIBRARY_NAME} PRIVATE /EHsc
            /Wall
            /wd4100 # unreferenced formal parameter
            /wd4514 # unreferenced inline
            /wd4625 # copy constructor deleted
            /wd5026 # move constructor deleted
            /wd4626 # move assignment operator deleted
            /wd4668 # not defined preprocessor macro
            /wd4710 # function not inlined
            /wd4711 # function was inlined
            /wd4820 # structure padding
            /wd4946 # reinterpret_cast used between related classes
            /wd5027 # move assignment operator was implicitly defined as deleted
        )
    endif(MSVC)
    target_link_libraries(${LIBRARY_NAME} PRIVATE psapi advapi32)
endif(WIN32)

if(UNIX)
    set(BASE_RPC_SRC ${BASE_RPC_SRC} connection_unix.cpp)

    if (APPLE)
        add_definitions(-DDISCORD_OSX)
        set(BASE_RPC_SRC ${BASE_RPC_SRC} discord_register_osx.m)
    else (APPLE)
        add_definitions(-DDISCORD_LINUX)
        set(BASE_RPC_SRC ${BASE_RPC_SRC} discord_register_linux.cpp)
    endif(APPLE)

    add_library(${LIBRARY_NAME} ${BASE_RPC_SRC})
    target_link_libraries(${LIBRARY_NAME} PUBLIC pthread)
    target_compile_options(${LIBRARY_NAME} PRIVATE
        -g
        -Wall
        -Wextra
        -Wpedantic
    )

    if (${WARNINGS_AS_ERRORS})
      target_compile_options(${LIBRARY_NAME} PRIVATE -Werror)
    endif (${WARNINGS_AS_ERRORS})

    target_compile_options(discord-rpc PRIVATE
        -Wno-unknown-pragmas # pragma push thing doesn't work on clang
        -Wno-old-style-cast # it's fine
        -Wno-c++98-compat # that was almost 2 decades ago
        -Wno-c++98-compat-pedantic
        -Wno-missing-noreturn
        -Wno-padded # structure padding
        -Wno-covered-switch-default
        -Wno-exit-time-destructors # not sure about these
        -Wno-global-constructors
    )

    if (${BUILD_SHARED_LIBS})
        target_compile_options(${LIBRARY_NAME} PRIVATE -fPIC)
    endif (${BUILD_SHARED_LIBS})

    if (APPLE)
        target_link_libraries(${LIBRARY_NAME} PRIVATE "-framework AppKit")
    endif (APPLE)
endif(UNIX)

target_include_directories(${LIBRARY_NAME} PRIVATE ${RAPIDJSON}/include)

if (NOT ${ENABLE_IO_THREAD})
    target_compile_definitions(${LIBRARY_NAME} PUBLIC -DDISCORD_DISABLE_IO_THREAD)
endif (NOT ${ENABLE_IO_THREAD})

if (${BUILD_SHARED_LIBS})
    target_compile_definitions(${LIBRARY_NAME} PUBLIC -DDISCORD_DYNAMIC_LIB)
    target_compile_definitions(${LIBRARY_NAME} PRIVATE -DDISCORD_BUILDING_SDK)
endif(${BUILD_SHARED_LIBS})

if (CLANG_FORMAT_CMD)
    add_dependencies(${LIBRARY_NAME} clangformat)
endif(CLANG_FORMAT_CMD)

# install

install(
    TARGETS ${LIBRARY_NAME}
    EXPORT "${LIBRARY_NAME}"
    RUNTIME
        DESTINATION "${CMAKE_INSTALL_BINDIR}"
    LIBRARY
        DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE
        DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    INCLUDES
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

install(
    FILES
        "../include/discord_rpc.h"
		"../include/discord_register.h"
    DESTINATION "include"
)
