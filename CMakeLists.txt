cmake_minimum_required(VERSION 3.5)
project(trantor)

option(BUILD_DOC "Build Doxygen documentation" OFF)
option(BUILD_C-ARES "Build C-ARES" ON)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules/)

set(TRANTOR_MAJOR_VERSION 1)
set(TRANTOR_MINOR_VERSION 5)
set(TRANTOR_PATCH_VERSION 4)
set(TRANTOR_VERSION
    ${TRANTOR_MAJOR_VERSION}.${TRANTOR_MINOR_VERSION}.${TRANTOR_PATCH_VERSION})

include(GNUInstallDirs)
# Offer the user the choice of overriding the installation directories
set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE PATH "Installation directory for binaries")
set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE PATH "Installation directory for libraries")
set(INSTALL_INCLUDE_DIR
    ${CMAKE_INSTALL_INCLUDEDIR}
    CACHE PATH "Installation directory for header files")
set(DEF_INSTALL_TRANTOR_CMAKE_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/Trantor)
set(INSTALL_TRANTOR_CMAKE_DIR
    ${DEF_INSTALL_TRANTOR_CMAKE_DIR}
    CACHE PATH "Installation directory for cmake files")

if(BUILD_TRANTOR_SHARED)
  set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)
  list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES
            "${CMAKE_INSTALL_PREFIX}/${INSTALL_LIB_DIR}" isSystemDir)
  if("${isSystemDir}" STREQUAL "-1")
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${INSTALL_LIB_DIR}")
  endif("${isSystemDir}" STREQUAL "-1")
  add_library(${PROJECT_NAME} SHARED)
  set_target_properties(${PROJECT_NAME} PROPERTIES
      VERSION ${TRANTOR_VERSION}
      SOVERSION ${TRANTOR_MAJOR_VERSION})
  if(CMAKE_CXX_COMPILER_ID MATCHES MSVC)
    # Ignore MSVC C4251 and C4275 warning of exporting std objects with no dll export
    # We export class to facilitate maintenance, thus if you compile
    # drogon on windows as a shared library, you will need to use
    # exact same compiler for drogon and your app.
    target_compile_options(${PROJECT_NAME} PUBLIC /wd4251 /wd4275)
  endif()
else(BUILD_TRANTOR_SHARED)
  add_library(${PROJECT_NAME} STATIC)
endif(BUILD_TRANTOR_SHARED)

if (NOT ${CMAKE_SYSTEM_NAME} STREQUAL "Windows" AND CMAKE_CXX_COMPILER_ID MATCHES Clang|GNU)
    target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra -Werror)
endif()

if(${CMAKE_SYSTEM_NAME} STREQUAL "Haiku")
    target_link_libraries(${PROJECT_NAME} PRIVATE network)
endif()

include(GenerateExportHeader)
generate_export_header(${PROJECT_NAME} EXPORT_FILE_NAME ${CMAKE_CURRENT_BINARY_DIR}/exports/trantor/exports.h)

# include directories
target_include_directories(
  ${PROJECT_NAME}
  PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
         $<INSTALL_INTERFACE:${INSTALL_INCLUDE_DIR}>
         $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/exports>
  PRIVATE ${PROJECT_SOURCE_DIR}
          ${PROJECT_SOURCE_DIR}/trantor/utils
          ${PROJECT_SOURCE_DIR}/trantor/net
          ${PROJECT_SOURCE_DIR}/trantor/net/inner
          $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/third_party/wepoll>)

if(MINGW)
  target_compile_definitions(
    ${PROJECT_NAME}
    PUBLIC -D_WIN32_WINNT=0x0601)
endif(MINGW)

set(TRANTOR_SOURCES
    trantor/utils/AsyncFileLogger.cc
    trantor/utils/ConcurrentTaskQueue.cc
    trantor/utils/Date.cc
    trantor/utils/LogStream.cc
    trantor/utils/Logger.cc
    trantor/utils/MsgBuffer.cc
    trantor/utils/SerialTaskQueue.cc
    trantor/utils/TimingWheel.cc
    trantor/utils/Utilities.cc
    trantor/net/EventLoop.cc
    trantor/net/EventLoopThread.cc
    trantor/net/EventLoopThreadPool.cc
    trantor/net/InetAddress.cc
    trantor/net/TcpClient.cc
    trantor/net/TcpServer.cc
    trantor/net/Channel.cc
    trantor/net/inner/Acceptor.cc
    trantor/net/inner/Connector.cc
    trantor/net/inner/Poller.cc
    trantor/net/inner/Socket.cc
    trantor/net/inner/TcpConnectionImpl.cc
    trantor/net/inner/Timer.cc
    trantor/net/inner/TimerQueue.cc
    trantor/net/inner/poller/EpollPoller.cc
    trantor/net/inner/poller/KQueue.cc
    trantor/net/inner/poller/PollPoller.cc)
