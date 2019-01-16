cmake_minimum_required(VERSION 3.1)
project(SVNET_RELEASE)


# ----------------------------------------------------------------------------
# Check Device
# ----------------------------------------------------------------------------
if("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "aarch64")
    find_library(DRIVEWORKS_LIBRARY "driveworks"
                PATHS "/usr/local/driveworks"
                PATH_SUFFIXES "lib"
    )
    if(DRIVEWORKS_LIBRARY)
        set(DRIVEPX2 ON)
    else()
        set(JETSON ON)
    endif()
endif()

# ----------------------------------------------------------------------------
# Directories
# ----------------------------------------------------------------------------
set(SVNET_DIR "${CMAKE_SOURCE_DIR}/svnet" CACHE PATH "SVNet Directory")
set(3RDPARTY_DIR "${CMAKE_SOURCE_DIR}/3rdparty" CACHE PATH "3rdparty Directory")


# ----------------------------------------------------------------------------
# Global Properties
# ----------------------------------------------------------------------------
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "")
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()


# ----------------------------------------------------------------------------
# CMake options for SVNET
# ----------------------------------------------------------------------------
option(SVNET_APPS_AUTOLABEL                 "Build AutoLabel"                                     )
option(SVNET_APPS_SDK_ZF_L4                 "Build SDK for ZF Sensing for L4"                     )
option(SVNET_APPS_SVNETOD                   "Build SVNetOD for SKT"                               )
option(SVNET_APPS_SVNETW                    "Build SVNetW for LGE"                                )
option(SVNET_CPU_USE_INT                    "Use integer for CPU"                                 )
option(SVNET_GPU                            "Use GPU"                                           ON)
option(SVNET_GPU_USE_FP16                   "Use float16 instead of float32 for GPU"              )
option(SVNET_GPU_USE_CUDNN                  "Use cuDNN for GPU"                                 ON)
option(SVNET_FEAT_ENABLE_OD                 "Enable Object Detection (OD) feature"              ON)
option(SVNET_FEAT_ENABLE_3D                 "Enable 3D Car feature"                             ON)
option(SVNET_FEAT_ENABLE_FSD                "Enable Free Space Detection (FSD) feature"         ON)
option(SVNET_FEAT_ENABLE_LD                 "Enable Lane Detection (LD) feature"                ON)
option(SVNET_FEAT_ENABLE_TSTL               "Enable Traffic Sign and Light (TSTL) feature"      ON)
option(SVNET_FEAT_ENABLE_TSR                "Enable Traffic Sign Recognition (TSR) feature"       )
option(SVNET_FEAT_USE_OD_DET_RESULT         "Use Detection Result of OD"                          )
option(SVNET_FEAT_USE_LD_DET_RESULT         "Use Detection Result of LD"                          )
option(SVNET_FEAT_USE_LD_SEG_MAP            "Use Segment Map of LD"                               )
option(SVNET_FEAT_USE_LD_VEHICLE_COORD      "Use Vehicle Coordinate of LD"                        )
option(SVNET_FEAT_USE_TRACKER_PER_CLASS     "Use tracker per class feature"                       )
option(SVNET_LIVEDEMO_LOGRANGE              "Prints more detailed logs for debugging"             )
option(SVNET_LIVEDEMO_STANDALONE_LD         "LD will be exclusively executed"                     )
option(SVNET_LIVEDEMO_PREVIEW               "Display preview for Livedemo"                      ON)
option(SVNET_LIVEDEMO_ENABLE_TMP_FLTR       "Enable Temporal Filtering feature for Livedemo"      )
option(SVNET_SDK_ENABLE_DRIVEWORKS          "Enable DriveWorks API in SDK"                        )
option(SVNET_SDK_V1                         "Enable SDK v1"                                     ON)
option(SVNET_SDK_V2                         "Enable SDK v2"                                       )
option(SVNET_SDK_V2_EXTENSION_TP            "Enable extension Truck Platooning in SDK v2"         )

# Unused
#set(SVNET_IMAGE_FORMAT "BGR" CACHE STRING "Image format")
#set_property(CACHE SVNET_IMAGE_FORMAT PROPERTY STRINGS "BGR;RGB;RGBA")

# MAXWELL: 52 / PASCAL: 61 / TX1: 53 / TX2: 62
set(SVNET_GPU_ARCH_VER "52;53;61;62" CACHE STRING "GPU architecture versions")

# Frame limit second, zero means unlimited
set(SVNET_FRAME_LIMIT_SECOND "1800" CACHE STRING "Frame limit second")

# Default value of LND Format
if(NOT SVNET_LIVEDEMO_LND_FORMAT)
    set(SVNET_LIVEDEMO_LND_FORMAT "DISABLE")
endif()

# Default SDK v2 sample code cases
if(SVNET_SDK_V2)
    set(SVNET_SDK_V2_SAMPLE_CASES "1;2;3;4" CACHE STRING "Case numbers of SDK v2 sample code to install")
