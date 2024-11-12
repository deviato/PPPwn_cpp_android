# [ Deviato android fixes for PPPwn_cpp ]
# This file is intended for use with NDK builder on Linux machines
# If you want to compile this natively in Termux, just call the script with `cmake -B build -DTERMUX=1`
# otherwise you have to set the options below to reflect your building system and your desired target.
# I assumed a default download of NDK r25c in your ~/Android/Sdk/ndk (standard from android studio)
# because it's the last one that supports Android sdk19 (KitKat 4.4) builds.
# The default minimum target is set as Lollipop 5.0 (sdk 21) for arm64 processors.

# NDK Related, change to your needs
if(NOT TERMUX)
  # Change to your ndk installation folder
  #set(CMAKE_TOOLCHAIN_FILE "~/Android/Sdk/ndk/22.1.7171670/build/cmake/android.toolchain.cmake")
  set(CMAKE_TOOLCHAIN_FILE "~/Android/Sdk/ndk/25.2.9519653/build/cmake/android.toolchain.cmake")
  # ANDROID_ABI is the target architecture, can be one of the following: armeabi-v7a, arm64-v8a, x86, x86_64
  set(ANDROID_ABI "arm64-v8a")
  # PLATFORM is the minimum supported android api level for your target. Notice: if you want to build for android KitKat 4.4.x
  # (sdk 19 and 20) you can't build static (I'm trying to solve) because of `duplicate symbol` errors on linking.
  set(ANDROID_PLATFORM "21")