set(private_headers
    trantor/net/inner/Acceptor.h
    trantor/net/inner/Connector.h
    trantor/net/inner/Poller.h
    trantor/net/inner/Socket.h
    trantor/net/inner/TcpConnectionImpl.h
    trantor/net/inner/Timer.h
    trantor/net/inner/TimerQueue.h
    trantor/net/inner/poller/EpollPoller.h
    trantor/net/inner/poller/KQueue.h
    trantor/net/inner/poller/PollPoller.h)

if(WIN32)
  set(TRANTOR_SOURCES
      ${TRANTOR_SOURCES}
      third_party/wepoll/Wepoll.c
      trantor/utils/WindowsSupport.cc)
  set(private_headers
      ${private_headers}
      third_party/wepoll/Wepoll.h
      trantor/utils/WindowsSupport.h)
endif(WIN32)

find_package(OpenSSL)
if(OpenSSL_FOUND)
  target_link_libraries(${PROJECT_NAME} PRIVATE OpenSSL::SSL OpenSSL::Crypto)
  target_compile_definitions(${PROJECT_NAME} PRIVATE USE_OPENSSL)
endif()

set(HAVE_C-ARES NO)
if (BUILD_C-ARES)
    find_package(c-ares)
    if(c-ares_FOUND)
      message(STATUS "c-ares found!")
      set(HAVE_C-ARES TRUE)
    endif()
endif ()

if(HAVE_C-ARES)
  target_link_libraries(${PROJECT_NAME} PRIVATE c-ares_lib)
  set(TRANTOR_SOURCES
      ${TRANTOR_SOURCES}
      trantor/net/inner/AresResolver.cc)
  set(private_headers
      ${private_headers}
      trantor/net/inner/AresResolver.h)
else()
  set(TRANTOR_SOURCES
      ${TRANTOR_SOURCES}
      trantor/net/inner/NormalResolver.cc)
  set(private_headers
      ${private_headers}
      trantor/net/inner/NormalResolver.h)
endif()

find_package(Threads)
target_link_libraries(${PROJECT_NAME} PUBLIC Threads::Threads)
if(WIN32)
  target_link_libraries(${PROJECT_NAME} PRIVATE ws2_32 Rpcrt4)
  if(OpenSSL_FOUND)
    target_link_libraries(${PROJECT_NAME} PRIVATE Crypt32 Secur32)
  endif(OpenSSL_FOUND)
endif(WIN32)

file(WRITE ${CMAKE_BINARY_DIR}/test_atomic.cpp
     "#include <atomic>\n"
     "int main() { std::atomic<int64_t> i(0); i++; return 0; }\n")
try_compile(ATOMIC_WITHOUT_LINKING ${CMAKE_BINARY_DIR} ${CMAKE_BINARY_DIR}/test_atomic.cpp)
if (NOT ATOMIC_WITHOUT_LINKING)
    target_link_libraries(${PROJECT_NAME} PUBLIC atomic)
endif ()
file(REMOVE ${CMAKE_BINARY_DIR}/test_atomic.cpp)

set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD 14)
set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD_REQUIRED ON)
set_target_properties(${PROJECT_NAME} PROPERTIES CXX_EXTENSIONS OFF)
set_target_properties(${PROJECT_NAME} PROPERTIES EXPORT_NAME Trantor)

if(BUILD_TESTING)
  add_subdirectory(trantor/tests)
  find_package(GTest)
  if(GTest_FOUND)
    enable_testing()
    add_subdirectory(trantor/unittests)
  endif()
endif()

set(public_net_headers
    trantor/net/EventLoop.h
    trantor/net/EventLoopThread.h
    trantor/net/EventLoopThreadPool.h
    trantor/net/InetAddress.h
    trantor/net/TcpClient.h
    trantor/net/TcpConnection.h
    trantor/net/TcpServer.h
    trantor/net/callbacks.h
    trantor/net/Resolver.h
    trantor/net/Channel.h)

