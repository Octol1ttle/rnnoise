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

set(SOURCES
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

add_compile_options(-O3 -s -fstack-protector-strong)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)

add_library(rnnoise STATIC ${SOURCES})
target_include_directories(rnnoise PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
