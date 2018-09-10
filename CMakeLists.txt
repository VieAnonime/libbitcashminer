# Copyright (c) 2017-2018 The Merit Foundation developers
# Copyright (c) 2018 The BitCash developers
# Distributed under the MIT software license, see the accompanying
# file COPYING or http://www.opensource.org/licenses/mit-license.php.

cmake_minimum_required(VERSION 2.8.11)
project(libbitcashminer LANGUAGES C CXX)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CUDA_STANDARD 14)

#setup boost
set(Boost_USE_STATIC_LIBS ON)
find_package(Boost COMPONENTS system program_options thread REQUIRED)
link_directories(${Boost_LIBRARY_DIRS})
include_directories(${Boost_INCLUDE_DIRS})
add_library(boost_system STATIC IMPORTED)
set_target_properties(boost_system PROPERTIES IMPORTED_LOCATION ${Boost_SYSTEM_LIBRARY_RELEASE})
add_library(boost_program_options STATIC IMPORTED)
set_target_properties(boost_program_options PROPERTIES IMPORTED_LOCATION ${Boost_PROGRAM_OPTIONS_LIBRARY_RELEASE})
add_library(boost_thread STATIC IMPORTED)
set_target_properties(boost_thread PROPERTIES IMPORTED_LOCATION ${Boost_THREAD_LIBRARY_RELEASE})

include_directories(include)

add_compile_options($<$<COMPILE_LANGUAGE:CXX>:-fPIC> $<$<COMPILE_LANGUAGE:C>:-fPIC>)

find_package(CUDA)
if(CUDA_FOUND)
    enable_language(CUDA)
    add_definitions(-DCUDA_ENABLED)
    set(CMAKE_CUDA_FLAGS "-Xcompiler \"-fPIC\"")

    link_directories(${CMAKE_CUDA_IMPLICIT_LINK_DIRECTORIES})
    include_directories(${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES})
    add_library(cuda STATIC IMPORTED)
    set_target_properties(cuda PROPERTIES IMPORTED_LOCATION ${CUDA_cudart_static_LIBRARY})
    set(COMBINE_LIBS 
        $<TARGET_FILE:bitcashminer>
        $<TARGET_FILE:boost_system>
        $<TARGET_FILE:boost_program_options>
        $<TARGET_FILE:boost_thread>
        $<TARGET_FILE:cuda>)
    set(FAT_OBJS *.cu.o *.o)

    add_library(bitcashminer STATIC
        src/public.cpp
        src/cuckoo/gpu/kernel.cu
        src/cuckoo/gpu/exceptions.h
        src/cuckoo/mean_cuckoo.cpp
        src/blake2/blake2b-ref.c
        src/stratum/stratum.cpp
        src/miner/miner.cpp
        src/util/util.cpp
        src/nvml/nvml.cpp)
else()
    set(COMBINE_LIBS 
        $<TARGET_FILE:bitcashminer>
        $<TARGET_FILE:boost_system>
        $<TARGET_FILE:boost_program_options>
        $<TARGET_FILE:boost_thread>)

    set(FAT_OBJS *.o)
    add_library(bitcashminer STATIC 
        src/public.cpp
        src/cuckoo/mean_cuckoo.cpp
        src/blake2/blake2b-ref.c
        src/stratum/stratum.cpp
        src/miner/miner.cpp
        src/util/util.cpp)
endif()

if(CMAKE_HOST_WIN32)
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
    set(FAT_LIB ${CMAKE_BINARY_DIR}/libfatbitcashminer.lib)
    add_custom_target(combined
        COMMAND lib.exe /OUT:${FAT_LIB} ${COMBINE_LIBS}
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
            DEPENDS bitcashminer boost_system boost_program_options boost_thread)
    add_library(fatbitcashminer STATIC IMPORTED GLOBAL)
    add_dependencies(fatbitcashminer combined)
    set_target_properties(fatbitcashminer PROPERTIES IMPORTED_LOCATION ${FAT_LIB})
else()
    set(FAT_LIB ${CMAKE_BINARY_DIR}/libfatbitcashminer.a)
    add_custom_target(combined
        COMMAND for f in ${COMBINE_LIBS}\; do ar -x \$\$f\; done 
        COMMAND ar -qcs ${FAT_LIB} ${FAT_OBJS}
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        DEPENDS bitcashminer boost_system boost_program_options boost_thread)
    add_library(fatbitcashminer STATIC IMPORTED GLOBAL)
    add_dependencies(fatbitcashminer combined)
    set_target_properties(fatbitcashminer PROPERTIES IMPORTED_LOCATION ${FAT_LIB})
endif()

add_executable(bitcash-minerd src/minerd.cpp)

if(CMAKE_HOST_WIN32)
	target_link_libraries(bitcash-minerd fatbitcashminer)
else()
	target_link_libraries(bitcash-minerd fatbitcashminer pthread rt dl)
endif()

install(TARGETS bitcash-minerd bitcashminer
            RUNTIME DESTINATION bin
            LIBRARY DESTINATION lib
            ARCHIVE DESTINATION lib)
install(FILES ${FAT_LIB} DESTINATION lib)

file(GLOB H_PUB include/bitcash/*.hpp)
file(GLOB H_BLAKE include/bitcash/blake/*.h)
file(GLOB H_CRYPTO include/bitcash/crypto/*.h)
file(GLOB H_CTPL include/bitcash/ctpl/*.h)
file(GLOB H_CUCKOO include/bitcash/cuckoo/*.h)
file(GLOB H_MINER include/bitcash/miner/*.hpp) 
file(GLOB H_PICO include/bitcash/PicoSHA2/*.h)
file(GLOB H_STRATUM include/bitcash/stratum/*.hpp)
file(GLOB H_UTIL include/bitcash/util/*.hpp)
file(GLOB H_NVML include/bitcash/nvml/*.h)
install(FILES ${H_PUB} DESTINATION include/bitcash)
install(FILES ${H_BLAKE} DESTINATION include/bitcash/blake)
install(FILES ${H_CRYPTO} DESTINATION include/bitcash/crypto)
install(FILES ${H_CTPL} DESTINATION include/bitcash/ctpl)
install(FILES ${H_CUCKOO} DESTINATION include/bitcash/cuckoo)
install(FILES ${H_MINER} DESTINATION include/bitcash/miner)
install(FILES ${H_PICO} DESTINATION include/bitcash/PicoSHA2)
install(FILES ${H_STRATUM} DESTINATION include/bitcash/stratum)
install(FILES ${H_UTIL} DESTINATION include/bitcash/util)
install(FILES ${H_NVML} DESTINATION include/bitcash/nvml)
include(CPack)