endif()

# Platform specific options
if(UNIX)
    option(SVNET_LIVEDEMO_USE_GLX        "Use GLX instead of OpenCV for Livedemo" ON)
    option(SVNET_BUILD_ADDRESS_SANITIZER "Building with Address Sanitizer"          )
endif()

if(UNIX)
    set(default_blas_lib "OpenBLAS")
    set(blas_libs "OpenBLAS;MKL;ATLAS;LAPACK")
elseif(WIN32)
    set(default_blas_lib "MKL")
    set(blas_libs "OpenBLAS;MKL;ATLAS;LAPACK")
endif()
set(SVNET_BLAS_LIBRARY "${default_blas_lib}" CACHE STRING "BLAS Library")
set_property(CACHE SVNET_BLAS_LIBRARY PROPERTY STRINGS "${blas_libs}")

if(UNIX)
    set(default_cam_type "V4L2")
    set(cam_types "V4L2;OpenCV;GMSL")
elseif(WIN32)
    set(default_cam_type "OpenCV")
    set(cam_types "OpenCV")
endif()
set(SVNET_LIVEDEMO_CAM_TYPE "${default_cam_type}" CACHE STRING "Livedemo camera type")
set_property(CACHE SVNET_LIVEDEMO_CAM_TYPE PROPERTY STRINGS "${cam_types}")

