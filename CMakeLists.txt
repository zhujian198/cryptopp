cmake_minimum_required(VERSION 2.8)
project(cryptopp)

option(CRYPTOPP_FIPS_COMPLIANT "CryptoPP FIPS 140 compliant" OFF)
option(CRYPTOPP_BUILD_TESTS "Build CryptoPP tests" ON)
option(CRYPTOPP_BUILD_STATIC_LIBS "Build static CryptoPP libraries" ON)
option(CRYPTOPP_BUILD_SHARED_LIBS "Build shared CryptoPP libraries" ON)
option(CRYPTOPP_INSTALL "Create CryptoPP install target" ON)

if (NOT CRYPTOPP_BUILD_STATIC_LIBS AND CRYPTOPP_BUILD_TESTS)
  set(CRYPTOPP_BUILD_STATIC_LIBS ON CACHE BOOL "" FORCE)
endif ()

if (NOT CRYPTOPP_BUILD_SHARED_LIBS AND CRYPTOPP_BUILD_TESTS)
  set(CRYPTOPP_BUILD_SHARED_LIBS ON CACHE BOOL "" FORCE)
endif ()

file(GLOB PROJECT_CPP *.cpp)
file(GLOB PROJECT_H *.h)
file(GLOB PROJECT_PCH pch.h)

set(PROJECT_TEST
    bench
    bench2
    test
    validat1
    validat2
    validat3
    adhoc
    datatest
    regtest
    fipsalgt
    fipstest)

if (WIN32)
  list(APPEND PROJECT_TEST dlltest)
endif ()

set(PROJECT_TEST_CPP)

foreach (_PROJECT_TEST_FILE ${PROJECT_TEST})
  foreach (_PROJECT_FILE ${PROJECT_CPP})
    if (${_PROJECT_FILE} MATCHES ".*${_PROJECT_TEST_FILE}\\.cpp$")
      list(REMOVE_ITEM PROJECT_CPP ${_PROJECT_FILE})
    endif ()
  endforeach ()
  list(APPEND PROJECT_TEST_CPP ${_PROJECT_TEST_FILE}.cpp)
endforeach ()

set(PROJECT_CPP_LIB ${PROJECT_CPP})
set(PROJECT_CPP_SHARED ${PROJECT_CPP})

list(APPEND PROJECT_CPP_SHARED ${CMAKE_CURRENT_SOURCE_DIR}/fipstest.cpp)
list(REMOVE_ITEM PROJECT_CPP_SHARED ${CMAKE_CURRENT_SOURCE_DIR}/ttmac.cpp)

foreach (_PROJECT_FILE ${PROJECT_CPP_LIB})
  if (${_PROJECT_FILE} MATCHES ".*dll.+\\.cpp$")
    list(REMOVE_ITEM PROJECT_CPP_LIB ${_PROJECT_FILE})
  endif ()
endforeach ()

add_definitions(-DCRYPTOPP_DISABLE_ASM)
remove_definitions(-DUNICODE -D_UNICODE)

if (WIN32)
  list(APPEND PROJECT_CPP_SHARED cryptopp.rc)
endif ()

if (CRYPTOPP_BUILD_SHARED_LIBS)
  add_library(cryptopp
      SHARED
      ${PROJECT_CPP_SHARED}
      ${PROJECT_H}
      ${PROJECT_PCH})
endif (CRYPTOPP_BUILD_SHARED_LIBS)

if (CRYPTOPP_BUILD_STATIC_LIBS)
  add_library(cryptlib
      STATIC
      ${PROJECT_CPP_LIB}
      ${PROJECT_H}
      ${PROJECT_PCH})
endif ()

if (CRYPTOPP_BUILD_TESTS)
  add_executable(crypttest ${PROJECT_TEST_CPP})

  if (CMAKE_COMPILER_IS_GNUCXX)
    set_target_properties(crypttest PROPERTIES LINK_FLAGS -pthread)
  endif ()

  if (WIN32)
    add_executable(dlltest dlltest.cpp)
  endif ()
endif ()

set (_CRYPTOPP_COMPILE_DEFINITIONS USE_PRECOMPILED_HEADERS)

if (CRYPTOPP_FIPS_COMPLIANT)
  list(APPEND _CRYPTOPP_COMPILE_DEFINITIONS
      CRYPTOPP_ENABLE_COMPLIANCE_WITH_FIPS_140_2=1)
