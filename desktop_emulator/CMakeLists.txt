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

# ---------------------------
# Detect Emscripten
# ---------------------------
if(DEFINED ENV{EMSCRIPTEN})
  set(USED_EMSCRIPTEN TRUE)
endif()
if(NOT DEFINED USED_EMSCRIPTEN)
  if(CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
    set(USED_EMSCRIPTEN TRUE)
  endif()
endif()

if(USED_EMSCRIPTEN)
  message(STATUS "Configuring build for Emscripten / WebAssembly")
  set(EMSCRIPTEN_COMMON_LINK_FLAGS
    "-sALLOW_MEMORY_GROWTH=1"
    "-sASSERTIONS=1"
    "-sUSE_SDL=2"
    "-sUSE_SDL_TTF=2"
  )
  option(EMBED_ASSETS "Embed data/ and fonts/ into the WASM filesystem via --preload-file" ON)
endif()

# ---------------------------
# SDL2 / SDL2_ttf discovery
# ---------------------------
set(_sdl_config_ok FALSE)

if (WIN32 AND NOT USED_EMSCRIPTEN)
  find_package(SDL2 CONFIG QUIET)
  if (TARGET SDL2::SDL2)
    set(_sdl_config_ok TRUE)
    find_package(SDL2_ttf CONFIG REQUIRED)
  endif()
endif()

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
if(USED_EMSCRIPTEN)
  add_library(SDL2::SDL2 INTERFACE IMPORTED)
  add_library(SDL2_ttf::SDL2_ttf INTERFACE IMPORTED)
  set(_sdl_config_ok TRUE)

  # If EMSDK environment is available (set by emsdk activation in CI),
  # add the Emscripten system include directories so #include <SDL2/SDL.h> resolves.
  if(DEFINED ENV{EMSDK})
    set(EMSDK_ROOT $ENV{EMSDK})
    set(EMSCRIPTEN_SYS_INCLUDE "${EMSDK_ROOT}/upstream/emscripten/system/include")
    if(EXISTS "${EMSCRIPTEN_SYS_INCLUDE}")
      target_include_directories(SDL2::SDL2 INTERFACE "${EMSCRIPTEN_SYS_INCLUDE}")
      target_include_directories(SDL2_ttf::SDL2_ttf INTERFACE "${EMSCRIPTEN_SYS_INCLUDE}")
      message(STATUS "Added Emscripten system include directory: ${EMSCRIPTEN_SYS_INCLUDE}")
      if(EXISTS "${EMSDK_ROOT}/upstream/emscripten/system/local/include")
        target_include_directories(SDL2::SDL2 INTERFACE "${EMSDK_ROOT}/upstream/emscripten/system/local/include")
        target_include_directories(SDL2_ttf::SDL2_ttf INTERFACE "${EMSDK_ROOT}/upstream/emscripten/system/local/include")
        message(STATUS "Added Emscripten local include: ${EMSDK_ROOT}/upstream/emscripten/system/local/include")
      endif()
    endif()
  endif()
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
set(EMULATOR_SOURCES
  src/main_new.cpp
  src/hardware_shim.cpp
  src/oled_service.cpp
  src/text_utils_desktop.cpp
)

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

# If building for Emscripten and EMSDK include dir was discovered, make sure the executable also has it.
if(USED_EMSCRIPTEN AND DEFINED EMSCRIPTEN_SYS_INCLUDE)
  target_include_directories(${PROJECT_NAME} PRIVATE "${EMSCRIPTEN_SYS_INCLUDE}")
  if(DEFINED EMSDK_ROOT)
    target_include_directories(${PROJECT_NAME} PRIVATE "${EMSDK_ROOT}/upstream/emscripten/system/local/include")
  endif()
endif()

# ---------------------------
# Platform-specific definitions & compile options
# ---------------------------
target_compile_definitions(${PROJECT_NAME} PRIVATE DESKTOP_EMULATOR)

if(USED_EMSCRIPTEN)
  target_compile_definitions(${PROJECT_NAME} PRIVATE SDL_MAIN_HANDLED)
  target_compile_options(${PROJECT_NAME} PRIVATE "-O2")
  foreach(_flag IN LISTS EMSCRIPTEN_COMMON_LINK_FLAGS)
    target_link_options(${PROJECT_NAME} PRIVATE "${_flag}")
  endforeach()

  if(EMBED_ASSETS)
    set(DATA_DIR "${CMAKE_CURRENT_SOURCE_DIR}/data")
    set(FONTS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/fonts")
    if(EXISTS "${DATA_DIR}")
      target_link_options(${PROJECT_NAME} PRIVATE "--preload-file" "${DATA_DIR}@/data")
    endif()
    if(EXISTS "${FONTS_DIR}")
      target_link_options(${PROJECT_NAME} PRIVATE "--preload-file" "${FONTS_DIR}@/fonts")
    endif()
  endif()
else()
  if (WIN32)
    message(STATUS "Applying Windows-specific build settings")
    target_compile_definitions(${PROJECT_NAME} PRIVATE SDL_MAIN_HANDLED)
    if (MSVC)
      target_compile_definitions(${PROJECT_NAME} PRIVATE _CRT_SECURE_NO_WARNINGS NOMINMAX)
      target_compile_options(${PROJECT_NAME} PRIVATE /W3)
      if(CMAKE_BUILD_TYPE MATCHES Debug)
        target_compile_options(${PROJECT_NAME} PRIVATE /Zi)
        target_link_options(${PROJECT_NAME} PRIVATE /DEBUG:FULL /INCREMENTAL:NO)
      endif()
    else()
      target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra)
      if(CMAKE_BUILD_TYPE MATCHES Debug)
        target_compile_options(${PROJECT_NAME} PRIVATE -g -O0)
      endif()
    endif()
  endif()

  if (UNIX AND NOT APPLE)
    message(STATUS "Applying Linux-specific build settings")
    target_compile_definitions(${PROJECT_NAME} PRIVATE __LINUX__)
    target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra -O2)
  endif()

  if (APPLE)
    message(STATUS "Applying macOS-specific build settings")
    target_compile_definitions(${PROJECT_NAME} PRIVATE __APPLE__)
    target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra -O2)
  endif()
endif()

# ---------------------------
# Link libraries
# ---------------------------
set(_sdl_targets SDL2::SDL2 SDL2_ttf::SDL2_ttf)
if (TARGET SDL2::SDL2main)
  list(INSERT _sdl_targets 0 SDL2::SDL2main)
endif()
target_link_libraries(${PROJECT_NAME} PRIVATE ${_sdl_targets})

# ---------------------------
# Post-build: copy assets & platform-specific files
# ---------------------------
if(NOT USED_EMSCRIPTEN)
  if(EXISTS ${CMAKE_SOURCE_DIR}/data)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy_directory
              ${CMAKE_SOURCE_DIR}/data $<TARGET_FILE_DIR:${PROJECT_NAME}>/data
      COMMENT "Copying data directory"
      VERBATIM
    )
  endif()

  if(EXISTS ${CMAKE_SOURCE_DIR}/fonts)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy_directory
              ${CMAKE_SOURCE_DIR}/fonts $<TARGET_FILE_DIR:${PROJECT_NAME}>/fonts
      COMMENT "Copying fonts directory"
      VERBATIM
    )
  endif()
endif()

if (NOT USED_EMSCRIPTEN)
  if (WIN32)
    message(STATUS "Setting up Windows post-build steps")
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
  elseif(APPLE)
    message(STATUS "Setting up macOS post-build steps")
  endif()
endif()