if(UNIX)
    if(SVNET_BUILD_ADDRESS_SANITIZER)
        if(NOT "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
            message(FATAL_ERROR "Address Sanitizer in only Clang can be used...")
        endif()
    endif()
endif()


# ----------------------------------------------------------------------------
# Find available models
# ----------------------------------------------------------------------------
file(READ "${SVNET_DIR}/demo/svnet.cpp" svnet_src)

# Fetch default model from header file
string(REGEX MATCH "#define DEFAULT_NET \"([^\n]*)\"\n" default_model "${svnet_src}")
string(STRIP "${CMAKE_MATCH_1}" default_model)
string(TOUPPER "${default_model}" default_model)

# Find available models from header file list
file(GLOB model_files "${SVNET_DIR}/include/nets/*.hpp")
foreach(model_file ${model_files})
    get_filename_component(model_name "${model_file}" NAME_WE)
    string(TOUPPER "${model_name}" model_name)
    list(APPEND SVNET_MODELS "${model_name}")

    file(READ "${model_file}" model_src)
    string(REGEX MATCHALL "#include \"nets/[^\.]*\.hpp\"" model_deps "${model_src}")
    string(REGEX REPLACE "#include \"nets/([^\.]*)\.hpp\"" "\\1" model_deps "${model_deps}")
    string(TOUPPER "${model_deps}" model_deps)
    list(APPEND SVNET_MODEL_${model_name}_DEPS ${model_deps})
endforeach()
list(SORT SVNET_MODELS)

# Check unavailable model
get_cmake_property(_vars VARIABLES)
foreach(var ${_vars})
    string(FIND "${var}" "SVNET_MODEL_USE_" find_result)
    if(NOT ${find_result} EQUAL -1)
        list(APPEND defined_model_options "${var}")
    endif()
endforeach()
foreach(defined_model_option ${defined_model_options})
    if(${defined_model_option})
        string(REPLACE "SVNET_MODEL_USE_" "" defined_model "${defined_model_option}")
        message(STATUS "Check model: ${defined_model}")
        list(FIND SVNET_MODELS "${defined_model}" list_index)
        if(${list_index} EQUAL -1)
            message(FATAL_ERROR "Model ${defined_model} is not available!")
        endif()
        message(STATUS "Check model: ${defined_model} - done")
    endif()
endforeach()

# Make options
set(SVNET_MODEL_DEFAULT ${default_model} CACHE STRING "Default model for SVNET")
set_property(CACHE SVNET_MODEL_DEFAULT PROPERTY STRINGS ${SVNET_MODELS})

foreach(model ${SVNET_MODELS})
    set(is_default)
    if("${model}" STREQUAL "${default_model}")
        set(is_default ON)
    endif()

    option(SVNET_MODEL_USE_${model} "Use model '${model}'" ${is_default})
endforeach()

# Check CMake-variable dependancy
foreach(model ${SVNET_MODELS})
    if("${model}" STREQUAL "${SVNET_MODEL_DEFAULT}")
        if(NOT SVNET_MODEL_USE_${model})
            message(FATAL_ERROR "Please select SVNET_MODEL_USE_${model} for using as default...")
        endif()
    endif()
endforeach()

# Mark dependancy models
foreach(model ${SVNET_MODELS})
    foreach(model_deps ${SVNET_MODEL_${model}_DEPS})
        if(SVNET_MODEL_USE_${model})
            set(SVNET_MODEL_USE_${model_deps} ON CACHE BOOL "Use model '${model_deps}'" FORCE)
        endif()
    endforeach()
endforeach()


# ----------------------------------------------------------------------------
# Find source files
# ----------------------------------------------------------------------------
# SVNET files
file(GLOB_RECURSE svnet_src_files   "${SVNET_DIR}/src/*.cpp")
file(GLOB_RECURSE svnet_cuda_files  "${SVNET_DIR}/src/*.cu")
file(GLOB_RECURSE svnet_esw_files   "${CMAKE_SOURCE_DIR}/modules/*")


# ----------------------------------------------------------------------------
# Calculate timelock
# ----------------------------------------------------------------------------
# Get default year
if(UNIX)
    execute_process(
        COMMAND bash -c "date +%Y"
        OUTPUT_VARIABLE timelock_year)
elseif(WIN32)
    execute_process(
        COMMAND POWERSHELL -Command "Get-Date -Format 'yyyy'"
        OUTPUT_VARIABLE timelock_year)
endif()
string(STRIP ${timelock_year} timelock_year)
set(SVNET_TIMELOCK_YEAR "${timelock_year}" CACHE STRING "Timelock - Year")

# Get default month
if(UNIX)
    execute_process(
        COMMAND bash -c "date +%-m"
        OUTPUT_VARIABLE timelock_month)
elseif(WIN32)
    execute_process(
        COMMAND POWERSHELL -Command "Get-Date -Format '%M'"
        OUTPUT_VARIABLE timelock_month)
endif()
string(STRIP ${timelock_month} timelock_month)
set(SVNET_TIMELOCK_MONTH "${timelock_month}" CACHE STRING "Timelock - Month")
set_property(CACHE SVNET_TIMELOCK_MONTH PROPERTY STRINGS "1;2;3;4;5;6;7;8;9;10;11;12")

# Calculate last day
if(UNIX)
    execute_process(
        COMMAND bash -c "date -d '${SVNET_TIMELOCK_YEAR}-${SVNET_TIMELOCK_MONTH}-1 + 1month - 1day' +%d"
        OUTPUT_VARIABLE timelock_day)
elseif(WIN32)
    execute_process(
        COMMAND POWERSHELL -Command "Get-Date (Get-Date '${SVNET_TIMELOCK_YEAR}/${SVNET_TIMELOCK_MONTH}/1').AddMonths(1).AddSeconds(-1) -Format 'dd'"
        OUTPUT_VARIABLE timelock_day)
endif()
string(STRIP ${timelock_day} timelock_day)
set(SVNET_TIMELOCK_DAY "${timelock_day}" CACHE STRING "Timelock - Day" FORCE)


# ----------------------------------------------------------------------------
# Limitations
# ----------------------------------------------------------------------------
# Timelock message
if(UNIX)
    execute_process(
        COMMAND bash -c "date -d '${SVNET_TIMELOCK_YEAR}-${SVNET_TIMELOCK_MONTH}-${SVNET_TIMELOCK_DAY}' +'%b %-d, %Y'"
        OUTPUT_VARIABLE timelock_string)
elseif(WIN32)
    execute_process(
        COMMAND POWERSHELL -Command "Get-Date '${SVNET_TIMELOCK_YEAR}-${SVNET_TIMELOCK_MONTH}-${SVNET_TIMELOCK_DAY}' -Format 'MMM d, yyyy'"
        OUTPUT_VARIABLE timelock_string)
endif()
string(STRIP ${timelock_string} timelock_string)
set(timelock_message  "Notice: The expiration date is ${timelock_string}.")

# Framelock message
if(NOT ${SVNET_FRAME_LIMIT_SECOND} EQUAL 0)
    math(EXPR framelock_min "${SVNET_FRAME_LIMIT_SECOND} / 60")
    math(EXPR framelock_sec "${SVNET_FRAME_LIMIT_SECOND} % 60")
    math(EXPR framelock_frm "${SVNET_FRAME_LIMIT_SECOND} * 30")
    set(framelock_string "${framelock_min} min ${framelock_sec} sec (${framelock_frm} frames @ 30 fps)")
    set(framelock_message "Notice: The frames will be processed only for ${framelock_string}.")
endif()

# Save
set(LIMITATIONS_FILE "${CMAKE_BINARY_DIR}/LIMITATIONS")
file(WRITE  "${LIMITATIONS_FILE}" "${timelock_message}\n")
file(APPEND "${LIMITATIONS_FILE}" "${framelock_message}\n")


# ----------------------------------------------------------------------------
# Get Git revision
# ----------------------------------------------------------------------------
find_package(Git REQUIRED)
if(GIT_FOUND)
    if(UNIX)
        set(git_rev_parse "${GIT_EXECUTABLE} describe --match=NeVeRmAtCh --always --dirty")
        execute_process(
            COMMAND bash -c "cd ${SVNET_DIR} && ${git_rev_parse}"
            OUTPUT_VARIABLE SVNET_GIT_REVISION)
        execute_process(
            COMMAND bash -c "cd ${CMAKE_SOURCE_DIR} && ${git_rev_parse}"
            OUTPUT_VARIABLE SVNET_LIVEDEMO_GIT_REVISION)
    elseif(WIN32)
        set(git_rev_parse "&'${GIT_EXECUTABLE}' describe --match=NeVeRmAtCh --always --dirty")
        execute_process(
            COMMAND POWERSHELL -Command "cd '${SVNET_DIR}'; ${git_rev_parse}"
            OUTPUT_VARIABLE SVNET_GIT_REVISION)
        execute_process(
            COMMAND POWERSHELL -Command "cd '${CMAKE_SOURCE_DIR}'; ${git_rev_parse}"
            OUTPUT_VARIABLE SVNET_LIVEDEMO_GIT_REVISION)
    endif()
    string(STRIP "${SVNET_GIT_REVISION}" SVNET_GIT_REVISION)
    string(STRIP "${SVNET_LIVEDEMO_GIT_REVISION}" SVNET_LIVEDEMO_GIT_REVISION)

    if(UNIX)
        set(git_describe "${GIT_EXECUTABLE} describe --tags --match v* --dirty")
        execute_process(
            COMMAND bash -c "${git_describe}"
            OUTPUT_VARIABLE latest_git_v_tag)
    elseif(WIN32)
        set(git_describe "&'${GIT_EXECUTABLE}' describe --tags --match v* --dirty")
        execute_process(
            COMMAND POWERSHELL -Command "${git_describe}"
            OUTPUT_VARIABLE latest_git_v_tag)
    endif()
    string(REPLACE "v" "" latest_version "${latest_git_v_tag}")
    string(STRIP "${latest_version}" SVNET_LATEST_VERSION)
    message(STATUS "Latest Version: ${SVNET_LATEST_VERSION}")
endif()


# ----------------------------------------------------------------------------
# Find OpenCV
# ----------------------------------------------------------------------------
if(WIN32)
    if(MSVC_VERSION EQUAL 1800)
        if(EXISTS "${3RDPARTY_DIR}/opencv-2.4.9")
            set(OpenCV_DIR "${3RDPARTY_DIR}/opencv-2.4.9")
        endif()
    elseif((MSVC_VERSION GREATER 1900) OR (MSVC_VERSION EQUAL 1900))
        if(EXISTS "${3RDPARTY_DIR}/opencv-3.4.0")
            set(OpenCV_DIR "${3RDPARTY_DIR}/opencv-3.4.0")
        endif()
    endif()

    if(NOT OpenCV_DIR)
        string(REPLACE "\\" "/" OPENCV_DIR "$ENV{OPENCV_DIR}")
        set(OpenCV_DIR "${OPENCV_DIR}/../..")
    endif()

    set(OpenCV_STATIC ON)
endif()
find_package(OpenCV REQUIRED)
get_filename_component(OPENCV_DIR "${OpenCV_CONFIG_PATH}/../" REALPATH)


# ----------------------------------------------------------------------------
# SVNet Options
# ----------------------------------------------------------------------------
macro(SVNET_PARSE_OPTIONS ini_file postfix)
    if(EXISTS "${ini_file}")
        message(STATUS "Parse SVNet options for ${postfix}: ${ini_file}")

        file(READ ${ini_file} options_str)
        string(REPLACE "\n" "\\n" OPTIONS_STR_${postfix} "${options_str}")

        string(REPLACE "\n" ";" options "${options_str}")
        foreach(option ${options})
            unset(key)
            unset(val)

            string(REPLACE "=" ";" key_val "${option}")
            list(LENGTH key_val len)
            if(NOT len EQUAL 2)
                continue()
            endif()

            list(GET key_val 0 key)
            list(GET key_val 1 val)
            string(STRIP "${key}" key)
            string(STRIP "${val}" val)
            set(${key}_${postfix} "${val}")
        endforeach()
    else()
        message(FATAL_ERROR "There is no option file...")
    endif()
endmacro()

if(SVNET_OPTIONS_INI_FILE_IMAGE)
    file(GLOB ini_file_image "${CMAKE_SOURCE_DIR}/config/*/${SVNET_OPTIONS_INI_FILE_IMAGE}")
endif()
if(ini_file_image)
    SVNET_PARSE_OPTIONS(${ini_file_image} "IMAGE")
else()
    message(STATUS "SVNET_OPTIONS_INI_FILE_IMAGE not found...")
    set(TILING_IMAGE 0)
endif()

if(SVNET_OPTIONS_INI_FILE_VIDEO)
    file(GLOB ini_file_video "${CMAKE_SOURCE_DIR}/config/*/${SVNET_OPTIONS_INI_FILE_VIDEO}")
endif()
if(ini_file_video)
    SVNET_PARSE_OPTIONS(${ini_file_video} "VIDEO")
else()
    message(STATUS "SVNET_OPTIONS_INI_FILE_VIDEO not found...")
    set(TILING_VIDEO 0)
endif()

configure_file(cmake/options_image.h.in options_image.h)
configure_file(cmake/options_video.h.in options_video.h)
configure_file(cmake/options_etc.h.in options_etc.h)

include_directories(${CMAKE_CURRENT_BINARY_DIR})


# ----------------------------------------------------------------------------
# Configure build flags
# ----------------------------------------------------------------------------

# CXX flags
if(UNIX)
    if(SVNET_BUILD_ADDRESS_SANITIZER)
        set(CMAKE_CXX_FLAGS_RELEASE     "-fPIC -O -g -fsanitize=address -fno-omit-frame-pointer")
    else()
        set(CMAKE_CXX_FLAGS_RELEASE     "-fPIC -O2 -DNDEBUG")
    endif()
elseif(WIN32)
    set(CMAKE_CXX_FLAGS_DEBUG           "/MTd /Od /Ob0 /D_DEBUG /DDEBUG /Zi /RTC1")
    set(CMAKE_CXX_FLAGS_MINSIZEREL      "/MT  /O1 /Ob1 /DNDEBUG")
    set(CMAKE_CXX_FLAGS_RELEASE         "/MT  /O2 /Ob2 /DNDEBUG")
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO  "/MT  /O2 /Ob1 /DNDEBUG /Zi")
endif()

