cmake_minimum_required(VERSION 3.16)
project(PocketMage_Desktop_Emulator)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Enable debug symbols by default
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Build type" FORCE)
endif()

# Add debug flags
if(CMAKE_BUILD_TYPE MATCHES Debug)
  message(STATUS "Building in Debug mode with debug symbols")
  if(MSVC)
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /Zi /Od /DEBUG:FULL")
    set(CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG} /DEBUG:FULL")
  else()
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -O0")
  endif()
endif()

# Detect Emscripten
if(DEFINED ENV{EMSCRIPTEN})
  set(USED_EMSCRIPTEN TRUE)
endif()
if(NOT DEFINED USED_EMSCRIPTEN)
  # CMake sets CMAKE_SYSTEM_NAME to Emscripten when using emcmake
  if(CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
    set(USED_EMSCRIPTEN TRUE)
  endif()
endif()

if(USED_EMSCRIPTEN)
  message(STATUS "Configuring build for Emscripten / WebAssembly")
  # Allow memory growth and assertions during development; tune for production
  set(EMSCRIPTEN_COMMON_LINK_FLAGS
    "-sALLOW_MEMORY_GROWTH=1"
    "-sASSERTIONS=1"
    "-sUSE_SDL=2"
    "-sUSE_SDL_TTF=2"
    # Use FS preloading and force filesystem if you plan to use --preload-file
    # "-sFORCE_FILESYSTEM=1"
  )
  # Add an option to let users specify preloaded asset dirs on the cmake command line:
  option(EMBED_ASSETS "Embed data/ and fonts/ into the WASM filesystem via --preload-file" ON)
  if(EMBED_ASSETS)
    # Preload will be appended to CMAKE_EXE_LINKER_FLAGS below once CMAKE_SOURCE_DIR is known
    set(EMSCRIPTEN_PRELOAD_COMMANDS "")
  endif()
endif()

# ---------------------------
# SDL2 / SDL2_ttf discovery
# ---------------------------
set(_sdl_config_ok FALSE)

# Prefer CMake CONFIG packages on Windows (vcpkg or official SDL2 devel zips)
if (WIN32 AND NOT USED_EMSCRIPTEN)
  find_package(SDL2 CONFIG QUIET)       # provides SDL2::SDL2 and (usually) SDL2::SDL2main
  if (TARGET SDL2::SDL2)
    set(_sdl_config_ok TRUE)
    find_package(SDL2_ttf CONFIG REQUIRED)  # provides SDL2_ttf::SDL2_ttf
  endif()
endif()

# Fallback to pkg-config (macOS/Linux; also works for MSYS2/MinGW on Windows)
if (NOT _sdl_config_ok AND NOT USED_EMSCRIPTEN)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(SDL2 REQUIRED sdl2)
  pkg_check_modules(SDL2_TTF REQUIRED SDL2_ttf)

  add_library(SDL2::SDL2 INTERFACE IMPORTED)
  target_include_directories(SDL2::SDL2 INTERFACE ${SDL2_INCLUDE_DIRS})
  target_link_libraries(SDL2::SDL2 INTERFACE ${SDL2_LIBRARIES})

  add_library(SDL2_ttf::SDL2_ttf INTERFACE IMPORTED)
  target_include_directories(SDL2_ttf::SDL2_ttf INTERFACE ${SDL2_TTF_INCLUDE_DIRS})
  target_link_libraries(SDL2_ttf::SDL2_ttf INTERFACE ${SDL2_TTF_LIBRARIES})
endif()

# If building for Emscripten we expect the SDL ports to be provided by emscripten.
# We'll not try to find system SDL packages in that case; instead rely on emcmake/emcc flags.
if(USED_EMSCRIPTEN)
  # Provide dummy imported targets so the rest of the CMake that references SDL2::SDL2 and SDL2_ttf::SDL2_ttf keeps working
  add_library(SDL2::SDL2 INTERFACE IMPORTED)
  add_library(SDL2_ttf::SDL2_ttf INTERFACE IMPORTED)
  # mark as available (no include dirs or link libs set here; emscripten ports supply them at link time)
  set(_sdl_config_ok TRUE)
endif()

# Homebrew helper for macOS 
if(APPLE AND NOT USED_EMSCRIPTEN)
  execute_process(COMMAND brew --prefix OUTPUT_VARIABLE HOMEBREW_PREFIX OUTPUT_STRIP_TRAILING_WHITESPACE)
  list(APPEND CMAKE_PREFIX_PATH "${HOMEBREW_PREFIX}")
  include_directories("${HOMEBREW_PREFIX}/include")
  link_directories("${HOMEBREW_PREFIX}/lib")
endif()

# PocketMage source paths
set(POCKETMAGE_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/../Code/PocketMage_V3)
set(POCKETMAGE_SRC ${POCKETMAGE_ROOT}/src)
set(POCKETMAGE_INCLUDE ${POCKETMAGE_ROOT}/include)

# PocketMage source files (actual app code) - sysFunc.cpp excluded due to updateKeypress conflict
set(POCKETMAGE_SOURCES_MANUAL
    ${POCKETMAGE_SRC}/globals.cpp
    ${POCKETMAGE_SRC}/assets.cpp
    ${POCKETMAGE_SRC}/PocketMageV3.cpp
    # ${POCKETMAGE_SRC}/USB.cpp  # Excluded - requires ESP32-specific headers
    ${POCKETMAGE_SRC}/TXT.cpp
    ${POCKETMAGE_SRC}/FILEWIZ.cpp
    ${POCKETMAGE_SRC}/TASKS.cpp
    ${POCKETMAGE_SRC}/CALENDAR.cpp
    ${POCKETMAGE_SRC}/SETTINGS.cpp
    ${POCKETMAGE_SRC}/HOME.cpp
    ${POCKETMAGE_SRC}/JOURNAL.cpp
    ${POCKETMAGE_SRC}/LEXICON.cpp
    ${POCKETMAGE_SRC}/POKEDEX.cpp
    ${POCKETMAGE_SRC}/PERIODIC.cpp
    ${POCKETMAGE_SRC}/BT.cpp
    ${POCKETMAGE_SRC}/PokedexUI.cpp
    ${POCKETMAGE_SRC}/PocketMageGraphics.cpp
)

# ---------------------------
# Project sources / includes
# ---------------------------
# Base emulator sources (cross-platform)
set(EMULATOR_SOURCES
  src/main_new.cpp
  src/hardware_shim.cpp
  src/oled_service.cpp
  src/text_utils_desktop.cpp
)

# Platform-specific display backend
# For Emscripten use the cross-platform SDL2 backend (desktop_display_sdl2.cpp)
if(USED_EMSCRIPTEN)
  message(STATUS "Using cross-platform display backend for Emscripten")
  list(APPEND EMULATOR_SOURCES src/desktop_display_sdl2.cpp)
else()
  if(WIN32)
    message(STATUS "Building for Windows - using Windows-specific display backend")
    list(APPEND EMULATOR_SOURCES src/desktop_display_sdl2_windows.cpp)
  elseif(UNIX AND NOT APPLE)
    message(STATUS "Building for Linux - using cross-platform display backend")
    list(APPEND EMULATOR_SOURCES src/desktop_display_sdl2.cpp)
  elseif(APPLE)
    message(STATUS "Building for macOS - using cross-platform display backend")
    list(APPEND EMULATOR_SOURCES src/desktop_display_sdl2.cpp)
  else()
    message(STATUS "Building for unknown platform - using cross-platform display backend")
    list(APPEND EMULATOR_SOURCES src/desktop_display_sdl2.cpp)
  endif()
endif()

add_executable(${PROJECT_NAME}
  ${EMULATOR_SOURCES}
  ${POCKETMAGE_SOURCES_MANUAL}
)

target_include_directories(${PROJECT_NAME} PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  ${POCKETMAGE_INCLUDE}
)

# ---------------------------
# Platform-specific definitions & compile options
# ---------------------------
target_compile_definitions(${PROJECT_NAME} PRIVATE DESKTOP_EMULATOR)

# Emscripten-specific options
if(USED_EMSCRIPTEN)
  # Prevent attempts to use native SDL2::SDL2main or other platform-specific main handling
  target_compile_definitions(${PROJECT_NAME} PRIVATE SDL_MAIN_HANDLED)
  # Recommended compile options: tune for size/performance as needed
  target_compile_options(${PROJECT_NAME} PRIVATE "-O2")
  # Linker flags passed via target_link_options (CMake 3.13+). Convert list to a single string
  foreach(_flag IN LISTS EMSCRIPTEN_COMMON_LINK_FLAGS)
    # prefix flags for the linker
    target_link_options(${PROJECT_NAME} PRIVATE "${_flag}")
  endforeach()
  # If embedding assets, add preload-file flags (use absolute paths for source dirs)
  if(EMBED_ASSETS)
    # Preload data and fonts directory into /data and /fonts respectively
    # Note: CMAKE_SOURCE_DIR here is the top-level source; if you're invoking cmake from repo root this should be correct.
    # Use generator expressions only for definition expansion at configure time.
    set(DATA_DIR "${CMAKE_CURRENT_SOURCE_DIR}/data")
    set(FONTS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/fonts")
    if(EXISTS "${DATA_DIR}")
      target_link_options(${PROJECT_NAME} PRIVATE "--preload-file" "${DATA_DIR}@/data")
    endif()
    if(EXISTS "${FONTS_DIR}")
      target_link_options(${PROJECT_NAME} PRIVATE "--preload-file" "${FONTS_DIR}@/fonts")
    endif()
    # Force FS if you need immediate file access without async fetch
    # target_link_options(${PROJECT_NAME} PRIVATE "-sFORCE_FILESYSTEM=1")
  endif()
else()
  # Windows-specific settings
  if (WIN32)
    message(STATUS "Applying Windows-specific build settings")
    target_compile_definitions(${PROJECT_NAME} PRIVATE SDL_MAIN_HANDLED)
    if (MSVC)
      target_compile_definitions(${PROJECT_NAME} PRIVATE _CRT_SECURE_NO_WARNINGS NOMINMAX)
      # Enable Windows-specific compile options
      target_compile_options(${PROJECT_NAME} PRIVATE /W3)
      # Generate PDB files for debugging
      if(CMAKE_BUILD_TYPE MATCHES Debug)
        target_compile_options(${PROJECT_NAME} PRIVATE /Zi)
        target_link_options(${PROJECT_NAME} PRIVATE /DEBUG:FULL /INCREMENTAL:NO)
      endif()
    else()
      # MinGW or other compilers on Windows
      target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra)
      if(CMAKE_BUILD_TYPE MATCHES Debug)
        target_compile_options(${PROJECT_NAME} PRIVATE -g -O0)
      endif()
    endif()
  endif()

  # Linux-specific settings
  if (UNIX AND NOT APPLE)
    message(STATUS "Applying Linux-specific build settings")
    target_compile_definitions(${PROJECT_NAME} PRIVATE __LINUX__)
    # Enable Linux-specific optimizations
    target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra -O2)
  endif()

  # macOS-specific settings
  if (APPLE)
    message(STATUS "Applying macOS-specific build settings")
    target_compile_definitions(${PROJECT_NAME} PRIVATE __APPLE__)
    # Enable macOS-specific optimizations
    target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra -O2)
    # macOS-specific frameworks if needed
    # target_link_libraries(${PROJECT_NAME} PRIVATE "-framework Cocoa")
  endif()
