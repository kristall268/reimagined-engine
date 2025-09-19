cmake_minimum_required(VERSION 3.22)
project(Winni)


set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)


set(CMAKE_BUILD_TYPE Debug CACHE STRING "Build type")
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")

if(MSVC)
    add_compile_options(/W4 /permissive-)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
else()
    add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# Debug/Release specific settings
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions(-DDEBUG)
    if(NOT MSVC)
        add_compile_options(-g -O0)
    endif()
else()
    add_definitions(-DNDEBUG)
    if(NOT MSVC)
        add_compile_options(-O3)
    endif()
endif()

find_package(OpenGL REQUIRED)
find_package(glfw3 REQUIRED)
find_package(glm REQUIRED)

# Optional packages
find_package(PkgConfig QUIET)
if(PkgConfig_FOUND)
    pkg_check_modules(FREETYPE freetype2)
    pkg_check_modules(OPENAL openal)
endif()

set(INCLUDE_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}/external
)


set(ENGINE_SOURCES
    src/Core/main.cpp
)

add_library(${PROJECT_NAME} STATIC
    ${ENGINE_SOURCES}
    ${ENGINE_HEADERS}
    ${EXTERNAL_SOURCES}
)
# Set target properties
set_target_properties(${PROJECT_NAME} PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    OUTPUT_NAME "gameengine"
)
target_include_directories(${PROJECT_NAME}
    PUBLIC
        ${INCLUDE_DIRS}
    PRIVATE
        src
)

target_link_libraries(${PROJECT_NAME}
    PUBLIC
        OpenGL::GL
        glfw
        glm::glm
)


if(FREETYPE_FOUND)
    target_include_directories(${PROJECT_NAME} PRIVATE ${FREETYPE_INCLUDE_DIRS})
    target_link_libraries(${PROJECT_NAME} PRIVATE ${FREETYPE_LIBRARIES})
endif()

if(OPENAL_FOUND)
    target_include_directories(${PROJECT_NAME} PRIVATE ${OPENAL_INCLUDE_DIRS})
    target_link_libraries(${PROJECT_NAME} PRIVATE ${OPENAL_LIBRARIES})
endif()

if(WIN32)
    target_link_libraries(${PROJECT_NAME} PRIVATE winmm)
elseif(APPLE)
    find_library(COCOA_LIBRARY Cocoa)
    find_library(IOKIT_LIBRARY IOKit)
    find_library(COREVIDEO_LIBRARY CoreVideo)
    target_link_libraries(${PROJECT_NAME} PRIVATE 
        ${COCOA_LIBRARY} 
        ${IOKIT_LIBRARY} 
        ${COREVIDEO_LIBRARY}
    )
elseif(UNIX)
    target_link_libraries(${PROJECT_NAME} PRIVATE dl pthread)
endif()

install(TARGETS ${PROJECT_NAME}
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
)

option(BUILD_EXAMPLES "Build example applications" ON)

if (BUILD_EXAMPLES)
    add_executable(GameEngineExample 
    examples/main.cpp
    )

    target_link_libraries(GameEngineExample PRIVATE ${PROJECT_NAME})

    target_include_directories(GameEngineExample PRIVATE
        examples
    )

    file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/assets
        DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/bin)
endif()