# WORKAROUND
include_directories(
    "${CMAKE_SOURCE_DIR}/apps/svnetod/include"
)

# Common flags
include_directories(
    "${SVNET_DIR}/include"
    "${SVNET_DIR}/exp"
)

if(UNIX)
    add_compile_options(
        -pthread
        -std=c++11
    )
elseif(WIN32)
    add_compile_options(
        /wd4819
    )
endif()

if(JETSON OR DRIVEPX2)
    add_compile_options(
      -ffast-math
      -march=armv8-a+crypto
      -mcpu=cortex-a57+crypto
      -ftree-vectorize
      #-flto
      #-ffat-lto-objects
      #-fwhole-program
    )
    add_definitions(
      -DUSE_NEON
    )
endif()

add_definitions(
    -DSPEEDUP_RPN
    -DSPEEDUP_IM2COL
    -DSPEEDUP_GRP_CONV
    -DSPEEDUP_GRIDCONV
    -DTIMELOCK_YEAR=${SVNET_TIMELOCK_YEAR}
    -DTIMELOCK_MONTH=${SVNET_TIMELOCK_MONTH}
    -DTIMELOCK_DAY=${SVNET_TIMELOCK_DAY}
   #-DIMAGE_FORMAT=${SVNET_IMAGE_FORMAT}
)
if(WIN32)
    add_definitions(
        -DNOMINMAX
    )