endif()

# ---------------------------
# Link libraries
# ---------------------------
set(_sdl_targets SDL2::SDL2 SDL2_ttf::SDL2_ttf)
if (TARGET SDL2::SDL2main)
  # Link SDL2main if it's available (fixes WinMain/SDL_main mismatch on Windows)
  list(INSERT _sdl_targets 0 SDL2::SDL2main)
endif()

# For Emscripten, linking is handled by emcc; the SDL2 ports are provided by emscripten flags above.
if(USED_EMSCRIPTEN)
  # Create an empty link to keep the rest of the logic consistent
  target_link_libraries(${PROJECT_NAME} PRIVATE ${_sdl_targets})
else()
  target_link_libraries(${PROJECT_NAME} PRIVATE ${_sdl_targets})
endif()

# ---------------------------
# Post-build: copy assets & platform-specific files
# ---------------------------
if(NOT USED_EMSCRIPTEN)
  # Copy data directory (if it exists)
  if(EXISTS ${CMAKE_SOURCE_DIR}/data)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy_directory
              ${CMAKE_SOURCE_DIR}/data $<TARGET_FILE_DIR:${PROJECT_NAME}>/data
      COMMENT "Copying data directory"
      VERBATIM
    )
  endif()

  # Copy fonts directory (cross-platform)
  if(EXISTS ${CMAKE_SOURCE_DIR}/fonts)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy_directory
              ${CMAKE_SOURCE_DIR}/fonts $<TARGET_FILE_DIR:${PROJECT_NAME}>/fonts
      COMMENT "Copying fonts directory"
      VERBATIM
    )
  endif()
endif()

# Platform-specific post-build steps for native platforms
if (NOT USED_EMSCRIPTEN)
  if (WIN32)
    message(STATUS "Setting up Windows post-build steps")
    # Copy SDL2/SDL2_ttf DLLs next to the .exe on Windows
    foreach(tgt IN ITEMS SDL2::SDL2 SDL2_ttf::SDL2_ttf)
      if (TARGET ${tgt})
        add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
          COMMAND ${CMAKE_COMMAND} -E copy_if_different
                  $<TARGET_FILE:${tgt}> $<TARGET_FILE_DIR:${PROJECT_NAME}>
          COMMENT "Copying ${tgt} DLL"
        )
      endif()
    endforeach()
  elseif(UNIX AND NOT APPLE)
    message(STATUS "Setting up Linux post-build steps")
    # Linux-specific post-build steps (if any)
  elseif(APPLE)
    message(STATUS "Setting up macOS post-build steps")
    # macOS-specific post-build steps (if any)
  endif()
endif()