endif ()
##stop-editing

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
# Termux specific, irrelevant to others. Fixes the `ld cannot find -lpthread` error when building static binary
set(THREADS_PREFER_PTHREAD_FLAG ON)
# Common, static building of external libpcap
option(USE_SYSTEM_PCAP "Disable System PCAP for static build" OFF)
if(ANDROID_PLATFORM STREQUAL "19" OR ANDROID_PLATFORM STREQUAL "20")
  # Needs a fix for duplicate symbol error on sdk19 static building, leaving to shared for now
  set(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++ -fPIC -fPIE" CACHE STRING "" FORCE)
  # NW set(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++ -static -z muldefs" CACHE STRING "" FORCE)
else ()
  set(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++ -static -fPIC -fPIE" CACHE STRING "" FORCE)
endif ()

cmake_minimum_required(VERSION 3.12)
include(CMakeDependentOption)
set(APP_BUILD_OPTIONS)
set(APP_LINK_LIB)

# Zig makes cross-compiling easier and ignores glibc version issue
# target can be: x86_64-macos-none, x86_64-macos.11.0-none, x86_64-windows-gnu, mipsel-linux-musl, x86_64-linux-gnu.2.17, ...
# Using `zig targets | jq .libc` to get the list of targets
option(ZIG_TARGET "Setting a target and using `zig cc/c++` to compile" OFF)

# example: -DZIG_NIGHTLY="0.13.0-dev.46+3648d7df1"
# You can find the latest version by: `curl -sS https://ziglang.org/download/index.json | jq .master.version`
option(ZIG_NIGHTLY "Download nightly build instead of release version if zig is not in the system path" OFF)

# Is not recommended to use pcap++ provided by package manager, because we need immediate_mode
option(USE_SYSTEM_PCAPPLUSPLUS "Find pcapp++ from system path" OFF)
cmake_dependent_option(USE_SYSTEM_PCAP "Find pcap from system path" ON "NOT USE_SYSTEM_PCAPPLUSPLUS" ON)

option(BUILD_CLI "build command line tool" ON)
option(BUILD_TEST "build test" OFF)

if (ZIG_TARGET)
    include(${CMAKE_SOURCE_DIR}/cmake/zig.cmake)
    set(CMAKE_TOOLCHAIN_FILE "${CMAKE_SOURCE_DIR}/cmake/zig.toolchain.cmake")
    add_compile_options(--target=${ZIG_TARGET})
    message(STATUS "Custom compile options: ${ZIG_COMPILE_OPTION}")
    if (ZIG_COMPILE_OPTION)
        add_compile_options(${ZIG_COMPILE_OPTION})
        add_link_options(${ZIG_COMPILE_OPTION})
    endif ()
endif ()

if (CMAKE_BUILD_TYPE STREQUAL Debug)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address,undefined")
    list(APPEND APP_BUILD_OPTIONS -DDEBUG)
else ()
    # Remove prefix path in log for release build
    add_compile_options(-fmacro-prefix-map=${CMAKE_SOURCE_DIR}/=/)
endif ()

project(pppwn)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS ON)
if (MINGW)
    set(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++ -static" CACHE STRING "" FORCE)
    add_compile_definitions(_Post_invalid_=) # Fix incompatibility with old npcap
    list(APPEND APP_LINK_LIB winmm)
endif ()

if (USE_SYSTEM_PCAPPLUSPLUS)
    find_package(PcapPlusPlus REQUIRED)
    list(APPEND APP_LINK_LIB PcapPlusPlus::Pcap++)
else ()
    set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)
    set(BUILD_SHARED_LIBS OFF)
    set(PPPWN_PCAP_DIRECTION ON)
    if (MINGW)
        # Fix a pcap CMakeLists bug
        set(BUILD_SHARED_LIBS ON)
        # Fix zig cross-compiling issue
        set(CMAKE_USE_PTHREADS_INIT ON)
        set(PPPWN_PCAP_DIRECTION OFF)
    endif ()
    if (NOT USE_SYSTEM_PCAP)
        include(FetchContent)
        set(DISABLE_DBUS ON)
        set(DISABLE_RDMA ON)
        set(BUILD_WITH_LIBNL OFF)
        FetchContent_Declare(pcap GIT_REPOSITORY https://github.com/the-tcpdump-group/libpcap.git
                GIT_TAG libpcap-1.10.4)
        FetchContent_MakeAvailable(pcap)
        file(COPY ${pcap_SOURCE_DIR}/pcap DESTINATION ${CMAKE_BINARY_DIR}/vendor/include)
        include_directories("${CMAKE_BINARY_DIR}/vendor/include")
        set(PCAP_INCLUDE_DIR "${CMAKE_BINARY_DIR}/vendor/include/pcap")
        set(PCAP_LIBRARY pcap_static)
        # Add some variables to let PcapPlusPlus know the libpcap path
        add_library(PCAP::PCAP INTERFACE IMPORTED)
        target_link_libraries(PCAP::PCAP INTERFACE pcap_static)
        target_include_directories(PCAP::PCAP INTERFACE "${PCAP_INCLUDE_DIR}")
        target_compile_definitions(PCAP::PCAP INTERFACE "_BSD_SOURCE" "_DEFAULT_SOURCE")
        target_compile_options(pcap_static PRIVATE "-Wno-macro-redefined" "-Wno-implicit-function-declaration")
        set(PCAP_NEEDS_THREADS ON)
        set(HAVE_PCAP_IMMEDIATE_MODE ON)
        set(HAVE_PCAP_DIRECTION ON)
    endif ()
    set(BUILD_SHARED_LIBS OFF)
    set(PCAPPP_ENABLE_PCAP_SET_DIRECTION ${PPPWN_PCAP_DIRECTION})
    set(PCAPPP_ENABLE_PCAP_IMMEDIATE_MODE ON)
    include(FetchContent)
    FetchContent_Declare(PcapPlusPlus GIT_REPOSITORY https://github.com/seladb/PcapPlusPlus.git
            GIT_TAG v23.09)
    FetchContent_MakeAvailable(PcapPlusPlus)
    set_property(TARGET Packet++ PROPERTY COMPILE_WARNING_AS_ERROR OFF)
    set_property(TARGET Pcap++ PROPERTY COMPILE_WARNING_AS_ERROR OFF)
    set_property(TARGET Common++ PROPERTY COMPILE_WARNING_AS_ERROR OFF)
    list(APPEND APP_LINK_LIB Pcap++)
    set(ZIG_BE "mips" "mips64" "armeb" "powerpc64" "powerpc" "aarch64_be" "s390x")
    list(FIND ZIG_BE "${ZIG_TARGET_ARCH}" INDEX)
    if (${INDEX} GREATER -1)
        message(STATUS "Patching Layer.h to support big endian when cross-compiling")
        add_custom_target(PatchLayer
                WORKING_DIRECTORY ${PcapPlusPlus_SOURCE_DIR}
                COMMAND git checkout -- ${PcapPlusPlus_SOURCE_DIR}/Packet++/header/Layer.h
                COMMAND git apply --stat --apply ${CMAKE_SOURCE_DIR}/endian.patch
        )
        add_dependencies(Packet++ PatchLayer)
    endif ()
endif ()

add_library(${PROJECT_NAME}_static STATIC src/exploit.cpp src/packet.cpp)
set_target_properties(${PROJECT_NAME}_static PROPERTIES OUTPUT_NAME ${PROJECT_NAME})
target_link_libraries(${PROJECT_NAME}_static PUBLIC ${APP_LINK_LIB})
target_compile_options(${PROJECT_NAME}_static PUBLIC ${APP_BUILD_OPTIONS})
target_include_directories(${PROJECT_NAME}_static PUBLIC include)

# Remove -g flag for NDK cc
string(REPLACE "-g " "" CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
if (BUILD_CLI)
    include(FetchContent)
    FetchContent_Declare(clipp GIT_REPOSITORY https://github.com/muellan/clipp.git
            GIT_TAG v1.2.3)
    FetchContent_MakeAvailable(clipp)

    FetchContent_Declare(mongoose GIT_REPOSITORY https://github.com/cesanta/mongoose.git
            GIT_TAG 7.14)
    FetchContent_MakeAvailable(mongoose)

    add_library(mongoose STATIC ${mongoose_SOURCE_DIR}/mongoose.c)
    if(ANDROID_PLATFORM STREQUAL "19" OR ANDROID_PLATFORM STREQUAL "20")
      # Fix implicit declaration of function 'epoll_create1' on Android 19 & 20 by disabling epoll
      # The issue is solved in Android 5 (sdk 21) which introduces the missing epoll_create1 syscall in kernel.
      target_compile_options(mongoose PUBLIC -DMG_ENABLE_PACKED_FS=1 -DMG_ENABLE_EPOLL=0)
    else ()
      target_compile_options(mongoose PUBLIC -DMG_ENABLE_PACKED_FS=1)
    endif ()
    target_include_directories(mongoose PUBLIC ${mongoose_SOURCE_DIR})

    add_custom_command(
            OUTPUT ${CMAKE_BINARY_DIR}/static.c
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            COMMAND ${ZIG} cc ${CMAKE_C_FLAGS} -o ${CMAKE_BINARY_DIR}/pack ${mongoose_SOURCE_DIR}/test/pack.c
            COMMAND ${CMAKE_BINARY_DIR}/pack web/index.html web/IBMPlexMono-Regular.ttf > ${CMAKE_BINARY_DIR}/static.c
            DEPENDS web/index.html
    )

    add_executable(${PROJECT_NAME} src/main.cpp src/web.cpp ${mongoose_SOURCE_DIR}/mongoose.c ${CMAKE_BINARY_DIR}/static.c)
    target_include_directories(${PROJECT_NAME} PRIVATE ${clipp_SOURCE_DIR}/include ${mongoose_SOURCE_DIR})
    target_link_libraries(${PROJECT_NAME} PRIVATE ${PROJECT_NAME}_static mongoose)
endif ()

if (BUILD_TEST)
    add_subdirectory(tests)
endif ()