endif()

# CPU flags
if(SVNET_CPU_USE_INT)
    add_definitions(-DUSE_INT)
endif()

# GPU flags
if(SVNET_GPU)
    if(WIN32)
        # Find CUDA for Visual Studio
        if    ((MSVC_VERSION EQUAL 1914) OR (MSVC_VERSION GREATER 1914))    # _MSC_VER >= VS2017-15.7
            string(REPLACE "\\" "/" cuda_path_v10_0 "$ENV{CUDA_PATH_V10_0}")
            set(cuda_path ${cuda_path_v10_0})
            set(min_cuda_version "10.0")
        elseif((MSVC_VERSION EQUAL 1911) OR (MSVC_VERSION GREATER 1911))    # _MSC_VER >= VS2017-15.3
            string(REPLACE "\\" "/" cuda_path_v9_2 "$ENV{CUDA_PATH_V9_2}")
            set(cuda_path ${cuda_path_v9_2})
            set(min_cuda_version "9.2")
        elseif((MSVC_VERSION EQUAL 1910) OR (MSVC_VERSION GREATER 1910))    # _MSC_VER >= VS2017-15.0
            string(REPLACE "\\" "/" cuda_path_v9_0 "$ENV{CUDA_PATH_V9_0}")
            string(REPLACE "\\" "/" cuda_path_v9_1 "$ENV{CUDA_PATH_V9_1}")
            if    (IS_DIRECTORY ${cuda_path_v9_0})
                set(cuda_path ${cuda_path_v9_0})
            elseif(IS_DIRECTORY ${cuda_path_v9_1})
                set(cuda_path ${cuda_path_v9_1})
            endif()
            set(min_cuda_version "9.0")
        else()
            string(REPLACE "\\" "/" cuda_path "$ENV{CUDA_PATH}")
        endif()

        if(IS_DIRECTORY ${cuda_path})
            set(CUDA_TOOLKIT_ROOT_DIR ${cuda_path})
            find_package(CUDA ${min_cuda_version} REQUIRED)
        else()  # Fallback
            message(WARNING "Radically, CUDA ${min_cuda_version} is required for this compiler. Please select right Platform Toolset for this CUDA.")
            find_package(CUDA REQUIRED)
        endif()
    else()
        # Find CUDA for Linux
        find_package(CUDA REQUIRED)
    endif()

    set(CUDA_PROPAGATE_HOST_FLAGS OFF)
