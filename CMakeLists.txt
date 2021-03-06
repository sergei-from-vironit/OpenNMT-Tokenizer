cmake_minimum_required(VERSION 3.2.0)

include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.23.98.tar.gz"
    SHA1 "bf539cb70375ac319e386ae2ec5b5a0c4923480a"
)

project(OpenNMTTokenizer)

option(WITH_ICU "Compile with ICU" OFF)
option(BUILD_SHARED_LIBS "Build shared libraries" ON)

set(CMAKE_CXX_STANDARD 11 CACHE STRING "Default C++ standard")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

if(MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /Wall")
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
endif()

set(PUBLIC_HEADERS
  include/onmt/Alphabet.h
  include/onmt/AnnotatedToken.h
  include/onmt/BPE.h
  include/onmt/BPELearner.h
  include/onmt/CaseModifier.h
  include/onmt/ITokenizer.h
  include/onmt/SpaceTokenizer.h
  include/onmt/SubwordEncoder.h
  include/onmt/SubwordLearner.h
  include/onmt/Tokenizer.h
  )

set(SOURCES
  src/Alphabet.cc
  src/AnnotatedToken.cc
  src/BPE.cc
  src/BPELearner.cc
  src/CaseModifier.cc
  src/ITokenizer.cc
  src/SpaceTokenizer.cc
  src/SubwordEncoder.cc
  src/SubwordLearner.cc
  src/Tokenizer.cc
  src/unicode/Unicode.cc
  )

target_include_directories(
    ${PROJECT_NAME} PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>
)
list(APPEND LINK_LIBRARIES "")

if (WITH_ICU)
  if(HUNTER_ENABLED)
    hunter_add_package(ICU)
    find_package(ICU CONFIG REQUIRED)
  else()

  find_package(ICU REQUIRED)

  endif()

  add_definitions(-DWITH_ICU)

  if(HUNTER_ENABLED)
    list(APPEND LINK_LIBRARIES ICU::uc)
  else()

  list(APPEND INCLUDE_DIRECTORIES ${ICU_INCLUDE_DIRS})
  list(APPEND LINK_LIBRARIES ${ICU_LIBRARIES})

  endif()
else()
  list(APPEND SOURCES src/unicode/Data.cc)
endif()

if(HUNTER_ENABLED)
  hunter_add_package(sentencepiece)
  find_package(sentencepiece CONFIG REQUIRED)
  set(SP_LIBRARY sentencepiece::sentencepiece)
  set(SP_INCLUDE_DIR "${SENTENCEPIECE_ROOT}/include")
else()

find_library(SP_LIBRARY NAMES sentencepiece)
find_path(SP_INCLUDE_DIR NAMES sentencepiece_processor.h)

endif()

if(NOT SP_LIBRARY OR NOT SP_INCLUDE_DIR)
  message(WARNING "sentencepiece not found; will not be supported")
else()
  message(STATUS "Found sentencepiece: ${SP_LIBRARY}")
  add_definitions(-DWITH_SP)
  list(APPEND PUBLIC_HEADERS include/onmt/SentencePiece.h)
  list(APPEND SOURCES src/SentencePiece.cc)
  list(APPEND INCLUDE_DIRECTORIES ${SP_INCLUDE_DIR})
  list(APPEND LINK_LIBRARIES ${SP_LIBRARY})

  if(HUNTER_ENABLED)
    set(SP_TRAIN_LIBRARY sentencepiece::sentencepiece_train)
  else()
  find_library(SP_TRAIN_LIBRARY NAMES sentencepiece_train)
  endif()
  if(SP_TRAIN_LIBRARY)
    add_definitions(-DWITH_SP_TRAIN)
    message(STATUS "Found sentencepiece_train: ${SP_TRAIN_LIBRARY}")
    list(APPEND PUBLIC_HEADERS include/onmt/SPMLearner.h)
    list(APPEND SOURCES src/SPMLearner.cc)
    list(APPEND LINK_LIBRARIES ${SP_TRAIN_LIBRARY})
  else()
    message(WARNING "sentencepiece_train not found: training SentencePiece models will not be supported")
  endif()

  file(STRINGS ${SP_INCLUDE_DIR}/sentencepiece_processor.h HAS_SAMPLE_ENCODE REGEX "SampleEncode")
  if(HAS_SAMPLE_ENCODE)
    add_definitions(-DSP_HAS_SAMPLE_ENCODE)
  endif()

  file(STRINGS ${SP_INCLUDE_DIR}/sentencepiece_processor.h HAS_VOCAB_RESTRICTION REGEX "SetVocabulary")
  if(HAS_VOCAB_RESTRICTION)
    add_definitions(-DSP_HAS_VOCAB_RESTRICTION)
  endif()
endif()

add_library(${PROJECT_NAME} ${SOURCES})
include(GNUInstallDirs)
include(GenerateExportHeader)
string(TOLOWER ${PROJECT_NAME} PROJECT_NAME_LOWER)
generate_export_header(${PROJECT_NAME} EXPORT_FILE_NAME ${PROJECT_BINARY_DIR}/onmt/${PROJECT_NAME_LOWER}_export.h)
target_include_directories(${PROJECT_NAME} PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include> $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
target_link_libraries(${PROJECT_NAME} ${LINK_LIBRARIES})

if (NOT LIB_ONLY)
  add_subdirectory(cli)
  add_subdirectory(test)
endif()

####
# Installation (https://github.com/forexample/package-example)

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

# Use:
#   * targets_export_name
#   * PROJECT_NAME
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
  TARGETS ${PROJECT_NAME}
  EXPORT "${targets_export_name}"
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  )
install(FILES
  ${PUBLIC_HEADERS} "${PROJECT_BINARY_DIR}/onmt/${PROJECT_NAME_LOWER}_export.h"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/onmt"
  )
install(FILES
  include/onmt/unicode/Unicode.h
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/onmt/unicode"
  )

install(
    FILES "${project_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${targets_export_name}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
