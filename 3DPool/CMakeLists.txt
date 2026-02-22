cmake_minimum_required(VERSION 3.18)
project(3DPool LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)


if(APPLE)
    option(MOLTEN "Use MoltenVK instead of the Vulkan loader (macOS)" ON)
else()
    option(MOLTEN "Use MoltenVK instead of the Vulkan loader (macOS)" OFF)
endif()


find_package(SDL2 REQUIRED)
find_package(SDL2_ttf REQUIRED)
find_package(SDL2_mixer REQUIRED)


if(MOLTEN)
    if(NOT MOLTEN_PATH)
        execute_process(
            COMMAND brew --prefix
            OUTPUT_VARIABLE HOMEBREW_PREFIX
            OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET)
        if(NOT HOMEBREW_PREFIX)
            if(CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "arm64")
                set(HOMEBREW_PREFIX "/opt/homebrew")
            else()
                set(HOMEBREW_PREFIX "/usr/local")
            endif()
        endif()

        execute_process(
            COMMAND brew --prefix molten-vk
            OUTPUT_VARIABLE BREW_MOLTEN_PATH
            OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET)

        if(EXISTS "${BREW_MOLTEN_PATH}")
            set(MOLTEN_PATH "${BREW_MOLTEN_PATH}")
        elseif(EXISTS "${HOMEBREW_PREFIX}/opt/molten-vk")
            set(MOLTEN_PATH "${HOMEBREW_PREFIX}/opt/molten-vk")
        elseif(DEFINED ENV{VULKAN_SDK} AND EXISTS "$ENV{VULKAN_SDK}")
            set(MOLTEN_PATH "$ENV{VULKAN_SDK}")
        endif()
    endif()

    if(NOT MOLTEN_PATH OR NOT EXISTS "${MOLTEN_PATH}")
        message(FATAL_ERROR
            "MoltenVK not found.\n"
            "  Install:   brew install molten-vk\n"
            "  Or set:    cmake -DMOLTEN_PATH=/path/to/molten-vk ..\n"
            "  Or use the Vulkan loader with: cmake -DMOLTEN=OFF ..")
    endif()

    if(EXISTS "${MOLTEN_PATH}/include/vulkan/vulkan.h")
        set(MOLTEN_INCLUDE_DIR "${MOLTEN_PATH}/include")
    elseif(EXISTS "${MOLTEN_PATH}/include/MoltenVK/vulkan/vulkan.h")
        set(MOLTEN_INCLUDE_DIR "${MOLTEN_PATH}/include/MoltenVK")
    elseif(DEFINED HOMEBREW_PREFIX AND EXISTS "${HOMEBREW_PREFIX}/include/vulkan/vulkan.h")
        set(MOLTEN_INCLUDE_DIR "${HOMEBREW_PREFIX}/include")
    else()
        message(FATAL_ERROR
            "MoltenVK vulkan.h not found under ${MOLTEN_PATH}/include.\n"
            "  Try: brew install molten-vk vulkan-headers")
    endif()

    if(EXISTS "${MOLTEN_PATH}/lib/libMoltenVK.dylib")
        set(MOLTEN_LIBRARY "${MOLTEN_PATH}/lib/libMoltenVK.dylib")
        set(MOLTEN_LIB_DIR "${MOLTEN_PATH}/lib")
    elseif(DEFINED HOMEBREW_PREFIX AND EXISTS "${HOMEBREW_PREFIX}/lib/libMoltenVK.dylib")
        set(MOLTEN_LIBRARY "${HOMEBREW_PREFIX}/lib/libMoltenVK.dylib")
        set(MOLTEN_LIB_DIR "${HOMEBREW_PREFIX}/lib")
    else()
        message(FATAL_ERROR
            "libMoltenVK.dylib not found in ${MOLTEN_PATH}/lib.\n"
            "  Try: brew install molten-vk")
    endif()

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
else()
    find_package(Vulkan REQUIRED)
    message(STATUS "Vulkan: ${Vulkan_VERSION}")
endif()

find_package(glm REQUIRED)

find_package(libmx2 REQUIRED
    HINTS /usr/local/lib/cmake/libmx2)

foreach(_target libmx2::mx libmx2::mxvk)
    if(NOT TARGET ${_target})
        message(FATAL_ERROR
            "Required libmx2 target '${_target}' was not found.\n"
            "Make sure libmx2 is installed (headers in /usr/local/include/mx2, "
            "cmake config in /usr/local/lib/cmake/libmx2).")
    endif()
endforeach()

if(NOT MOLTEN AND NOT TARGET libmx2::volk)
    message(FATAL_ERROR
        "Required libmx2 target 'libmx2::volk' was not found.\n"
        "This target is expected when building without MoltenVK.\n"
        "If you are on macOS, try: cmake -DMOLTEN=ON ..")
endif()

set(_mx2_prefix "/usr/local/include/mx2")
foreach(_hdr vk.hpp vk_model.hpp loadpng.hpp argz.hpp mx.hpp)
    if(NOT EXISTS "${_mx2_prefix}/${_hdr}")
        message(FATAL_ERROR
            "Required libmx2 header not found: ${_mx2_prefix}/${_hdr}\n"
            "Please install or rebuild libmx2.")
    endif()
endforeach()