endif()

if(SVNET_GPU_USE_CUDNN)
    if(NOT SVNET_GPU)
        message(FATAL_ERROR "If you want to use cuDNN, please turn on SVNET_GPU first...")
    endif()

    add_definitions(-DUSE_CUDNN)

    if(UNIX)
        # Add cuDNN static library first since dependency
        find_cuda_helper_libs("cudnn_static")
        find_cuda_helper_libs("culibos")
        list(APPEND svnet_lib_items ${CUDA_cudnn_static_LIBRARY}
                                    ${CUDA_culibos_LIBRARY})
    elseif(WIN32)
        # Find right version of cuDNN
        # 1. Find from CUDA path
        find_cuda_helper_libs("cudnn")
        get_filename_component(cudnn_dir ${CUDA_cudnn_LIBRARY} DIRECTORY)
        string(FIND "${cudnn_dir}" "${CUDA_TOOLKIT_ROOT_DIR}" cudnn_dir_find)

        # 2. Check right version with CUDA
        if(${cudnn_dir_find} EQUAL -1)
            # 3. If not matched, find from 3rdparty
            unset(CUDA_cudnn_LIBRARY CACHE)
            file(GLOB cudnn_3rdparty_dir "${3RDPARTY_DIR}/cudnn-${CUDA_VERSION_STRING}-*")
            if(IS_DIRECTORY ${cudnn_3rdparty_dir})
                find_library(CUDA_cudnn_LIBRARY "cudnn"
                    PATHS "${cudnn_3rdparty_dir}"
                    PATH_SUFFIXES "cuda/lib/x64" "lib/x64"
                    NO_DEFAULT_PATH
                )
                find_path(CUDA_cudnn_INCLUDE "cudnn.h"
                    PATHS "${cudnn_3rdparty_dir}"
                    PATH_SUFFIXES "cuda/include" "include"
                    NO_DEFAULT_PATH
                )
                file(GLOB CUDA_cudnn_DLL "${cudnn_3rdparty_dir}/cuda/bin/cudnn*.dll"
                                         "${cudnn_3rdparty_dir}/bin/cudnn*.dll")
                set(CUDA_cudnn_DLL "${CUDA_cudnn_DLL}" CACHE STRING "\"cudnn\" DLL" FORCE)

                message(STATUS "Found cuDNN: ${CUDA_cudnn_LIBRARY}")
                if(NOT CUDA_cudnn_DLL)
                    message(WARNING "cuDNN DLL is not found...")
                endif()
            else()
                set(CUDA_cudnn_INCLUDE "CUDA_cudnn_INCLUDE-NOTFOUND" CACHE STRING "\"cudnn\" include" FORCE)
                set(CUDA_cudnn_LIBRARY "CUDA_cudnn_LIBRARY-NOTFOUND" CACHE STRING "\"cudnn\" library" FORCE)
            endif()
        endif()
        list(APPEND svnet_lib_items ${CUDA_cudnn_LIBRARY})
    endif()
endif()

if(SVNET_GPU)
    if(JETSON AND CUDA_USE_STATIC_CUDA_RUNTIME)
        message(FATAL_ERROR "CUDA_USE_STATIC_CUDA_RUNTIME must be OFF in JETSON...")
    endif()

    string(REPLACE "," ";" arch_ver_list "${SVNET_GPU_ARCH_VER}")
    foreach(arch_ver ${arch_ver_list})
        if(SVNET_GPU_USE_FP16)
            if(${arch_ver} LESS 53)
                message(FATAL_ERROR "CUDA Compute Capability ${arch_ver} doesn't support FP16...")
            endif()
        endif()
        set(gpu_arch_flags "${gpu_arch_flags} -gencode arch=compute_${arch_ver},code=sm_${arch_ver}")
        add_definitions(-DCUDA_COMPUTE_CAPABILITY_${arch_ver})
    endforeach()
    if(UNIX)
        set(CUDA_NVCC_FLAGS "${gpu_arch_flags} -std=c++11 -Xcompiler \"-fPIC\"")
    elseif(WIN32)
        set(CUDA_NVCC_FLAGS "${gpu_arch_flags} -Xcompiler \"/wd4819\"")
        set(CUDA_NVCC_FLAGS_DEBUG "${gpu_arch_flags} -Xcompiler \"/wd4819 /MTd\"")
    endif()

    add_definitions(-DUSE_GPU)
    include_directories(
        "${CUDA_TOOLKIT_ROOT_DIR}/include"
        ${CUDA_cudnn_INCLUDE}   # Without quote to prevent empty-string error
    )

    if(UNIX)
        if(CUDA_USE_STATIC_CUDA_RUNTIME)
            find_cuda_helper_libs("cublas_static")
            find_cuda_helper_libs("curand_static")
            list(APPEND svnet_lib_items ${CUDA_cublas_static_LIBRARY}
                                        ${CUDA_curand_static_LIBRARY})
        else()
            list(APPEND svnet_lib_items ${CUDA_cublas_LIBRARY}
                                        ${CUDA_curand_LIBRARY})
        endif()
    elseif(WIN32)
        list(APPEND svnet_lib_items ${CUDA_cublas_LIBRARY}
                                    ${CUDA_curand_LIBRARY})
    endif()
