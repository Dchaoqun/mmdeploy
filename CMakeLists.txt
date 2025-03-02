# Copyright (c) OpenMMLab. All rights reserved.
if (NOT DEFINED CMAKE_INSTALL_PREFIX)
    set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "MMDeploy's installation directory")
endif ()
message(STATUS "CMAKE_INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX}")

cmake_minimum_required(VERSION 3.14)
project(MMDeploy VERSION 0.1.0)

set(CMAKE_CXX_STANDARD 17)


set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
if (MSVC)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
else ()
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
endif ()
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# options
option(BUILD_SHARED_LIBS "build shared lib" ON)
option(MMDEPLOY_BUILD_MODEL_OPTIMIZER "build MMDeploy's model optimizer" ON)
option(MMDEPLOY_BUILD_SDK "build MMDeploy's SDK" OFF)
option(MMDEPLOY_ZIP_MODEL "support sdk model in zip format" OFF)
option(MMDEPLOY_BUILD_TEST "build MMDeploy's csrc's unittest" OFF)
set(MMDEPLOY_TARGET_DEVICES
        "cpu" CACHE STRING "MMDeploy's target devices")
set(MMDEPLOY_TARGET_BACKENDS "" CACHE STRING "MMDeploy's target inference engines")
set(MMDEPLOY_CODEBASES "all"  CACHE STRING "select OpenMMLab's codebases")

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "choose 'Release' as default build type" FORCE)
endif ()

# when CUDA devices are enabled, the environment variable ASAN_OPTIONS=protect_shadow_gap=0
# must be set at runtime
if (MMDEPLOY_ASAN_ENABLE)
    add_compile_options($<$<COMPILE_LANGUAGE:CXX>:-fsanitize=address>)
    add_link_options(-fsanitize=address)
endif ()

# notice that ubsan has linker issues for ubuntu < 18.04, see
# https://stackoverflow.com/questions/50024731/ld-unrecognized-option-push-state-no-as-needed
if (MMDEPLOY_UBSAN_ENABLE)
    add_compile_options($<$<COMPILE_LANGUAGE:CXX>:-fsanitize=undefined>)
    add_link_options(-fsanitize=undefined)
endif ()

if (MSVC)
    add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/diagnostics:classic>)
    add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/Zc:preprocessor>) # /experimental:preprocessor on VS2017
    add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/wd4251>)
else ()
    add_compile_options($<$<COMPILE_LANGUAGE:CXX>:-fvisibility=hidden>)
endif ()

include(${CMAKE_SOURCE_DIR}/cmake/MMDeploy.cmake)

add_library(MMDeployStaticModules INTERFACE)
add_library(MMDeployDynamicModules INTERFACE)
add_library(MMDeployLibs INTERFACE)

add_subdirectory(csrc)

if (MMDEPLOY_BUILD_SDK)
    install(TARGETS MMDeployStaticModules
            MMDeployDynamicModules
            MMDeployLibs
            EXPORT MMDeployTargets)

    if (MMDEPLOY_BUILD_TEST)
        add_subdirectory(tests/test_csrc)
    endif ()

    if (MMDEPLOY_BUILD_SDK_PYTHON_API)
        add_subdirectory(csrc/apis/python)
    endif ()

    # export MMDeploy package
    install(EXPORT MMDeployTargets
            FILE MMDeployTargets.cmake
            DESTINATION lib/cmake/MMDeploy)

    # append backend deps
    mmdeploy_add_deps(trt BACKENDS ${MMDEPLOY_TARGET_BACKENDS} DEPS TENSORRT CUDNN)
    mmdeploy_add_deps(ort BACKENDS ${MMDEPLOY_TARGET_BACKENDS} DEPS ONNXRUNTIME)
    mmdeploy_add_deps(ncnn BACKENDS ${MMDEPLOY_TARGET_BACKENDS} DEPS ncnn)
    mmdeploy_add_deps(openvino BACKENDS ${MMDEPLOY_TARGET_BACKENDS} DEPS InferenceEngine)
    mmdeploy_add_deps(pplnn BACKENDS ${MMDEPLOY_TARGET_BACKENDS} DEPS pplnn)

    include(CMakePackageConfigHelpers)
    # generate the config file that is includes the exports
    configure_package_config_file(${CMAKE_SOURCE_DIR}/cmake/MMDeployConfig.cmake.in
            "${CMAKE_CURRENT_BINARY_DIR}/MMDeployConfig.cmake"
            INSTALL_DESTINATION "lib/cmake"
            NO_SET_AND_CHECK_MACRO
            NO_CHECK_REQUIRED_COMPONENTS_MACRO
            )

    write_basic_package_version_file(
            "${CMAKE_CURRENT_BINARY_DIR}/MMDeployConfigVersion.cmake"
            VERSION "${MMDeploy_VERSION_MAJOR}.${MMDeploy_VERSION_MINOR}"
            COMPATIBILITY AnyNewerVersion
    )

    install(FILES
            ${CMAKE_CURRENT_BINARY_DIR}/MMDeployConfig.cmake
            ${CMAKE_CURRENT_BINARY_DIR}/MMDeployConfigVersion.cmake
            ${CMAKE_CURRENT_SOURCE_DIR}/cmake/MMDeploy.cmake
            ${CMAKE_CURRENT_SOURCE_DIR}/cmake/loader.cpp.in
            DESTINATION lib/cmake/MMDeploy
            )
    install(DIRECTORY
            ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules
            DESTINATION lib/cmake/MMDeploy
            )

    install(DIRECTORY ${CMAKE_SOURCE_DIR}/demo/csrc/ DESTINATION example)
endif ()
