cmake_minimum_required(VERSION 3.29)
project(
        Cockatoo
        VERSION 0.0.1
        DESCRIPTION "A simple C++ project template"
        HOMEPAGE_URL "https://github.com/NicolasReboule/Cockatoo"
        LANGUAGES CXX
)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_BUILD_PARALLEL_LEVEL 8)

option(IS_LIBRARY "Is this project a library?" ON)
option(BUILD_COVERAGE "Build the coverage" OFF)
option(BUILD_TESTS "Build the tests" OFF)
option(USE_CLANG_TIDY "Use clang-tidy" OFF) # Used by the CI
option(USE_CLANG_TIDY_WITH_ERRORS "Use clang tidy but all warnings are errors" OFF) # Used by the CI
option(USE_CLANG_TIDY_WITH_FIX "Use clang-tidy with fix" OFF) # Used by the CI

set(CMAKE_INSTALL_INCLUDEDIR include CACHE PATH "Installation directory for header files")

# Headers
add_library(${PROJECT_NAME}_Headers INTERFACE)
target_include_directories(${PROJECT_NAME}_Headers INTERFACE
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Coverage
if (BUILD_COVERAGE)
    target_compile_options(${PROJECT_NAME}_Headers INTERFACE --coverage)
    target_link_options(${PROJECT_NAME}_Headers INTERFACE --coverage)
endif ()

# Sources
add_library(${PROJECT_NAME}_Sources STATIC
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src/example.cpp>
)
target_link_libraries(${PROJECT_NAME}_Sources PUBLIC ${PROJECT_NAME}_Headers)

add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME}_Sources)

if (IS_LIBRARY)
    # Install Lib (Optional if executable only project)
    install(TARGETS ${PROJECT_NAME}_Headers ${PROJECT_NAME}_Sources
            EXPORT ${PROJECT_NAME}_Targets
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT archive
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT library
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT runtime
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR} COMPONENT headers
    )

    install(DIRECTORY include
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/../ # Remove the extra include directory
            COMPONENT headers
    )

    install(EXPORT ${PROJECT_NAME}_Targets
            FILE ${PROJECT_NAME}Targets.cmake
            NAMESPACE ${PROJECT_NAME}::
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME} COMPONENT cmake
    )

    # Setup the version file
    include(CMakePackageConfigHelpers)
    write_basic_package_version_file(
            ${PROJECT_NAME}ConfigVersion.cmake
            VERSION ${PROJECT_VERSION}
            COMPATIBILITY AnyNewerVersion
    )

    # generate the config file that is includes the exports
    configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/Config.cmake.in
            "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
            INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
            NO_SET_AND_CHECK_MACRO
            NO_CHECK_REQUIRED_COMPONENTS_MACRO
    )

    install(FILES
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME} COMPONENT cmake
    )

    export(EXPORT ${PROJECT_NAME}_Targets FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Targets.cmake")
else ()
    # Main executable (Optional if library only project)
    add_executable(${PROJECT_NAME} main.cpp)
    target_link_libraries(${PROJECT_NAME} PRIVATE ${PROJECT_NAME}_Sources)

    # Compiler Warnings
    target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_20)
    if (MSVC)
        target_compile_options(${PROJECT_NAME} PRIVATE /W4 /WX)
    else ()
        target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra -pedantic -Werror)
    endif ()
endif ()

# Tests
if (BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

# CLang-tidy
if (USE_CLANG_TIDY OR USE_CLANG_TIDY_WITH_ERRORS OR USE_CLANG_TIDY_WITH_FIX)
    find_program(CLANG_TIDY_EXE NAMES "clang-tidy-19")
    if (NOT CLANG_TIDY_EXE)
        message(WARNING "clang-tidy not found.")
    else ()
        message(STATUS "clang-tidy found: ${CLANG_TIDY_EXE}")
        if (USE_CLANG_TIDY_WITH_FIX)
            set(CLANG_TIDY_COMMAND "${CLANG_TIDY_EXE}" "--fix" "--config-file=${CMAKE_SOURCE_DIR}/.clang-tidy")
        elseif (USE_CLANG_TIDY_WITH_ERRORS)
            set(CLANG_TIDY_COMMAND "${CLANG_TIDY_EXE}" "--warnings-as-errors=*" "--config-file=${CMAKE_SOURCE_DIR}/.clang-tidy")
        else ()
            set(CLANG_TIDY_COMMAND "${CLANG_TIDY_EXE}" "--config-file=${CMAKE_SOURCE_DIR}/.clang-tidy")
        endif ()
        set_target_properties(${BINARY_NAME} PROPERTIES CXX_CLANG_TIDY "${CLANG_TIDY_COMMAND}")
    endif ()
endif ()