endif()

if(SVNET_GPU_USE_FP16)
    if(NOT SVNET_GPU)
        message(FATAL_ERROR "If you want to use FP16, please turn on SVNET_GPU first...")
    endif()

    add_definitions(-DUSE_FP16)
endif()

# BLAS library flags
if(SVNET_BLAS_LIBRARY STREQUAL "OpenBLAS")
    if(UNIX)
        find_library(OPENBLAS_LIBRARY "openblas")
        list(APPEND svnet_lib_items ${OPENBLAS_LIBRARY})
    elseif(WIN32)
        message(FATAL_ERROR "OpenBLAS is not supported yet...")
    endif()
elseif(SVNET_BLAS_LIBRARY STREQUAL "MKL")
    if(UNIX)
        message(FATAL_ERROR "MKL is not supported yet...")
        find_path(MKL_INCLUDE_DIR "mkl.h" "/opt/intel/mkl/include")
        find_library(MKL_RT_LIBRARY "mkl_rt" "/opt/intel/mkl/lib/*")
        list(APPEND svnet_lib_items ${MKL_RT_LIBRARY})
    elseif(WIN32)
        set(MKL_3RDPARTY_DIR "${3RDPARTY_DIR}/mkl-2018.1.156")

        find_path(MKL_INCLUDE_DIR "mkl.h" "${MKL_3RDPARTY_DIR}/include" "$ENV{MKL_DIR}/include")

        find_library(MKL_CORE_STATIC_LIBRARY "mkl_core"       "${MKL_3RDPARTY_DIR}/lib" "$ENV{MKL_DIR}/lib/intel64")
        find_library(MKL_LP64_STATIC_LIBRARY "mkl_intel_lp64" "${MKL_3RDPARTY_DIR}/lib" "$ENV{MKL_DIR}/lib/intel64")
        find_library(MKL_SEQN_STATIC_LIBRARY "mkl_sequential" "${MKL_3RDPARTY_DIR}/lib" "$ENV{MKL_DIR}/lib/intel64")
        list(APPEND svnet_lib_items ${MKL_CORE_STATIC_LIBRARY}
                                    ${MKL_LP64_STATIC_LIBRARY}
                                    ${MKL_SEQN_STATIC_LIBRARY})
    endif()

    add_definitions(-DUSE_MKL)
    include_directories("${MKL_INCLUDE_DIR}")
elseif(SVNET_BLAS_LIBRARY STREQUAL "ATLAS")
    if(UNIX)
        find_library(ATLAS_LIBRARY "atlas")
        find_library(CBLAS_LIBRARY "cblas")
        list(APPEND svnet_lib_items ${ATLAS_LIBRARY}
                                    ${CBLAS_LIBRARY})
    elseif(WIN32)
        message(FATAL_ERROR "ATLAS is not supported yet...")
    endif()
elseif(SVNET_BLAS_LIBRARY STREQUAL "LAPACK")
    if(UNIX)
        find_library(LAPACK_LIBRARY liblapack.a "lapack")
        list(APPEND svnet_lib_items ${LAPACK_LIBRARY})
    elseif(WIN32)
        message(FATAL_ERROR "LAPACK is not supported yet...")
    endif()
endif()

# Engine feature flags
if(NOT SVNET_FEAT_ENABLE_OD  AND
   NOT SVNET_FEAT_ENABLE_3D  AND
   NOT SVNET_FEAT_ENABLE_FSD AND
   NOT SVNET_FEAT_ENABLE_LD  AND
   NOT SVNET_FEAT_ENABLE_TSTL)
    message(FATAL_ERROR "Please select the feature at least one...")
endif()

if(SVNET_FEAT_ENABLE_OD)
    add_definitions(-DENABLE_GOD)
endif()

if(SVNET_FEAT_ENABLE_3D)
    if(NOT SVNET_FEAT_ENABLE_OD)
        message(FATAL_ERROR "If you want to enable 3D, please turn on SVNET_FEAT_ENABLE_OD first...")
    endif()

    add_definitions(-DENABLE_3D)
endif()

if(SVNET_FEAT_ENABLE_FSD)
    add_definitions(-DENABLE_FSD)
endif()

