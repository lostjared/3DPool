cmake_minimum_required(VERSION 3.18)
project(3DPool VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(MINGW)
    set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
    set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
    add_compile_options(-Wno-narrowing)
    add_link_options(-mwindows)
endif()

if(APPLE)
    option(MOLTEN "Use MoltenVK instead of the Vulkan loader (macOS)" ON)
else()
    option(MOLTEN "Use MoltenVK instead of the Vulkan loader (macOS)" OFF)
endif()

# Homebrew prefix handling (Apple Silicon is usually /opt/homebrew, Intel is /usr/local)
if(APPLE AND NOT CMAKE_CROSSCOMPILING)
    execute_process(
        COMMAND brew --prefix
        OUTPUT_VARIABLE BREW_PREFIX
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
    if(BREW_PREFIX AND EXISTS "${BREW_PREFIX}")
        list(PREPEND CMAKE_PREFIX_PATH "${BREW_PREFIX}")
    endif()
endif()

# Keep /usr/local as a fallback prefix (Intel Homebrew, custom installs)
list(APPEND CMAKE_PREFIX_PATH "/usr/local")

find_package(SDL2        REQUIRED)
find_package(SDL2_ttf    REQUIRED)
find_package(SDL2_mixer  REQUIRED)
find_package(libmx2      REQUIRED)

# Only require Vulkan SDK via find_package when we are NOT using MoltenVK
if(NOT MINGW AND NOT MOLTEN)
    find_package(Vulkan REQUIRED)
endif()

if(MOLTEN)
    if(NOT MOLTEN_PATH)
        if(NOT CMAKE_CROSSCOMPILING)
            execute_process(
                COMMAND brew --prefix molten-vk
                OUTPUT_VARIABLE BREW_MOLTEN_PATH
                OUTPUT_STRIP_TRAILING_WHITESPACE
                ERROR_QUIET
            )
            if(EXISTS "${BREW_MOLTEN_PATH}")
                set(MOLTEN_PATH "${BREW_MOLTEN_PATH}")
            endif()
        endif()

        if(NOT MOLTEN_PATH AND DEFINED ENV{VULKAN_SDK} AND EXISTS "$ENV{VULKAN_SDK}")
            set(MOLTEN_PATH "$ENV{VULKAN_SDK}")
        endif()
    endif()

    if(NOT MOLTEN_PATH OR NOT EXISTS "${MOLTEN_PATH}")
        message(FATAL_ERROR
            "MoltenVK not found.\n"
            "  Install:   brew install molten-vk\n"
            "  Or set:    cmake -DMOLTEN_PATH=/path/to/molten-vk ..\n"
            "  Or use the Vulkan loader: cmake -DMOLTEN=OFF .."
        )
    endif()

    # Vulkan headers are often NOT inside the molten-vk keg on Homebrew.
    # They are typically in the main prefix include via vulkan-headers.
    if(NOT DEFINED BREW_PREFIX AND NOT CMAKE_CROSSCOMPILING)
        execute_process(
            COMMAND brew --prefix
            OUTPUT_VARIABLE BREW_PREFIX
            OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET
        )
    endif()

    find_path(MOLTEN_INCLUDE_DIR vulkan/vulkan.h
        HINTS
            "${MOLTEN_PATH}/include"
            "${MOLTEN_PATH}/include/MoltenVK"
            "${BREW_PREFIX}/include"
            "/opt/homebrew/include"
            "/usr/local/include"
    )

    if(NOT MOLTEN_INCLUDE_DIR)
        message(FATAL_ERROR
            "MoltenVK Vulkan headers not found.\n"
            "  Wanted: vulkan/vulkan.h\n"
            "  Tried:  ${MOLTEN_PATH}/include and Homebrew include.\n"
            "  Fix:    brew install vulkan-headers\n"
            "  Or set: cmake -DMOLTEN_PATH=/path/to/VulkanSDK .."
        )
    endif()

    find_library(MOLTEN_LIBRARY NAMES MoltenVK
        HINTS
            "${MOLTEN_PATH}/lib"
            "${BREW_PREFIX}/lib"
            "/opt/homebrew/lib"
            "/usr/local/lib"
    )

    if(NOT MOLTEN_LIBRARY)
        message(FATAL_ERROR
            "libMoltenVK not found.\n"
            "  Tried: ${MOLTEN_PATH}/lib and Homebrew lib.\n"
            "  Fix:   brew install molten-vk"
        )
    endif()

    get_filename_component(MOLTEN_LIB_DIR "${MOLTEN_LIBRARY}" DIRECTORY)

    set(MOLTEN_FRAMEWORKS
        "-framework QuartzCore"
        "-framework Metal"
        "-framework IOSurface"
        "-framework CoreGraphics"
        "-framework Foundation"
        "-framework Cocoa"
        "-framework IOKit"
    )

    message(STATUS "MoltenVK library : ${MOLTEN_LIBRARY}")
    message(STATUS "MoltenVK include : ${MOLTEN_INCLUDE_DIR}")
endif()

if(TARGET libmx2::mx)
    get_target_property(_mx2_iface_dirs libmx2::mx INTERFACE_INCLUDE_DIRECTORIES)
    set(_mx2_inc "")
    foreach(_d ${_mx2_iface_dirs})
        if(EXISTS "${_d}/mx.hpp")
            set(_mx2_inc "${_d}")
            break()
        endif()
    endforeach()
    if(NOT _mx2_inc AND _mx2_iface_dirs)
        list(GET _mx2_iface_dirs 0 _mx2_inc)
    endif()
else()
    set(_mx2_inc "${CMAKE_INSTALL_PREFIX}/include/mx2")
endif()

set(_check_pass TRUE)

set(_mx2_headers vk.hpp vk_model.hpp loadpng.hpp argz.hpp mx.hpp)
foreach(_hdr ${_mx2_headers})
    if(EXISTS "${_mx2_inc}/${_hdr}")
        message(STATUS "  [OK]   ${_hdr}")
    else()
        message(WARNING "  [MISS] ${_hdr}")
        set(_check_pass FALSE)
    endif()
endforeach()

if(EXISTS "${_mx2_inc}/volk/volk.h")
    message(STATUS "  [OK]   volk/volk.h")
else()
    set(_volk_found FALSE)
    if(TARGET libmx2::volk)
        get_target_property(_volk_dirs libmx2::volk INTERFACE_INCLUDE_DIRECTORIES)
        foreach(_d ${_volk_dirs})
            if(EXISTS "${_d}/volk.h")
                message(STATUS "  [OK]   volk.h  (${_d})")
                set(_volk_found TRUE)
                break()
            endif()
        endforeach()
    endif()
    if(NOT _volk_found)
        message(WARNING "  [MISS] volk/volk.h")
        set(_check_pass FALSE)
    endif()
endif()

find_path(_sdl2_check SDL.h
    HINTS ${SDL2_INCLUDE_DIRS}
    PATH_SUFFIXES SDL2
)
if(_sdl2_check)
    message(STATUS "  [OK]   SDL.h  (${_sdl2_check})")
else()
    message(WARNING "  [MISS] SDL.h")
    set(_check_pass FALSE)
endif()

set(_sdl2_headers SDL_vulkan.h SDL_ttf.h SDL_mixer.h)
foreach(_hdr ${_sdl2_headers})
    find_path(_sdl2_${_hdr}_dir ${_hdr}
        HINTS ${SDL2_INCLUDE_DIRS}
        PATH_SUFFIXES SDL2
    )
    if(_sdl2_${_hdr}_dir)
        message(STATUS "  [OK]   ${_hdr}")
    else()
        message(WARNING "  [MISS] ${_hdr}")
        set(_check_pass FALSE)
    endif()
    unset(_sdl2_${_hdr}_dir CACHE)
endforeach()

find_path(_glm_dir glm/glm.hpp)
if(_glm_dir)
    message(STATUS "  [OK]   glm/glm.hpp  (${_glm_dir})")
else()
    message(WARNING "  [MISS] glm/glm.hpp")
    set(_check_pass FALSE)
endif()
unset(_glm_dir CACHE)

find_path(_glm_transform_dir glm/gtc/matrix_transform.hpp)
if(_glm_transform_dir)
    message(STATUS "  [OK]   glm/gtc/matrix_transform.hpp")
else()
    message(WARNING "  [MISS] glm/gtc/matrix_transform.hpp")
    set(_check_pass FALSE)
endif()
unset(_glm_transform_dir CACHE)

if(MINGW)
    message(STATUS "  [SKIP] MINGW build - Vulkan loaded at runtime via volk")
elseif(MOLTEN)
    message(STATUS "  [OK]   MoltenVK build - Vulkan headers from ${MOLTEN_INCLUDE_DIR}")
elseif(Vulkan_FOUND)
    message(STATUS "  [OK]   Vulkan ${Vulkan_VERSION}  (${Vulkan_LIBRARIES})")
else()
    message(WARNING "  [MISS] Vulkan SDK not found")
    set(_check_pass FALSE)
endif()

set(_mx2_lib_targets libmx2::mx libmx2::mxvk)
if(NOT MINGW AND NOT MOLTEN)
    list(APPEND _mx2_lib_targets libmx2::volk)
endif()

foreach(_tgt ${_mx2_lib_targets})
    if(TARGET ${_tgt})
        get_target_property(_loc ${_tgt} IMPORTED_LOCATION)
        if(NOT _loc)
            get_target_property(_loc ${_tgt} IMPORTED_LOCATION_RELEASE)
        endif()
        if(NOT _loc)
            get_target_property(_loc ${_tgt} IMPORTED_LOCATION_DEBUG)
        endif()
        if(_loc AND EXISTS "${_loc}")
            get_filename_component(_name "${_loc}" NAME)
            message(STATUS "  [OK]   ${_name}  (${_loc})")
        elseif(_loc)
            get_filename_component(_name "${_loc}" NAME)
            message(WARNING "  [MISS] ${_name}  (expected at ${_loc})")
            set(_check_pass FALSE)
        else()
            message(STATUS "  [OK]   ${_tgt}  (target exists, location deferred)")
        endif()
    else()
        message(WARNING "  [MISS] ${_tgt} target not found")
        set(_check_pass FALSE)
    endif()
endforeach()

set(_sdl2_required_targets SDL2::SDL2 SDL2_ttf::SDL2_ttf SDL2_mixer::SDL2_mixer)
if(MINGW)
    list(APPEND _sdl2_required_targets SDL2::SDL2main)
endif()

foreach(_tgt ${_sdl2_required_targets})
    if(TARGET ${_tgt})
        message(STATUS "  [OK]   ${_tgt}")
    else()
        message(WARNING "  [MISS] ${_tgt}")
        set(_check_pass FALSE)
    endif()
endforeach()

message(STATUS "")
message(STATUS "[Source] Checking source files ...")
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/main.cpp")
    message(STATUS "  [OK]   main.cpp")
else()
    message(WARNING "  [MISS] main.cpp")
    set(_check_pass FALSE)
endif()

message(STATUS "")
message(STATUS "[Assets] Checking data directory ...")
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/data")
    file(GLOB _data_files "${CMAKE_CURRENT_SOURCE_DIR}/data/*")
    list(LENGTH _data_files _data_count)
    message(STATUS "  [OK]   data/  (${_data_count} files)")
else()
    message(WARNING "  [MISS] data/ directory not found")
endif()

set(_required_assets
    data/geosphere.mxmod.z
    data/pocket.mxmod.z
    data/cube.mxmod.z
    data/pooltable_felt.mxmod.z
    data/pooltable_wood.mxmod.z
    data/pooltable_pocket.mxmod.z
    data/background.png
    data/start.png
    data/scores.png
    data/logo.png
    data/sprite_vert.spv
    data/sprite_frag.spv
    data/bend_dir.spv
    data/vert.spv
    data/frag.spv
    data/text_vert.spv
    data/text_frag.spv
    font.ttf
)

foreach(_asset ${_required_assets})
    if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${_asset}")
        message(WARNING "  [MISS] ${_asset}")
    endif()
endforeach()

message(STATUS "")
if(_check_pass)
    message(STATUS "=========== All checks PASSED ===================")
else()
    message(FATAL_ERROR "One or more required headers/libraries are missing. See above.")
endif()
message(STATUS "")

# Use a safe CMake target name, keep output binary/app name as 3DPool
if(MINGW)
    set(WIN_RC "${CMAKE_CURRENT_SOURCE_DIR}/ball.rc")
    add_executable(Pool3D WIN32 main.cpp ${WIN_RC})
else()
    add_executable(Pool3D main.cpp)
endif()

set_target_properties(Pool3D PROPERTIES
    OUTPUT_NAME "3DPool"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}"
)

if(MINGW)
    target_link_libraries(Pool3D PRIVATE
        SDL2::SDL2main
        libmx2::mx
        libmx2::mxvk
        SDL2::SDL2
        SDL2_ttf::SDL2_ttf
        SDL2_mixer::SDL2_mixer
    )
elseif(MOLTEN)
    target_compile_definitions(Pool3D PRIVATE WITH_MOLTEN)
    target_link_directories(Pool3D PRIVATE ${MOLTEN_LIB_DIR})
    target_include_directories(Pool3D PRIVATE ${MOLTEN_INCLUDE_DIR})
    target_link_libraries(Pool3D PRIVATE
        libmx2::mx
        libmx2::mxvk
        SDL2::SDL2
        SDL2_ttf::SDL2_ttf
        SDL2_mixer::SDL2_mixer
        ${MOLTEN_LIBRARY}
        ${MOLTEN_FRAMEWORKS}
    )
else()
    target_link_libraries(Pool3D PRIVATE
        libmx2::mx
        libmx2::mxvk
        libmx2::volk
        SDL2::SDL2
        SDL2_ttf::SDL2_ttf
        SDL2_mixer::SDL2_mixer
        Vulkan::Vulkan
        ${CMAKE_DL_LIBS}
    )
endif()

add_custom_command(TARGET Pool3D POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${CMAKE_CURRENT_SOURCE_DIR}/data"
        "$<TARGET_FILE_DIR:Pool3D>/data"
    COMMENT "Copying data/ assets to build directory"
)

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/font.ttf")
    add_custom_command(TARGET Pool3D POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${CMAKE_CURRENT_SOURCE_DIR}/font.ttf"
            "$<TARGET_FILE_DIR:Pool3D>/font.ttf"
        COMMENT "Copying font.ttf to build directory"
    )
endif()

install(TARGETS Pool3D RUNTIME DESTINATION bin)
install(DIRECTORY data/ DESTINATION share/3DPool/data)
install(FILES font.ttf DESTINATION share/3DPool)

message(STATUS "----------- 3DPool build summary -----------")
message(STATUS "  C++ standard : C++${CMAKE_CXX_STANDARD}")
message(STATUS "  Compiler     : ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "  SDL2         : ${SDL2_VERSION}")
message(STATUS "  SDL2_ttf     : found")
message(STATUS "  SDL2_mixer   : found")
if(MOLTEN)
    message(STATUS "  Vulkan       : MoltenVK (${MOLTEN_LIBRARY})")
elseif(NOT MINGW)
    message(STATUS "  Vulkan       : ${Vulkan_VERSION}")
endif()
message(STATUS "  libmx2       : found")
message(STATUS "  Build type   : ${CMAKE_BUILD_TYPE}")
message(STATUS "----------------------------------------------")