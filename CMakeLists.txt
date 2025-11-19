cmake_minimum_required(VERSION 3.22)

project(Winni VERSION 0.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# If single-config generator and no build type was set, default to Debug
if(NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# Sources
set(ENGINE_SOURCES
    src/Core/main.cpp
)

add_library(${PROJECT_NAME} STATIC
    ${ENGINE_SOURCES}
)

# Public include dirs for consumers of the library
target_include_directories(${PROJECT_NAME}
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${CMAKE_CURRENT_SOURCE_DIR}/src
        ${CMAKE_CURRENT_SOURCE_DIR}/external
    PRIVATE
        src
)

# Compiler warnings / flags (per-target, portable)
target_compile_options(${PROJECT_NAME}
    PRIVATE
        $<$<CXX_COMPILER_ID:MSVC>:/W4;/permissive->
        $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic>
)

# Compile-time definitions per-configuration
target_compile_definitions(${PROJECT_NAME}
    PUBLIC
        $<$<CONFIG:Debug>:DEBUG>
        $<$<CONFIG:Release>:NDEBUG>
)

# Prefer modern find_package targets
find_package(OpenGL REQUIRED)
find_package(glfw3 REQUIRED)   # expects glfw3 package (may provide glfw, glfw3 or GLFW::GLFW)
find_package(glm REQUIRED)

# Link libraries (use imported targets if available)
target_link_libraries(${PROJECT_NAME}
    PUBLIC
        OpenGL::GL
        glm::glm
        glfw
)

# Optional: freetype / openal via pkg-config if available
find_package(PkgConfig QUIET)
if(PkgConfig_FOUND)
    pkg_check_modules(FREETYPE freetype2 QUIET)
    if(FREETYPE_FOUND)
        target_include_directories(${PROJECT_NAME} PRIVATE ${FREETYPE_INCLUDE_DIRS})
        target_link_libraries(${PROJECT_NAME} PRIVATE ${FREETYPE_LIBRARIES})
    endif()

    pkg_check_modules(OPENAL openal QUIET)
    if(OPENAL_FOUND)
        target_include_directories(${PROJECT_NAME} PRIVATE ${OPENAL_INCLUDE_DIRS})
        target_link_libraries(${PROJECT_NAME} PRIVATE ${OPENAL_LIBRARIES})
    endif()
endif()

# Platform specific libraries
if(WIN32)
    target_link_libraries(${PROJECT_NAME} PRIVATE winmm)
elseif(APPLE)
    find_library(COCOA_LIBRARY Cocoa REQUIRED)
    find_library(IOKIT_LIBRARY IOKit REQUIRED)
    find_library(COREVIDEO_LIBRARY CoreVideo REQUIRED)
    target_link_libraries(${PROJECT_NAME} PRIVATE ${COCOA_LIBRARY} ${IOKIT_LIBRARY} ${COREVIDEO_LIBRARY})
elseif(UNIX)
    target_link_libraries(${PROJECT_NAME} PRIVATE dl pthread)
endif()

# Install targets
install(TARGETS ${PROJECT_NAME}
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)

option(BUILD_EXAMPLES "Build example applications" ON)

if(BUILD_EXAMPLES)
    add_executable(GameEngineExample
        examples/main.cpp
    )

    target_link_libraries(GameEngineExample PRIVATE ${PROJECT_NAME})

    target_include_directories(GameEngineExample PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/examples
    )

    # Copy assets next to the built example executable after build
    add_custom_command(TARGET GameEngineExample POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory
            ${CMAKE_CURRENT_SOURCE_DIR}/assets
            $<TARGET_FILE_DIR:GameEngineExample>/assets
    )
endif()

