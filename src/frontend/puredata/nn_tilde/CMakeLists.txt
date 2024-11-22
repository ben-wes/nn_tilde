cmake_minimum_required(VERSION 3.0)
project(nn_tilde_pd)
find_package(Torch REQUIRED)

file(GLOB SRC *.cpp)

if (MSVC)
   set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
endif()

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

if (APPLE)
    set_target_properties(nn PROPERTIES PREFIX "" SUFFIX "~.pd_darwin")
	set_target_properties(nn PROPERTIES
		BUILD_WITH_INSTALL_RPATH FALSE
		LINK_FLAGS "-Wl,-rpath,@loader_path/"
	)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -undefined dynamic_lookup -stdlib=libc++")
endif()


if (UNIX AND NOT APPLE)
    set_target_properties(nn PROPERTIES PREFIX "" SUFFIX "~.pd_linux")
endif()

if (MSVC)
    set_target_properties(nn PROPERTIES PREFIX "" SUFFIX "~.dll")
endif()

if (MSVC) # COPY HELP FILES
    add_custom_command(TARGET nn POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${CMAKE_SOURCE_DIR}/frontend/puredata/nn_tilde/nn~-help.pd"
            "${CMAKE_CURRENT_BINARY_DIR}/Release/nn~-help.pd"
        COMMENT "Copy Help File"
    )
else()
	add_custom_command(TARGET nn POST_BUILD
	    COMMAND ${CMAKE_COMMAND} -E copy
	        "${CMAKE_SOURCE_DIR}/frontend/puredata/nn_tilde/nn~-help.pd"
	        "${CMAKE_CURRENT_BINARY_DIR}/nn~-help.pd"
	    COMMENT "Copy Help File"
	)
endif()

if (APPLE) # COPY TORCH DYLIB IN THE LOADER FOLDER
	add_custom_command(
		TARGET nn
		POST_BUILD
		COMMAND cp "${TORCH_INSTALL_PREFIX}/lib/*.dylib" "${CMAKE_CURRENT_BINARY_DIR}/"
		COMMENT "Copy Torch Libraries"
	)
endif()

if (UNIX AND NOT APPLE)
    set_target_properties(nn PROPERTIES PREFIX "" SUFFIX "~.pd_linux")
    
    # Define essential libraries to copy
    set(ESSENTIAL_LIBS
        "libtorch.so"
        "libtorch_cpu.so"
        "libc10.so"
        "libtorch_global_deps.so"
    )
    
    if(CUDA_FOUND OR EXISTS "${TORCH_INSTALL_PREFIX}/lib/libtorch_cuda.so")
        list(APPEND ESSENTIAL_LIBS
            "libtorch_cuda.so"
            "libc10_cuda.so"
            "libtorch_cuda_linalg.so"
            "libcaffe2_nvrtc.so"
            "libnvrtc-builtins.so"
        )
    endif()

    # Copy only essential libraries
    foreach(LIB ${ESSENTIAL_LIBS})
        add_custom_command(
            TARGET nn
            POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                "${TORCH_INSTALL_PREFIX}/lib/${LIB}"
                "${CMAKE_CURRENT_BINARY_DIR}/${LIB}"
            COMMENT "Copying ${LIB}"
        )
    endforeach()
endif()

if(MSVC)
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

    # Add CUDA DLLs only if CUDA is found
    if(CUDA_FOUND OR EXISTS "${TORCH_INSTALL_PREFIX}/lib/torch_cuda.dll")
        list(APPEND REQUIRED_DLLS
            "torch_cuda.dll"
            "c10_cuda.dll"
            "cudart64_12.dll"
            "cublas64_12.dll"
            "cublasLt64_12.dll"
            "cusparse64_12.dll"
            "cufft64_11.dll"
            "cusolver64_11.dll"
            "cudnn64_9.dll"
        )

        # Try multiple possible locations for VC Runtime
        set(VC_POSSIBLE_PATHS
            # GitHub Actions paths
            "$ENV{VCToolsRedistDir}/x64"
            "$ENV{VSINSTALLDIR}/VC/Redist/MSVC/14.30.30704/x64/Microsoft.VC143.CRT"
            # Local Visual Studio paths
            "C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Redist/MSVC/14.30.30704/x64/Microsoft.VC143.CRT"
            "C:/Program Files/Microsoft Visual Studio/2022/Enterprise/VC/Redist/MSVC/14.30.30704/x64/Microsoft.VC143.CRT"
        )

        # Find vcruntime140_1.dll
        foreach(VC_PATH ${VC_POSSIBLE_PATHS})
            if(EXISTS "${VC_PATH}/vcruntime140_1.dll")
                list(APPEND REQUIRED_DLLS "${VC_PATH}/vcruntime140_1.dll")
                set(VC_CRT_PATH ${VC_PATH})
                break()
            endif()
        endforeach()
    endif()

    # Copy DLLs to output directory
    foreach(DLL ${REQUIRED_DLLS})
        # Check if it's a full path (VC runtime) or just filename (libtorch DLLs)
        if(DLL MATCHES "^.*[\\/].*\\.dll$")
            # Full path
            add_custom_command(TARGET nn POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy_if_different
                    "${DLL}"
                    "${CMAKE_CURRENT_BINARY_DIR}/Release/${DLL}"
                COMMENT "Copying ${DLL}"
            )
        else()
            # Just filename, look in TORCH_INSTALL_PREFIX
            add_custom_command(TARGET nn POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy_if_different
                    "${TORCH_INSTALL_PREFIX}/lib/${DLL}"
                    "${CMAKE_CURRENT_BINARY_DIR}/Release/${DLL}"
                COMMENT "Copying ${DLL}"
            )
        endif()
    endforeach()
endif()

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

target_include_directories(nn PRIVATE "${PUREDATA_INCLUDE_DIR}")
target_link_libraries(nn PRIVATE backend)
if (MSVC)
	target_link_libraries(nn PRIVATE "${PUREDATA_BIN_DIR}/pd.lib")
endif()
set_property(TARGET nn PROPERTY CXX_STANDARD 17)