if(SVNET_FEAT_ENABLE_LD)
    add_definitions(-DENABLE_LD)
endif()

if(SVNET_FEAT_ENABLE_TSTL)
    add_definitions(-DENABLE_TSTL)
endif()

if(SVNET_FEAT_ENABLE_TSR)
    if(NOT SVNET_FEAT_ENABLE_TSTL)
        message(FATAL_ERROR "If you want to enable TSR, please turn on SVNET_FEAT_ENABLE_TSTL first...")
    endif()

    add_definitions(-DENABLE_TSR)
endif()

if(SVNET_FEAT_ENABLE_VL)
    add_definitions(-DENABLE_VL -DRETURN_VL)
endif()

if(SVNET_FEAT_USE_TRACKER_PER_CLASS)
    add_definitions(-DUSE_OPTION_MAX_TRACKER_PER_CLASS)
endif()

# OD options
if(SVNET_FEAT_USE_OD_DET_RESULT)
    add_definitions(-DRETURN_OD_DETECTION_RESULT)
endif()

# LD options
if(SVNET_FEAT_USE_LD_VEHICLE_COORD)
    add_definitions(-DLD_USE_VEHICLE_COORDINATE)
endif()
if(SVNET_FEAT_USE_LD_SEG_MAP)
    add_definitions(-DRETURN_LANE_SEG_MAP)
endif()
if(SVNET_FEAT_USE_LD_DET_RESULT)
    add_definitions(-DRETURN_LANE_DETECTION_RESULT)
endif()

# Model flags
string(TOLOWER ${SVNET_MODEL_DEFAULT} default_model)

# Common libraries
if(UNIX)
    list(APPEND svnet_lib_items "m"
                                "stdc++"
                                "pthread"
                                "dl")
endif()


# ----------------------------------------------------------------------------
# Static libraries for Models
# ----------------------------------------------------------------------------
find_path(SVNET_WEIGHTS_ROOT_DIR "weights" ${CMAKE_SOURCE_DIR}
                                           ${SVNET_DIR})

if(NOT SVNET_WEIGHTS_ROOT_DIR)
    message(FATAL_ERROR "Can't locate SVNET_WEIGHTS_ROOT_DIR...")
endif()

foreach(model ${SVNET_MODELS})
    string(TOLOWER "${model}" model_target)
    if(SVNET_MODEL_USE_${model})
        if("${SVNET_MODEL_${model}_DEPS}" STREQUAL "")
            # The model which has dependancies will be omitted

            # Find model files
            file(GLOB model_files "${SVNET_WEIGHTS_ROOT_DIR}/weights/${model_target}_wt*.cpp"
                                  "${SVNET_WEIGHTS_ROOT_DIR}/weights/${model_target}_int8_wt*.cpp")

            list(LENGTH model_files files_count)
            if(${files_count} EQUAL 0)
                message(FATAL_ERROR "Weight of the model ${model} not found...")
            endif()

            # Target for model
            add_library(${model_target} STATIC
                ${model_files}
            )
            target_compile_definitions(${model_target} PUBLIC "-D${model}")

            set_property(TARGET ${model_target} PROPERTY FOLDER "MODELS")

            # Link a model
            list(APPEND SVNET_MODEL_LIB_ITEMS ${model_target})
        endif()
    endif()
endforeach()


# ----------------------------------------------------------------------------
# CUDA Static Library
# ----------------------------------------------------------------------------
if(SVNET_GPU)
    set(TARGET_CUDA "svnet_core_cuda")
    cuda_add_library(${TARGET_CUDA} STATIC
        ${svnet_cuda_files}
    )
    set_property(TARGET ${TARGET_CUDA} PROPERTY FOLDER "SVNET")

    source_group("SVNet CUDA Files" FILES ${svnet_cuda_files})
endif()


# ----------------------------------------------------------------------------
# Installing Configuration
# ----------------------------------------------------------------------------
set(CMAKE_INSTALL_PREFIX "${CMAKE_SOURCE_DIR}/install" CACHE PATH "Install path prefix, prepended onto install directories." FORCE)
file(GLOB LICENSE_FILE "${CMAKE_SOURCE_DIR}/LICENSE.txt")

SET(CMAKE_INSTALL_SO_NO_EXE OFF CACHE BOOL "Install .so files without exe bit." FORCE)


# ----------------------------------------------------------------------------
# Sub Directories
# ----------------------------------------------------------------------------
add_subdirectory(apps)
add_subdirectory(sdk)


# ----------------------------------------------------------------------------
# Packaging
# ----------------------------------------------------------------------------
set(CPACK_GENERATOR ZIP)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_PACKAGE_VERSION "${SVNET_LATEST_VERSION}")
set(CPACK_PACKAGE_NAME "SVNet-ESW")
set(CPACK_OUTPUT_FILE_PREFIX "${CMAKE_SOURCE_DIR}/packages")
set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")
include(CPack)

