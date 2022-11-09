cmake_minimum_required(VERSION 3.11...3.18)

include(FetchContent)

find_package(CUDAToolkit)

project("eznve" VERSION 0.1 LANGUAGES CXX C)


# ffmpeg's mirror of nvenc headers
FetchContent_Declare(
  nv-codec-headers
  GIT_REPOSITORY https://github.com/FFmpeg/nv-codec-headers.git
)
FetchContent_MakeAvailable(nv-codec-headers)

FetchContent_Declare(
    dylib
    GIT_REPOSITORY "https://github.com/martin-olivier/dylib"
    GIT_TAG        "v2.1.0"
)
FetchContent_MakeAvailable(dylib)

add_library(
    eznve STATIC
    src/eznve.cpp
    include/eznve.hpp
)
target_compile_features(eznve PUBLIC cxx_std_23)
target_include_directories(eznve PUBLIC include)
target_include_directories(eznve PRIVATE ${nv-codec-headers_SOURCE_DIR}/include)

target_link_libraries(eznve PRIVATE dylib CUDA::cuda_driver)