set(public_utils_headers
    trantor/utils/AsyncFileLogger.h
    trantor/utils/ConcurrentTaskQueue.h
    trantor/utils/Date.h
    trantor/utils/Funcs.h
    trantor/utils/LockFreeQueue.h
    trantor/utils/LogStream.h
    trantor/utils/Logger.h
    trantor/utils/MsgBuffer.h
    trantor/utils/NonCopyable.h
    trantor/utils/ObjectPool.h
    trantor/utils/SerialTaskQueue.h
    trantor/utils/TaskQueue.h
    trantor/utils/TimingWheel.h
    trantor/utils/Utilities.h)

target_sources(${PROJECT_NAME} PRIVATE
               ${TRANTOR_SOURCES}
               ${CMAKE_CURRENT_BINARY_DIR}/exports/trantor/exports.h
               ${public_net_headers}
               ${public_utils_headers}
               ${private_headers})

source_group("Public API"
             FILES
             ${CMAKE_CURRENT_BINARY_DIR}/exports/trantor/exports.h
             ${public_net_headers}
             ${public_utils_headers})

source_group("Private Headers"
             FILES
             ${private_headers})

install(TARGETS trantor
                # IMPORTANT: Add the trantor library to the "export-set"
        EXPORT TrantorTargets
        RUNTIME DESTINATION "${INSTALL_BIN_DIR}" COMPONENT bin
        ARCHIVE DESTINATION "${INSTALL_LIB_DIR}" COMPONENT lib
        LIBRARY DESTINATION "${INSTALL_LIB_DIR}" COMPONENT lib)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/exports/trantor/exports.h
        DESTINATION ${INSTALL_INCLUDE_DIR}/trantor)
install(FILES ${public_net_headers}
        DESTINATION ${INSTALL_INCLUDE_DIR}/trantor/net)
install(FILES ${public_utils_headers}
        DESTINATION ${INSTALL_INCLUDE_DIR}/trantor/utils)

include(CMakePackageConfigHelpers)
# ... for the install tree
configure_package_config_file(
  cmake/templates/TrantorConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/TrantorConfig.cmake
  INSTALL_DESTINATION
  ${INSTALL_TRANTOR_CMAKE_DIR})

# version
write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/TrantorConfigVersion.cmake
  VERSION ${TRANTOR_VERSION}
  COMPATIBILITY SameMajorVersion)

# Install the TrantorConfig.cmake and TrantorConfigVersion.cmake
install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/TrantorConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/TrantorConfigVersion.cmake"
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/Findc-ares.cmake"
  DESTINATION "${INSTALL_TRANTOR_CMAKE_DIR}"
  COMPONENT dev)

# Install the export set for use with the install-tree
install(EXPORT TrantorTargets
        DESTINATION "${INSTALL_TRANTOR_CMAKE_DIR}"
        NAMESPACE Trantor::
        COMPONENT dev)

# Doxygen documentation
find_package(Doxygen OPTIONAL_COMPONENTS dot dia)
if(DOXYGEN_FOUND)
  set(DOXYGEN_PROJECT_BRIEF "Non-blocking I/O cross-platform TCP network library, using C++14")
  set(DOXYGEN_OUTPUT_DIRECTORY docs/${PROJECT_NAME})
  set(DOXYGEN_GENERATE_LATEX NO)
  set(DOXYGEN_BUILTIN_STL_SUPPORT YES)
  set(DOXYGEN_USE_MDFILE_AS_MAINPAGE README.md)
  set(DOXYGEN_STRIP_FROM_INC_PATH ${PROJECT_SOURCE_DIR}
                                  ${CMAKE_CURRENT_BINARY_DIR}/exports)
  if (WIN32)
    set(DOXYGEN_PREDEFINED _WIN32)
  endif(WIN32)
  doxygen_add_docs(doc_${PROJECT_NAME}
                   README.md
                   ChangeLog.md
                   ${public_net_headers}
                   ${public_utils_headers}
                   COMMENT "Generate documentation")
  if(NOT TARGET doc)
    add_custom_target(doc)
  endif()
  add_dependencies(doc doc_${PROJECT_NAME})
  if (BUILD_DOC)
    add_dependencies(${PROJECT_NAME} doc_${PROJECT_NAME})
    # Don't install twice, so limit to Debug (assume developer)
    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs/${PROJECT_NAME}
            TYPE DOC
            CONFIGURATIONS Debug)
  endif(BUILD_DOC)
endif(DOXYGEN_FOUND)