else ()
  list(APPEND _CRYPTOPP_COMPILE_DEFINITIONS
      CRYPTOPP_ENABLE_COMPLIANCE_WITH_FIPS_140_2=0)
endif ()

if (CRYPTOPP_BUILD_STATIC_LIBS)
  list (APPEND _CRYPTOPP_INSTALL_TARGETS cryptlib)
endif ()

if (CRYPTOPP_BUILD_SHARED_LIBS)
  list (APPEND _CRYPTOPP_INSTALL_TARGETS cryptopp)
endif ()

# Prevent MSVC from issuing the LNK4221 warning
set_source_files_properties (strciphr.cpp simple.cpp polynomi.cpp algebra.cpp
  eccrypto.cpp eprecomp.cpp PROPERTIES HEADER_FILE_ONLY TRUE)

if (MSVC)
    set_target_properties(${_CRYPTOPP_INSTALL_TARGETS} PROPERTIES
    COMPILE_DEFINITIONS "${_CRYPTOPP_COMPILE_DEFINITIONS}")
endif ()

if (_CRYPTOPP_INSTALL_TARGETS AND WIN32)
  set_target_properties(${_CRYPTOPP_INSTALL_TARGETS} PROPERTIES DEFINE_SYMBOL
    CRYPTOPP_EXPORTS)
endif ()

if (TARGET crypttest)
  target_link_libraries(crypttest cryptlib cryptopp)
endif ()

if (TARGET dlltest)
  target_link_libraries(dlltest cryptopp)
endif ()

if (WIN32)
  get_target_property(_TEMP cryptopp COMPILE_DEFINITIONS)

  if (NOT _TEMP)
    unset(_TEMP)
  endif ()

  #  list(APPEND _TEMP _USRDLL)

  if (TARGET cryptopp)
    set_target_properties(cryptopp PROPERTIES COMPILE_DEFINITIONS "${_TEMP}")
  endif ()

  if (TARGET cryptlib)
  endif ()

  if (CRYPTOPP_BUILD_TESTS)
    set_target_properties(crypttest PROPERTIES COMPILE_DEFINITIONS _CONSOLE)

    set(_TEMP CRYPTOPP_DLL_ONLY _CONSOLE)
    set_target_properties(dlltest PROPERTIES COMPILE_DEFINITIONS
      "${_TEMP};CRYPTOPP_NO_LIB")
  endif ()
endif ()

if (TARGET cryptopp)
  set_target_properties(cryptopp PROPERTIES PUBLIC_HEADER "${PROJECT_H}")
endif ()

if (TARGET cryptlib)
  set_target_properties(cryptlib PROPERTIES PUBLIC_HEADER "${PROJECT_H}")
endif ()

IF (WIN32)
  if (TARGET cryptopp)
    target_link_libraries(cryptopp Ws2_32)
  endif ()

  if (TARGET cryptlib)
    target_link_libraries(cryptlib Ws2_32)
  endif ()
endif ()

if (CRYPTOPP_INSTALL AND _CRYPTOPP_INSTALL_TARGETS)
  install(TARGETS ${_CRYPTOPP_INSTALL_TARGETS}
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    PUBLIC_HEADER DESTINATION include/cryptopp)
endif ()

if (TARGET crypttest)
  enable_testing()
  # Run validation tests
  add_test(NAME cryptopp-validation COMMAND crypttest v WORKING_DIRECTORY
    ${CMAKE_CURRENT_SOURCE_DIR})
endif ()

if (TARGET crypttest)
  get_target_property(crypttest RUNTIME_OUTPUT_DIRECTORY _DATA_DIR)

  if (NOT _DATA_DIR)
    set(_DATA_DIR ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
  endif ()

  set(_DATA_DIR ${_DATA_DIR}/${CMAKE_CFG_INTDIR})

  add_custom_command(TARGET crypttest POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
            ${CMAKE_CURRENT_SOURCE_DIR}/TestData ${_DATA_DIR}/TestData
    COMMAND ${CMAKE_COMMAND} -E copy_directory
            ${CMAKE_CURRENT_SOURCE_DIR}/TestVectors ${_DATA_DIR}/TestVectors
    COMMENT "Copying cryptopp test data")
endif ()
