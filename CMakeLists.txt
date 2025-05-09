cmake_minimum_required(VERSION 3.31)
project(rnnoise LANGUAGES C)

set(RNNOISE_MODEL_URL "https://media.xiph.org/rnnoise/models/rnnoise_data-")

file(READ "./model_version" rnnoise_model_hash)
string(STRIP ${rnnoise_model_hash} rnnoise_model_hash)
string(CONCAT rnnoise_model_url "${RNNOISE_MODEL_URL}" "${rnnoise_model_hash}" ".tar.gz")

include(FetchContent)
FetchContent_Declare(
    rnnoise_model
    URL "${rnnoise_model_url}"
    URL_HASH SHA256=${rnnoise_model_hash}
    DOWNLOAD_EXTRACT_TIMESTAMP 1
)
FetchContent_MakeAvailable(rnnoise_model)

if (NOT EXISTS "${CMAKE_CURRENT_LIST_DIR}/src/rnnoise_data.c")
    file(COPY ${rnnoise_model_SOURCE_DIR}/src/rnnoise_data.c DESTINATION "${CMAKE_CURRENT_LIST_DIR}/src")
    file(COPY ${rnnoise_model_SOURCE_DIR}/src/rnnoise_data.h DESTINATION "${CMAKE_CURRENT_LIST_DIR}/src")
endif ()

add_library(rnnoise STATIC)

add_compile_options(-O3 -s -fstack-protector-strong)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)

target_sources(rnnoise PRIVATE
    src/denoise.c
    src/rnn.c
    src/pitch.c
    src/kiss_fft.c
    src/celt_lpc.c
    src/nnet.c
    src/nnet_default.c
    src/parse_lpcnet_weights.c
    src/rnnoise_data.c
    src/rnnoise_tables.c
)

option(RNNOISE_X86_RTCD OFF)

if (RNNOISE_X86_RTCD)
    add_compile_definitions(RNN_ENABLE_X86_RTCD CPU_INFO_BY_ASM)

    target_sources(rnnoise PRIVATE
        src/x86/x86_dnn_map.c
        src/x86/x86cpu.c
    )

    include(CheckSourceCompiles)
    set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
    set(CMAKE_REQUIRED_INCLUDES src)

    check_source_compiles(C
"#ifndef __SSE4_1__
#error SSE4.1 unavailable
#endif"
        SSE4_1_AVAILABLE)
    if (SSE4_1_AVAILABLE)
        message("Will use SSE4.1")
        target_sources(rnnoise PRIVATE src/x86/nnet_sse4_1.c)
    endif ()

    check_source_compiles(C
"#ifndef __AVX2__
#error AVX2 unavailable
#endif"
        AVX2_AVAILABLE)
    if (AVX2_AVAILABLE)
        message("Will use AVX2")
        target_sources(rnnoise PRIVATE src/x86/nnet_avx2.c)
    endif ()
endif ()

target_include_directories(rnnoise PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
