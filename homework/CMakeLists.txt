cmake_minimum_required(VERSION 3.10)

project(banking)

set(PRINT_VERSION_MAJOR 0)
set(PRINT_VERSION_MINOR 1)
set(PRINT_VERSION_PATCH 0)
set(PRINT_VERSION_TWEAK 0)
set(PRINT_VERSION ${PRINT_VERSION_MAJOR}.${PRINT_VERSION_MINOR}.${PRINT_VERSION_PATCH}.${PRINT_VERSION_TWEAK})
set(PRINT_VERSION_STRING "v${PRINT_VERSION}")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_subdirectory(banking)

# Явно говорим CMake, что библиотеку нужно установить, иначе CPack упадет
install(TARGETS banking DESTINATION lib)
install(DIRECTORY banking/ DESTINATION include/banking FILES_MATCHING PATTERN "*.h")

if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()

include(CPackConfig.cmake)