find_path(SDL2_INCLUDE_DIR SDL.h
    HINTS ${SDL2_INCLUDE_DIRS}
    PATH_SUFFIXES SDL2
    REQUIRED)

if(NOT SDL2_INCLUDE_DIR)
    message(FATAL_ERROR "Could not find SDL.h. Install libsdl2-dev.")
endif()

find_path(SDL2_TTF_INCLUDE_DIR SDL_ttf.h
    HINTS ${SDL2_TTF_INCLUDE_DIRS}
    PATH_SUFFIXES SDL2
    REQUIRED)

if(NOT SDL2_TTF_INCLUDE_DIR)
    message(FATAL_ERROR "Could not find SDL_ttf.h. Install libsdl2-ttf-dev.")
endif()

find_path(SDL2_MIXER_INCLUDE_DIR SDL_mixer.h
    HINTS ${SDL2_MIXER_INCLUDE_DIRS}
    PATH_SUFFIXES SDL2
    REQUIRED)

if(NOT SDL2_MIXER_INCLUDE_DIR)
    message(FATAL_ERROR "Could not find SDL_mixer.h. Install libsdl2-mixer-dev.")
endif()

find_path(GLM_INCLUDE_DIR glm/glm.hpp
    HINTS ${GLM_INCLUDE_DIRS} /usr/include /usr/local/include
    REQUIRED)

if(NOT GLM_INCLUDE_DIR)
    message(FATAL_ERROR "Could not find glm/glm.hpp. Install libglm-dev.")
endif()

foreach(_glm_hdr glm/glm.hpp glm/gtc/matrix_transform.hpp)
    if(NOT EXISTS "${GLM_INCLUDE_DIR}/${_glm_hdr}")
        message(FATAL_ERROR
            "Required GLM header not found: ${GLM_INCLUDE_DIR}/${_glm_hdr}\n"
            "Please install or update libglm-dev.")
    endif()
    message(STATUS "Found header: ${GLM_INCLUDE_DIR}/${_glm_hdr}")
endforeach()


if(NOT MOLTEN)
    find_path(VULKAN_INCLUDE_DIR vulkan/vulkan.h
        HINTS ${Vulkan_INCLUDE_DIRS} /usr/include /usr/local/include)
    if(NOT VULKAN_INCLUDE_DIR)
        message(FATAL_ERROR
            "Could not find vulkan/vulkan.h.\n"
            "Install the Vulkan headers: pacman -S vulkan-headers  "
            "or  apt install libvulkan-dev")
    endif()
    message(STATUS "Found header: ${VULKAN_INCLUDE_DIR}/vulkan/vulkan.h")
endif()

macro(require_library _var _name)
    find_library(${_var} NAMES ${_name} ${ARGN})
    if(NOT ${_var})
        string(REPLACE ";" " / " _names_str "${_name};${ARGN}")
        message(FATAL_ERROR
            "Required library '${_names_str}' not found.\n"
            "  ${_var} search failed. Check your library paths or install the "
            "missing package.")
    else()
        message(STATUS "Found library: ${${_var}}")
    endif()
endmacro()


require_library(LIB_MX     mx      HINTS /usr/local/lib)
require_library(LIB_MXVK   mxvk    HINTS /usr/local/lib)


if(NOT MOLTEN)
    require_library(LIB_VOLK   volk    HINTS /usr/local/lib)
endif()


require_library(LIB_SDL2       SDL2)
require_library(LIB_SDL2_TTF   SDL2_ttf)
require_library(LIB_SDL2_MIXER SDL2_mixer)


if(NOT MOLTEN)
    require_library(LIB_VULKAN vulkan)
endif()


require_library(LIB_PNG png)
require_library(LIB_Z   z)


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
        message(WARNING "Expected asset not found: ${CMAKE_CURRENT_SOURCE_DIR}/${_asset}")
    endif()
endforeach()


add_executable(3DPool main.cpp)

target_include_directories(3DPool PRIVATE
    ${GLM_INCLUDE_DIR}
    $<$<BOOL:${MOLTEN}>:${MOLTEN_INCLUDE_DIR}>
)

if(MOLTEN)
    target_compile_definitions(3DPool PRIVATE WITH_MOLTEN)
    target_link_directories(3DPool PRIVATE ${MOLTEN_LIB_DIR})
    target_link_libraries(3DPool PRIVATE
        libmx2::mxvk
        libmx2::mx
        SDL2::SDL2
        glm::glm
        ${MOLTEN_LIBRARY}
        ${MOLTEN_FRAMEWORKS}
    )
else()
    target_link_libraries(3DPool PRIVATE
        libmx2::mxvk
        libmx2::volk
        libmx2::mx
        SDL2::SDL2
        glm::glm
    )
endif()

add_custom_command(TARGET 3DPool POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${CMAKE_CURRENT_SOURCE_DIR}/data"
        "$<TARGET_FILE_DIR:3DPool>/data"
    COMMENT "Copying data/ assets to build directory"
)

add_custom_command(TARGET 3DPool POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${CMAKE_CURRENT_SOURCE_DIR}/font.ttf"
        "$<TARGET_FILE_DIR:3DPool>/font.ttf"
    COMMENT "Copying font.ttf to build directory"
)

install(TARGETS 3DPool RUNTIME DESTINATION bin)
install(DIRECTORY data/ DESTINATION share/3DPool/data)
install(FILES font.ttf DESTINATION share/3DPool)
