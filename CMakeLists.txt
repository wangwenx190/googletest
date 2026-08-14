cmake_minimum_required(VERSION 4.0)

project(googletest-distribution VERSION 1.18.0 LANGUAGES CXX)

if(WIN32)
  enable_language(RC)
endif()

include(GNUInstallDirs)

option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(BUILD_GMOCK "Build GoogleMock" ON)
option(INSTALL_GTEST "Enable installation" OFF)
option(GTEST_HAS_ABSL "Use Abseil and RE2" OFF)
option(gtest_force_shared_crt "Use the shared MSVC runtime" OFF)
option(gtest_disable_pthreads "Disable pthread support" OFF)
option(gtest_hide_internal_symbols "Hide internal symbols" OFF)

if(MSVC AND PROJECT_IS_TOP_LEVEL AND NOT DEFINED CMAKE_MSVC_RUNTIME_LIBRARY)
  if(BUILD_SHARED_LIBS OR gtest_force_shared_crt)
    set(CMAKE_MSVC_RUNTIME_LIBRARY
      "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
  else()
    set(CMAKE_MSVC_RUNTIME_LIBRARY
      "MultiThreaded$<$<CONFIG:Debug>:Debug>")
  endif()
endif()

set(THREADS_PREFER_PTHREAD_FLAG ON)
if(NOT gtest_disable_pthreads)
  find_package(Threads)
endif()

function(configure_googletest_target target resource_file)
  target_compile_features(${target} PUBLIC cxx_std_17)
  set_target_properties(${target} PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
    CXX_EXTENSIONS OFF)

  if(MSVC)
    target_compile_options(${target} PRIVATE
      /W4 /J /utf-8 /permissive- /wd4251 /wd4275)
    target_compile_definitions(${target} PRIVATE
      _UNICODE UNICODE WIN32_LEAN_AND_MEAN STRICT)
  else()
    target_compile_options(${target} PRIVATE
      -Wall -Wextra -Wshadow -Wconversion -Wpedantic)
  endif()

  if(CMAKE_USE_PTHREADS_INIT)
    target_compile_definitions(${target} PUBLIC GTEST_HAS_PTHREAD=1)
    target_link_libraries(${target} PUBLIC Threads::Threads)
  else()
    target_compile_definitions(${target} PUBLIC GTEST_HAS_PTHREAD=0)
  endif()

  if(BUILD_SHARED_LIBS)
    target_compile_definitions(${target}
      PRIVATE GTEST_CREATE_SHARED_LIBRARY=1
      INTERFACE GTEST_LINKED_AS_SHARED_LIBRARY=1)
  endif()

  if(gtest_hide_internal_symbols)
    set_target_properties(${target} PROPERTIES
      CXX_VISIBILITY_PRESET hidden
      VISIBILITY_INLINES_HIDDEN ON)
  endif()

  if(WIN32 AND BUILD_SHARED_LIBS)
    target_sources(${target} PRIVATE "${resource_file}")
  endif()
endfunction()

function(enable_absl_for target)
  target_compile_definitions(${target} PUBLIC GTEST_HAS_ABSL=1)
  target_link_libraries(${target} PUBLIC
    absl::failure_signal_handler
    absl::flags_parse
    absl::flags_reflection
    absl::flags_usage
    absl::stacktrace
    absl::strings
    absl::symbolize
    re2::re2)
endfunction()

add_library(gtest "googletest/src/gtest-all.cc")
add_library(GTest::gtest ALIAS gtest)
target_include_directories(gtest
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googletest/include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googletest>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/googletest")
configure_googletest_target(gtest "${CMAKE_CURRENT_SOURCE_DIR}/gtest.rc")

if(GTEST_HAS_ABSL)
  enable_absl_for(gtest)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "QNX" AND
   CMAKE_SYSTEM_VERSION VERSION_GREATER_EQUAL 7.1)
  target_link_libraries(gtest PUBLIC regex)
endif()

add_library(gtest_main "googletest/src/gtest_main.cc")
add_library(GTest::gtest_main ALIAS gtest_main)
target_include_directories(gtest_main PRIVATE
  "${CMAKE_CURRENT_SOURCE_DIR}/googletest")
target_link_libraries(gtest_main PUBLIC GTest::gtest)
configure_googletest_target(gtest_main
  "${CMAKE_CURRENT_SOURCE_DIR}/gtest_main.rc")

if(BUILD_GMOCK)
  if(MSVC)
    add_library(gmock
      "googletest/src/gtest-all.cc"
      "googlemock/src/gmock-all.cc")
  else()
    add_library(gmock "googlemock/src/gmock-all.cc")
  endif()
  add_library(GTest::gmock ALIAS gmock)
  target_include_directories(gmock
    PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googlemock/include>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googlemock>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googletest/include>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googletest>
      $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    PRIVATE
      "${CMAKE_CURRENT_SOURCE_DIR}/googlemock"
      "${CMAKE_CURRENT_SOURCE_DIR}/googletest")
  if(NOT MSVC)
    target_link_libraries(gmock PUBLIC GTest::gtest)
  elseif(GTEST_HAS_ABSL)
    enable_absl_for(gmock)
  endif()
  configure_googletest_target(gmock
    "${CMAKE_CURRENT_SOURCE_DIR}/gmock.rc")

  if(MSVC)
    add_library(gmock_main
      "googletest/src/gtest-all.cc"
      "googlemock/src/gmock-all.cc"
      "googlemock/src/gmock_main.cc")
  else()
    add_library(gmock_main "googlemock/src/gmock_main.cc")
  endif()
  add_library(GTest::gmock_main ALIAS gmock_main)
  target_include_directories(gmock_main
    PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googlemock/include>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googlemock>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googletest/include>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/googletest>
      $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    PRIVATE
      "${CMAKE_CURRENT_SOURCE_DIR}/googlemock"
      "${CMAKE_CURRENT_SOURCE_DIR}/googletest")
  if(MSVC)
    if(GTEST_HAS_ABSL)
      enable_absl_for(gmock_main)
    endif()
  else()
    target_link_libraries(gmock_main PUBLIC GTest::gmock)
  endif()
  configure_googletest_target(gmock_main
    "${CMAKE_CURRENT_SOURCE_DIR}/gmock_main.rc")
endif()

if(INSTALL_GTEST)
  set(googletest_targets gtest gtest_main)
  if(BUILD_GMOCK)
    list(APPEND googletest_targets gmock gmock_main)
  endif()

  install(TARGETS ${googletest_targets}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
  install(DIRECTORY googletest/include/gtest
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
  if(BUILD_GMOCK)
    install(DIRECTORY googlemock/include/gmock
      DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
  endif()
endif()
