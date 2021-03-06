# The name of our project is "VULKAN". CMakeLists files in this project can
# refer to the root source directory of the project as ${VULKAN_SOURCE_DIR} and
# to the root binary directory of the project as ${VULKAN_BINARY_DIR}.
cmake_minimum_required(VERSION 2.8.11)
project (VULKAN_SAMPLES)
# set (CMAKE_VERBOSE_MAKEFILE 1)

# The API_NAME allows renaming builds to avoid conflicts with installed SDKs
# The MAJOR number of the version we're building, used in naming
# <api-name>-<major>.dll (and other files).
set(API_NAME "Vulkan" CACHE STRING "API name to use when building")
set(MAJOR "1")
string(TOLOWER ${API_NAME} API_LOWERCASE)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
find_package(PythonInterp 3 REQUIRED)

option(USE_CCACHE "Use ccache" OFF)
if (USE_CCACHE)
    find_program(CCACHE_FOUND ccache)
    if(CCACHE_FOUND)
        set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
        set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
    endif(CCACHE_FOUND)
endif()

# Enable cmake folders
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(LVL_TARGET_FOLDER lvl_cmake_targets)

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(FALLBACK_CONFIG_DIRS "/etc/xdg" CACHE STRING
        "Search path to use when XDG_CONFIG_DIRS is unset or empty or the current process is SUID/SGID. Default is freedesktop compliant.")
    set(FALLBACK_DATA_DIRS "/usr/local/share:/usr/share" CACHE STRING
        "Search path to use when XDG_DATA_DIRS is unset or empty or the current process is SUID/SGID. Default is freedesktop compliant.")

    include(FindPkgConfig)
    option(BUILD_WSI_XCB_SUPPORT "Build XCB WSI support" ON)
    option(BUILD_WSI_XLIB_SUPPORT "Build Xlib WSI support" ON)
    option(BUILD_WSI_WAYLAND_SUPPORT "Build Wayland WSI support" ON)
    option(BUILD_WSI_MIR_SUPPORT "Build Mir WSI support" OFF)
    set(DEMOS_WSI_SELECTION "XCB" CACHE STRING "Select WSI target for demos (XCB, XLIB, WAYLAND, MIR, DISPLAY)")
    set(SAMPLES_WSI_SELECTION "XCB" CACHE STRING "Select WSI target for api-samples (XCB, WAYLAND, DISPLAY)")

    if (BUILD_WSI_XCB_SUPPORT)
        find_package(XCB REQUIRED)
    endif()

    if (BUILD_WSI_XLIB_SUPPORT)
        find_package(X11 REQUIRED)
    endif()

    if (BUILD_WSI_WAYLAND_SUPPORT)
        find_package(Wayland REQUIRED)
        include_directories(${WAYLAND_CLIENT_INCLUDE_DIR})
    endif()

    if (BUILD_WSI_MIR_SUPPORT)
        find_package(Mir REQUIRED)
    endif()

    # This option can be used to suppress the installation of artifacts from the
    # Vulkan-LoaderAndValidationLayers repo while running "make install" for the
    # VulkanTools and VulkanSamples repos.  This can be used to prevent the
    # overwriting of LVL artifacts when installing these downstream repos.
    option(INSTALL_LVL_FILES "Install content from LoaderAndValidationLayers repo" ON)
endif()

