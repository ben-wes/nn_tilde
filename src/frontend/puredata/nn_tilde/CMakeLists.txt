cmake_minimum_required(VERSION 3.10)
project(nn_tilde_pd)
find_package(Torch REQUIRED)

file(GLOB SRC *.cpp)

add_library(nn SHARED ${SRC})

# Get version from git for all platforms
execute_process(
    COMMAND git describe --tags
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
    OUTPUT_VARIABLE VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
if(VERSION)
    message(STATUS "Building version: ${VERSION}")
    add_definitions(-DVERSION="${VERSION}")
endif()

# COPY HELP FILES
add_custom_command(TARGET nn POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${CMAKE_SOURCE_DIR}/frontend/puredata/nn_tilde/nn~-help.pd"
        "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CFG_INTDIR}/nn~-help.pd"
    COMMENT "Copy Help File"
)

if (APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -undefined dynamic_lookup")

    add_custom_command(
        TARGET nn
        POST_BUILD
        COMMAND cp "${TORCH_INSTALL_PREFIX}/lib/*.dylib" "${CMAKE_CURRENT_BINARY_DIR}/"
        COMMENT "Copy Torch Libraries"
    )

    set_target_properties(nn PROPERTIES
        PREFIX ""
        SUFFIX "~.pd_darwin"
        BUILD_WITH_INSTALL_RPATH FALSE
        LINK_FLAGS "-Wl,-rpath,@loader_path/"
    )

    if (CMAKE_OSX_ARCHITECTURES STREQUAL "arm64")
        add_custom_command(
            TARGET nn
            POST_BUILD
            COMMAND "codesign" "--force" "--deep" "-s" "-" "${CMAKE_CURRENT_BINARY_DIR}/*.dylib"
            COMMAND "codesign" "--force" "--deep" "-s" "-" "${CMAKE_CURRENT_BINARY_DIR}/*.pd_darwin"
            COMMAND "xattr" "-r" "-d" "com.apple.quarantine" "${CMAKE_CURRENT_BINARY_DIR}"
            COMMENT "Codesign external"
        )
    endif()
endif()

if (UNIX AND NOT APPLE)
    set(ESSENTIAL_LIBS
        "libtorch.so"
        "libtorch_cpu.so"
        "libc10.so"
        "libtorch_global_deps.so"
    )

    # Copy essential Torch libraries
    add_custom_target(copy_torch_libs)
    foreach(LIB ${ESSENTIAL_LIBS})
        add_custom_command(
            TARGET copy_torch_libs
            PRE_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                "${TORCH_INSTALL_PREFIX}/lib/${LIB}"
                "${CMAKE_CURRENT_BINARY_DIR}/${LIB}"
            COMMENT "Copying ${LIB}"
        )
    endforeach()
    add_dependencies(nn copy_torch_libs)

    set_target_properties(nn PROPERTIES 
        PREFIX ""
        SUFFIX "~.pd_linux"
        BUILD_WITH_INSTALL_RPATH TRUE
        INSTALL_RPATH_USE_LINK_PATH TRUE
        INSTALL_RPATH "$ORIGIN"
    )

    # Add libgomp as a link dependency
    target_link_libraries(nn PRIVATE gomp)
endif()

if(MSVC)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")

    # Base DLLs (these work for CPU version)
    set(REQUIRED_DLLS
        "torch_cpu.dll"
        "c10.dll"
        "fbgemm.dll"
        "libiomp5md.dll"
        "libiompstubs5md.dll"
        "uv.dll"
        "asmjit.dll"
        "torch.dll"
        "torch_global_deps.dll"
    )

    # CUDA DLL setup
    if(CUDA_FOUND OR EXISTS "${TORCH_INSTALL_PREFIX}/lib/torch_cuda.dll")
        list(APPEND REQUIRED_DLLS
            # PyTorch CUDA DLLs
            "torch_cuda.dll"
            "c10_cuda.dll"
            # Core CUDA Runtime DLLs
            "cudart64_12.dll"
            # Additional CUDA DLLs
            "cudnn64_9.dll"
            "cudnn_graph64_9.dll"
            "cudnn_engines_precompiled64_9.dll"
            "cudnn_engines_runtime_compiled64_9.dll"
            "cudnn_heuristic64_9.dll"
            "nvrtc-builtins64_120.dll"
            "cudadevrt.dll"
        )

        # VC Runtime handling
        if(DEFINED ENV{VCREDIST_PATH} AND EXISTS "$ENV{VCREDIST_PATH}/vcruntime140_1.dll")
            message(STATUS "Found VC Runtime at: $ENV{VCREDIST_PATH}")
            list(APPEND REQUIRED_DLLS "$ENV{VCREDIST_PATH}/vcruntime140_1.dll")
        else()
            message(WARNING "VC Runtime not found in VCREDIST_PATH")
        endif()

        # CUDA runtime DLL handling
        if(DEFINED ENV{CUDA_PATH})
            file(GLOB CUDA_RUNTIME_DLLS "$ENV{CUDA_PATH}/bin/*.dll")
            foreach(CUDA_DLL ${CUDA_RUNTIME_DLLS})
                get_filename_component(DLL_NAME ${CUDA_DLL} NAME)
                foreach(REQUIRED_DLL ${REQUIRED_DLLS})
                    if(DLL_NAME STREQUAL REQUIRED_DLL)
                        add_custom_command(TARGET nn POST_BUILD
                            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                                "${CUDA_DLL}"
                                "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CFG_INTDIR}/${DLL_NAME}"
                            COMMENT "Copying CUDA runtime DLL: ${DLL_NAME}"
                        )
                    endif()
                endforeach()
            endforeach()
        endif()
    endif()

    # Copy libtorch DLLs
    foreach(DLL ${REQUIRED_DLLS})
        if(EXISTS "${TORCH_INSTALL_PREFIX}/lib/${DLL}")
            add_custom_command(TARGET nn POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy_if_different
                    "${TORCH_INSTALL_PREFIX}/lib/${DLL}"
                    "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CFG_INTDIR}/${DLL}"
                COMMENT "Copying ${DLL}"
            )
        endif()
    endforeach()

    set_target_properties(nn PROPERTIES PREFIX "" SUFFIX "~.dll")
endif()

target_include_directories(nn PRIVATE "${PUREDATA_INCLUDE_DIR}")
target_link_libraries(nn PRIVATE backend)
if (MSVC)
    target_link_libraries(nn PRIVATE "${PUREDATA_BIN_DIR}/pd.lib" shlwapi.lib)
endif()
set_property(TARGET nn PROPERTY CXX_STANDARD 17)