set(SCRIPTS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/scripts")

# Header file for CMake settings
include_directories("${PROJECT_SOURCE_DIR}/include")

if (CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID MATCHES "Clang")
    set(COMMON_COMPILE_FLAGS "-Wall -Wextra -Wno-unused-parameter -Wno-missing-field-initializers")
    set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -fno-strict-aliasing -fno-builtin-memcmp")

    # For GCC version 7.1 or greater, we need to disable the implicit fallthrough warning since
    # there's no consistent way to satisfy all compilers until they all accept the C++17 standard
    if (CMAKE_COMPILER_IS_GNUCC AND NOT (CMAKE_CXX_COMPILER_VERSION LESS 7.1))
        set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -Wimplicit-fallthrough=0")
    endif()

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 ${COMMON_COMPILE_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${COMMON_COMPILE_FLAGS} -std=c++11 -fno-rtti")
    if (UNIX)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden")
    endif()
endif()

if(WIN32)
    # Treat warnings as errors
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/WX>")
    # Disable RTTI
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/GR->")
    # Warn about nested declarations
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/w34456>")
    # Warn about potentially uninitialized variables
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/w34701>")
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/w34703>")
    # Warn about different indirection types.
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/w34057>")
    # Warn about signed/unsigned mismatch.
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/w34245>")
endif()

if(NOT WIN32)
    set (BUILDTGT_DIR build)
    set (BINDATA_DIR Bin)
    set (LIBSOURCE_DIR Lib)
else()
    # is WIN32
    option(DISABLE_BUILD_PATH_DECORATION "Disable the decoration of the gslang and SPIRV-Tools build path with MSVC build type info" OFF)
    option(DISABLE_BUILDTGT_DIR_DECORATION "Disable the decoration of the gslang and SPIRV-Tools build path with target info" OFF)

    # For Windows, since 32-bit and 64-bit items can co-exist, we build each in its own build directory.
    # 32-bit target data goes in build32, and 64-bit target data goes into build.  So, include/link the
    # appropriate data at build time.
    if (DISABLE_BUILDTGT_DIR_DECORATION)
        set (BUILDTGT_DIR "")
        set (BINDATA_DIR "")
        set (LIBSOURCE_DIR "")
    elseif (CMAKE_CL_64)
        set (BUILDTGT_DIR build)
        set (BINDATA_DIR Bin)
        set (LIBSOURCE_DIR Lib)
    else()
        set (BUILDTGT_DIR build32)
        set (BINDATA_DIR Bin32)
        set (LIBSOURCE_DIR Lib32)
    endif()
endif()

option(BUILD_LOADER "Build loader" ON)
option(BUILD_TESTS "Build tests" ON)
option(BUILD_LAYERS "Build layers" ON)
option(BUILD_DEMOS "Build demos" ON)
option(BUILD_VKJSON "Build vkjson" ON)
option(BUILD_UTILITIES "Build WSIWindow and Teapots" ON)
option(BUILD_ICD "Build icd" ON)
option(CUSTOM_GLSLANG_BIN_ROOT "Use the user defined GLSLANG_BINARY_ROOT" OFF)
option(CUSTOM_SPIRV_TOOLS_BIN_ROOT "Use the user defined SPIRV_TOOLS*BINARY_ROOT paths" OFF)

#Choose natural default paths for glslang and SPIRV-Tools binaries to support custom definition by the user on the CMake command line or in the GUI
set(GLSLANG_BINARY_ROOT "${CMAKE_BINARY_DIR}/../glslang" CACHE STRING "User defined path to the glslang binaries for this project")
set(SPIRV_TOOLS_BINARY_ROOT "${CMAKE_BINARY_DIR}/../SPIRV-Tools" CACHE STRING "User defined path to the SPIRV-Tools binaries for this project")
set(SPIRV_TOOLS_OPT_BINARY_ROOT "${CMAKE_BINARY_DIR}/../SPIRV-Tools/opt" CACHE STRING "User defined path to the SPIRV-Tools-opt binaries for this project")

# Define a variable for a default root location to the gslang, SPIRV-Tools and other external sources and cache it to allow the user to customize it as needed
set(EXTERNAL_SOURCE_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/external" CACHE STRING "Root path to external sources such as glslang and SPIRV-Tools")


if (WIN32)
    if(CUSTOM_GLSLANG_BIN_ROOT)
        set(GSLANG_FINAL_BINARY_PATH ${GLSLANG_BINARY_ROOT}/${BUILDTGT_DIR})
    else()
        set(GSLANG_FINAL_BINARY_PATH "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}")
    endif()

    if(DISABLE_BUILD_PATH_DECORATION)
        set (DEBUG_DECORATION "")
        set (RELEASE_DECORATION "")
    else()
        set (DEBUG_DECORATION "Debug")
        set (RELEASE_DECORATION "Release")
    endif()

    # Take some steps to set up a variable pointing to the final glslang binaries given the variety of input options
    set (GLSLANG_SEARCH_PATH "${GSLANG_FINAL_BINARY_PATH}/glslang/${RELEASE_DECORATION}"
                             "${GSLANG_FINAL_BINARY_PATH}/glslang/OSDependent/Windows/${RELEASE_DECORATION}"
                             "${GSLANG_FINAL_BINARY_PATH}/hlsl/${RELEASE_DECORATION}"
                             "${GSLANG_FINAL_BINARY_PATH}/OGLCompilersDLL/${RELEASE_DECORATION}"
                             "${GSLANG_FINAL_BINARY_PATH}/SPIRV/${RELEASE_DECORATION}" )

    set (GLSLANG_DEBUG_SEARCH_PATH "${GSLANG_FINAL_BINARY_PATH}/glslang/${DEBUG_DECORATION}"
                                   "${GSLANG_FINAL_BINARY_PATH}/glslang/OSDependent/Windows/${DEBUG_DECORATION}"
                                   "${GSLANG_FINAL_BINARY_PATH}/hlsl/${DEBUG_DECORATION}"
                                   "${GSLANG_FINAL_BINARY_PATH}/OGLCompilersDLL/${DEBUG_DECORATION}"
                                   "${GSLANG_FINAL_BINARY_PATH}/SPIRV/${DEBUG_DECORATION}")

    if(CUSTOM_SPIRV_TOOLS_BIN_ROOT)
        set (SPIRV_TOOLS_SEARCH_PATH "${SPIRV_TOOLS_BINARY_ROOT}/${BUILDTGT_DIR}/source/${RELEASE_DECORATION}")
        set (SPIRV_TOOLS_DEBUG_SEARCH_PATH "${SPIRV_TOOLS_BINARY_ROOT}/${BUILDTGT_DIR}/source/${DEBUG_DECORATION}")
        set (SPIRV_TOOLS_OPT_SEARCH_PATH "${SPIRV_TOOLS_OPT_BINARY_ROOT}/${BUILDTGT_DIR}/source/opt/${RELEASE_DECORATION}")
        set (SPIRV_TOOLS_OPT_DEBUG_SEARCH_PATH "${SPIRV_TOOLS_OPT_BINARY_ROOT}/${BUILDTGT_DIR}/source/opt/${DEBUG_DECORATION}")
    else()
        set (SPIRV_TOOLS_SEARCH_PATH "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}/External/spirv-tools/source/${RELEASE_DECORATION}")
        set (SPIRV_TOOLS_DEBUG_SEARCH_PATH "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}/External/spirv-tools/source/${DEBUG_DECORATION}")
        set (SPIRV_TOOLS_OPT_SEARCH_PATH "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}/External/spirv-tools/source/opt/${RELEASE_DECORATION}")
        set (SPIRV_TOOLS_OPT_DEBUG_SEARCH_PATH "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}/External/spirv-tools/source/opt/${DEBUG_DECORATION}")
    endif()
else()
    # not WIN32
    if(CUSTOM_GLSLANG_BIN_ROOT)
        set (GLSLANG_SEARCH_PATH "${GLSLANG_BINARY_ROOT}/install/lib"
                                 "${GLSLANG_BINARY_ROOT}/glslang"
                                 "${GLSLANG_BINARY_ROOT}/glslang/OSDependent/Unix"
                                 "${GLSLANG_BINARY_ROOT}/OGLCompilersDLL"
                                 "${GLSLANG_BINARY_ROOT}/SPIRV"
                                 "${GLSLANG_BINARY_ROOT}/hlsl"
                                 "${GLSLANG_BINARY_ROOT}/StandAlone")
    else()
        set (GLSLANG_SEARCH_PATH "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}/install/lib" "${CMAKE_SOURCE_DIR}/../x86_64/lib/glslang" )
    endif()

    if(CUSTOM_SPIRV_TOOLS_BIN_ROOT)
        set (SPIRV_TOOLS_SEARCH_PATH "${SPIRV_TOOLS_BINARY_ROOT}/source" )
        set (SPIRV_TOOLS_OPT_SEARCH_PATH "${SPIRV_TOOLS_OPT_BINARY_ROOT}/source/opt" )
    else()
        set (SPIRV_TOOLS_SEARCH_PATH "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}/External/spirv-tools/source" "${CMAKE_SOURCE_DIR}/../x86_64/lib/spirv-tools" )
        set (SPIRV_TOOLS_OPT_SEARCH_PATH "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}/External/spirv-tools/source/opt" "${CMAKE_SOURCE_DIR}/../x86_64/lib/spirv-tools" )
    endif()
endif()

find_program(GLSLANG_VALIDATOR NAMES glslangValidator
             HINTS "${EXTERNAL_SOURCE_ROOT}/glslang/${BUILDTGT_DIR}/install/bin"
                   "${GLSLANG_BINARY_ROOT}/StandAlone"
                   "${PROJECT_SOURCE_DIR}/external/${BINDATA_DIR}")

find_path(GLSLANG_SPIRV_INCLUDE_DIR SPIRV/spirv.hpp HINTS "${EXTERNAL_SOURCE_ROOT}/glslang"
                                                    "${CMAKE_SOURCE_DIR}/../glslang"
                                              DOC "Path to SPIRV/spirv.hpp")

find_path(SPIRV_TOOLS_INCLUDE_DIR spirv-tools/libspirv.h HINTS "${EXTERNAL_SOURCE_ROOT}/glslang/External/spirv-tools/include"
                                                   "${CMAKE_SOURCE_DIR}/../glslang/External/spirv-tools/include"
                                             DOC "Path to spirv-tools/libspirv.h")

find_library(GLSLANG_LIB NAMES glslang
             HINTS ${GLSLANG_SEARCH_PATH} )

find_library(OGLCompiler_LIB NAMES OGLCompiler
             HINTS ${GLSLANG_SEARCH_PATH} )

find_library(OSDependent_LIB NAMES OSDependent
             HINTS ${GLSLANG_SEARCH_PATH} )

find_library(HLSL_LIB NAMES HLSL
             HINTS ${GLSLANG_SEARCH_PATH} )

find_library(SPIRV_LIB NAMES SPIRV
             HINTS ${GLSLANG_SEARCH_PATH} )

find_library(SPIRV_REMAPPER_LIB NAMES SPVRemapper
             HINTS ${GLSLANG_SEARCH_PATH} )

find_library(SPIRV_TOOLS_LIB NAMES SPIRV-Tools
             HINTS ${SPIRV_TOOLS_SEARCH_PATH} )

find_library(SPIRV_TOOLS_OPT_LIB NAMES SPIRV-Tools-opt
             HINTS ${SPIRV_TOOLS_OPT_SEARCH_PATH} )

if (WIN32)
    add_library(glslang         STATIC IMPORTED)
    add_library(OGLCompiler     STATIC IMPORTED)
    add_library(OSDependent     STATIC IMPORTED)
    add_library(HLSL            STATIC IMPORTED)
    add_library(SPIRV           STATIC IMPORTED)
    add_library(SPVRemapper     STATIC IMPORTED)
    add_library(Loader          STATIC IMPORTED)
    add_library(SPIRV-Tools-opt STATIC IMPORTED)
    add_library(SPIRV-Tools     STATIC IMPORTED)

    find_library(GLSLANG_DLIB NAMES glslangd
                 HINTS ${GLSLANG_DEBUG_SEARCH_PATH} )
    find_library(OGLCompiler_DLIB NAMES OGLCompilerd
                 HINTS ${GLSLANG_DEBUG_SEARCH_PATH} )
    find_library(OSDependent_DLIB NAMES OSDependentd
                 HINTS ${GLSLANG_DEBUG_SEARCH_PATH} )
    find_library(HLSL_DLIB NAMES HLSLd
                 HINTS ${GLSLANG_DEBUG_SEARCH_PATH} )
    find_library(SPIRV_DLIB NAMES SPIRVd
                 HINTS ${GLSLANG_DEBUG_SEARCH_PATH} )
    find_library(SPIRV_REMAPPER_DLIB NAMES SPVRemapperd
                 HINTS ${GLSLANG_DEBUG_SEARCH_PATH} )
    find_library(SPIRV_TOOLS_DLIB NAMES SPIRV-Toolsd
                 HINTS ${SPIRV_TOOLS_DEBUG_SEARCH_PATH} )
    find_library(SPIRV_TOOLS_OPT_DLIB NAMES SPIRV-Tools-optd
                 HINTS ${SPIRV_TOOLS_OPT_DEBUG_SEARCH_PATH} )

    set_target_properties(glslang PROPERTIES
                         IMPORTED_LOCATION       "${GLSLANG_LIB}"
                         IMPORTED_LOCATION_DEBUG "${GLSLANG_DLIB}")
    set_target_properties(OGLCompiler PROPERTIES
                         IMPORTED_LOCATION       "${OGLCompiler_LIB}"
                         IMPORTED_LOCATION_DEBUG "${OGLCompiler_DLIB}")
    set_target_properties(OSDependent PROPERTIES
                         IMPORTED_LOCATION       "${OSDependent_LIB}"
                         IMPORTED_LOCATION_DEBUG "${OSDependent_DLIB}")
    set_target_properties(HLSL PROPERTIES
                         IMPORTED_LOCATION       "${HLSL_LIB}"
                         IMPORTED_LOCATION_DEBUG "${HLSL_DLIB}")
    set_target_properties(SPIRV PROPERTIES
                         IMPORTED_LOCATION       "${SPIRV_LIB}"
                         IMPORTED_LOCATION_DEBUG "${SPIRV_DLIB}")
    set_target_properties(SPVRemapper PROPERTIES
                         IMPORTED_LOCATION       "${SPIRV_REMAPPER_LIB}"
                         IMPORTED_LOCATION_DEBUG "${SPIRV_REMAPPER_DLIB}")
    set_target_properties(SPIRV-Tools PROPERTIES
                         IMPORTED_LOCATION       "${SPIRV_TOOLS_LIB}"
                         IMPORTED_LOCATION_DEBUG "${SPIRV_TOOLS_DLIB}")
    set_target_properties(SPIRV-Tools-opt PROPERTIES
                         IMPORTED_LOCATION       "${SPIRV_TOOLS_OPT_LIB}"
                         IMPORTED_LOCATION_DEBUG "${SPIRV_TOOLS_OPT_DLIB}")

    set (SPIRV_TOOLS_LIBRARIES SPIRV-Tools-opt SPIRV-Tools)
    set (GLSLANG_LIBRARIES glslang OGLCompiler OSDependent HLSL SPIRV SPVRemapper ${SPIRV_TOOLS_LIBRARIES})
else ()
    # not WIN32
    set (SPIRV_TOOLS_LIBRARIES ${SPIRV_TOOLS_OPT_LIB} ${SPIRV_TOOLS_LIB})
    set (GLSLANG_LIBRARIES ${GLSLANG_LIB} ${OGLCompiler_LIB} ${OSDependent_LIB} ${HLSL_LIB} ${SPIRV_LIB} ${SPIRV_REMAPPER_LIB} ${SPIRV_TOOLS_LIBRARIES})
endif()

set (PYTHON_CMD ${PYTHON_EXECUTABLE})

# Define macro used for building vkxml generated files
macro(run_vk_xml_generate dependency output)
    add_custom_command(OUTPUT ${output}
    COMMAND ${PYTHON_CMD} ${SCRIPTS_DIR}/lvl_genvk.py -registry ${SCRIPTS_DIR}/vk.xml ${output}
    DEPENDS ${SCRIPTS_DIR}/vk.xml ${SCRIPTS_DIR}/generator.py ${SCRIPTS_DIR}/${dependency} ${SCRIPTS_DIR}/lvl_genvk.py ${SCRIPTS_DIR}/reg.py
    )
endmacro()

# Define macro used for generating header files containing commit IDs for external dependencies
macro(run_external_revision_generate source_dir symbol_name output)
    add_custom_command(OUTPUT ${output}
    # NOTE: If you modify this call to use --rev_file instead of --git_dir (to read the commit ID from a file instead of
    # parsing from a Git repository), you probably also want to add the revision file to the list of DEPENDS on the
    # subsequent line (to ensure that the script is re-run when the revision file is modified).
    COMMAND ${PYTHON_CMD} ${SCRIPTS_DIR}/external_revision_generator.py --git_dir ${source_dir} -s ${symbol_name} -o ${output}
    DEPENDS ${SCRIPTS_DIR}/external_revision_generator.py ${source_dir}/.git/HEAD ${source_dir}/.git/index
    )
endmacro()

# Custom target for generated vulkan helper file dependencies
add_custom_target(generate_helper_files DEPENDS
    vk_enum_string_helper.h
    vk_struct_size_helper.h
    vk_struct_size_helper.c
    vk_safe_struct.h
    vk_safe_struct.cpp
    vk_object_types.h
    vk_layer_dispatch_table.h
    vk_dispatch_table_helper.h
    vk_extension_helper.h
    vk_typemap_helper.h
    spirv_tools_commit_id.h
    )
set_target_properties(generate_helper_files PROPERTIES FOLDER ${LVL_TARGET_FOLDER})

# Rules to build generated helper files
run_vk_xml_generate(loader_extension_generator.py vk_layer_dispatch_table.h)
run_vk_xml_generate(dispatch_table_helper_generator.py vk_dispatch_table_helper.h)
run_vk_xml_generate(helper_file_generator.py vk_safe_struct.h)
run_vk_xml_generate(helper_file_generator.py vk_safe_struct.cpp)
run_vk_xml_generate(helper_file_generator.py vk_struct_size_helper.h)
run_vk_xml_generate(helper_file_generator.py vk_struct_size_helper.c)
run_vk_xml_generate(helper_file_generator.py vk_enum_string_helper.h)
run_vk_xml_generate(helper_file_generator.py vk_object_types.h)
run_vk_xml_generate(helper_file_generator.py vk_extension_helper.h)
run_vk_xml_generate(helper_file_generator.py vk_typemap_helper.h)
run_external_revision_generate(${EXTERNAL_SOURCE_ROOT}/glslang/External/spirv-tools SPIRV_TOOLS_COMMIT_ID spirv_tools_commit_id.h)



if(NOT WIN32)
    include(GNUInstallDirs)

    add_definitions(-DFALLBACK_CONFIG_DIRS="${FALLBACK_CONFIG_DIRS}")
    add_definitions(-DFALLBACK_DATA_DIRS="${FALLBACK_DATA_DIRS}")
    add_definitions(-DSYSCONFDIR="${CMAKE_INSTALL_FULL_SYSCONFDIR}")

    # Make sure /etc is searched by the loader
    if(NOT (CMAKE_INSTALL_FULL_SYSCONFDIR STREQUAL "/etc"))
        add_definitions(-DEXTRASYSCONFDIR="/etc")
    endif()
endif()

if(UNIX)
    if(INSTALL_LVL_FILES)
        install(DIRECTORY "${PROJECT_SOURCE_DIR}/include/vulkan" DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
        install(FILES "${CMAKE_CURRENT_BINARY_DIR}/vk_layer_dispatch_table.h" DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/vulkan")
    endif()

# uninstall target
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif()

# Layer Utils Library
# For Windows, we use a static lib because the Windows loader has a fairly restrictive loader search
# path that can't be easily modified to point it to the same directory that contains the layers.
set(VKLAYER_UTILS_VLF_SOURCES
    layers/vk_layer_config.cpp
    layers/vk_layer_extension_utils.cpp
    layers/vk_layer_utils.cpp
    layers/vk_format_utils.cpp
    )
if (WIN32)
    add_library(VkLayer_utils STATIC ${VKLAYER_UTILS_VLF_SOURCES})
    target_compile_definitions(VkLayer_utils PUBLIC _CRT_SECURE_NO_WARNINGS)
else()
    add_library(VkLayer_utils SHARED ${VKLAYER_UTILS_VLF_SOURCES})
    if(INSTALL_LVL_FILES)
        install(TARGETS VkLayer_utils DESTINATION ${CMAKE_INSTALL_LIBDIR})
    endif()
endif()
add_dependencies(VkLayer_utils generate_helper_files)
target_include_directories(
    VkLayer_utils PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/loader
    ${CMAKE_CURRENT_SOURCE_DIR}/include/vulkan
    ${CMAKE_CURRENT_BINARY_DIR}
    )

add_definitions(-DAPI_NAME="${API_NAME}")

# loader: Generic VULKAN ICD loader
# tests: VULKAN tests
if(BUILD_LOADER)
    add_subdirectory(loader)
endif()

if(BUILD_TESTS)
    add_subdirectory(tests)
endif()

if(BUILD_LAYERS)
    add_subdirectory(layers)
endif()

if(BUILD_DEMOS)
    add_subdirectory(demos)
endif()

if(BUILD_VKJSON)
    add_subdirectory(libs/vkjson)
endif()

set (UTILS_NAME vsamputils)

if (CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID MATCHES "Clang")
    set(COMMON_COMPILE_FLAGS "-Wall -Wextra -Wno-unused-parameter -Wno-missing-field-initializers")
    set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -fno-strict-aliasing -fno-builtin-memcmp")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 ${COMMON_COMPILE_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${COMMON_COMPILE_FLAGS} -std=c++11")
    if (UNIX)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden")
    endif()
endif()

set (GLMINCLUDES "${CMAKE_SOURCE_DIR}/API-Samples/utils")

get_filename_component(GLMINC_PREFIX "${GLMINCLUDES}" ABSOLUTE)
if(NOT EXISTS ${GLMINC_PREFIX})
    message(FATAL_ERROR "Necessary glm headers do not exist: " ${GLMINC_PREFIX})
endif()

add_definitions(-DVULKAN_SAMPLES_BASE_DIR="${CMAKE_CURRENT_SOURCE_DIR}")

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/API-Samples/utils)
if(WIN32)
    include_directories(${CMAKE_SOURCE_DIR}\\..\\Include)
else()
    include_directories("/usr/include/vulkan")
endif()

option(BUILD_API_SAMPLES "Build API Samples " ON)
option(BUILD_SAMPLE_LAYERS "Build Sample Layers " ON)

if((CMAKE_SYSTEM_NAME STREQUAL "Linux") AND (NOT BUILD_WSI_XCB_SUPPORT) AND (NOT BUILD_WSI_WAYLAND_SUPPORT))
    set(BUILD_API_SAMPLES OFF)
    set(BUILD_SAMPLE_LAYERS OFF)
    set(BUILD_UTILITIES OFF)
endif()

if(BUILD_UTILITIES)
    add_subdirectory(Utilities)
endif()

if (BUILD_API_SAMPLES)
    add_subdirectory(API-Samples)
endif()

add_subdirectory(Sample-Programs/Hologram)

if (BUILD_SAMPLE_LAYERS)
    add_subdirectory(Layer-Samples/Overlay)
endif()
if(BUILD_ICD)
    add_subdirectory(icd)
endif()
