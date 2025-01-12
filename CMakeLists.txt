cmake_minimum_required(VERSION 3.5)

project(newParseGCode VERSION 0.1 LANGUAGES C CXX)

set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

# Default to C99
if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

# Default to C++17
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(QT NAMES Qt6 Qt5 COMPONENTS Widgets REQUIRED)
find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Widgets REQUIRED)

message("-- system processor: ${CMAKE_SYSTEM_PROCESSOR}")
cmake_host_system_information(RESULT host_platform QUERY OS_PLATFORM)
message("-- processor platfrom: ${host_platform}")

execute_process(COMMAND uname -m OUTPUT_VARIABLE MACHINES)
execute_process(COMMAND getconf LONG_BIT OUTPUT_VARIABLE MACHINES_BIT)
message(STATUS "Machine : ${MACHINES}")
message(STATUS "Machine Bits : ${MACHINES_BIT}")
message(STATUS "System : ${CMAKE_SYSTEM_NAME}")

# 获取工程路径
add_definitions(-DPROJECT_PATH="${PROJECT_SOURCE_DIR}")

# set(CMAKE_BUILD_TYPE "Debug")
if (${CMAKE_BUILD_TYPE} MATCHES "Debug")
  # add_definitions(-DBACKTRACE_DEBUG)
endif()

# 执行git命令，并把结果重定向到自定义变量（此处是我的变量<GIT_VERSION>）。
execute_process(
	COMMAND	git log -1 --format=%H 		
	WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} 
	OUTPUT_VARIABLE GIT_VERSION
)
execute_process(
	COMMAND git remote -v 			
	WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} 
	OUTPUT_VARIABLE GIT_REMOTE_PATH
)

# 对得到的结果进行处理，尤其注意 \n,\t,\r之类的特殊字符，在cmake时没问题，但是生成的.cmake文件有问题，导致make出错。
string (REGEX REPLACE ".*\t| .*" "" GIT_PATH ${GIT_REMOTE_PATH})
string (REGEX REPLACE "[\n\t\r]" "" GIT_VERSION ${GIT_VERSION})

# 增加编译选项，把宏导入源码
add_definitions( -DGIT_VERSION=\"${GIT_VERSION}\")
add_definitions( -DGIT_PATH=\"${GIT_PATH}\")

set(COMMON_LIBRARIES
    pthread
)

include_directories(
  src
  src/compensation
  src/common
  src/widget
  src/parseCode
)

if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
  find_package(PkgConfig REQUIRED)
  if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64")
    if (${host_platform} MATCHES "x86_64")
      # 交叉编译
      message(STATUS "Cross compile")
    else()
      # arm64 主机编译
      message(STATUS "Host mechine compile")
    endif()
  endif()
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
# find_package(hidapi REQUIRED)
find_package(PkgConfig REQUIRED)
set(CMAKE_EXE_LINKER_FLAGS "-framework IOKit -framework CoreFoundation")
if (${MACHINES} MATCHES "arm64")
  include_directories("/usr/local/include")
else()
  set(OPENSSL_ROOT_DIR "/usr/local/opt/openssl")
  include_directories(${OPENSSL_ROOT_DIR}/include)
endif()
else()
#elseif(${CMAKE_SYSTEM_NAME} MATCHES "CYGWIN")
  message(STATUS "System : ${CMAKE_SYSTEM_NAME} support")
endif()

set(PROJECT_SOURCES
  src/main.cpp
  src/mainwindow.cpp
  src/common/commonfunc.cpp
  src/common/element.cpp
  src/parseCode/parsecode.cpp
  src/widget/codeeditwidget.cpp
  src/widget/graphwidget.cpp
  src/widget/drawingarea.cpp
  src/compensation/compensation.cpp
  src/mainwindow.h
  src/common/commonfunc.h
  src/common/element.h
  src/parseCode/parsecode.h
  src/widget/codeeditwidget.h
  src/widget/graphwidget.h
  src/widget/drawingarea.h
  src/compensation/compensation.h
  src/mainwindow.ui
)

if(${QT_VERSION_MAJOR} GREATER_EQUAL 6)
    qt_add_executable(${PROJECT_NAME}
        MANUAL_FINALIZATION
        ${PROJECT_SOURCES}
    )
    find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Core5Compat REQUIRED)
    list(APPEND COMMON_LIBRARIES Qt${QT_VERSION_MAJOR}::Core5Compat)
# Define target properties for Android with Qt 6 as:
#    set_property(TARGET ${PROJECT_NAME} APPEND PROPERTY QT_ANDROID_PACKAGE_SOURCE_DIR
#                 ${CMAKE_CURRENT_SOURCE_DIR}/android)
# For more information, see https://doc.qt.io/qt-6/qt-add-executable.html#target-creation
else()
    if(ANDROID)
        add_library(${PROJECT_NAME} SHARED
            ${PROJECT_SOURCES}
        )
# Define properties for Android with Qt 5 after find_package() calls as:
#    set(ANDROID_PACKAGE_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/android")
    else()
        add_executable(${PROJECT_NAME}
            ${PROJECT_SOURCES}
        )
    endif()
endif()

target_link_libraries(${PROJECT_NAME} PRIVATE 
    Qt${QT_VERSION_MAJOR}::Widgets
    ${COMMON_LIBRARIES}
)

# Qt for iOS sets MACOSX_BUNDLE_GUI_IDENTIFIER automatically since Qt 6.1.
# If you are developing for iOS or macOS you should consider setting an
# explicit, fixed bundle identifier manually though.
if(${QT_VERSION} VERSION_LESS 6.1.0)
  set(BUNDLE_ID_OPTION MACOSX_BUNDLE_GUI_IDENTIFIER com.example.smart_car)
endif()
set_target_properties(${PROJECT_NAME} PROPERTIES
    ${BUNDLE_ID_OPTION}
    MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
    MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}
    MACOSX_BUNDLE TRUE
    WIN32_EXECUTABLE TRUE
)

include(GNUInstallDirs)
install(TARGETS ${PROJECT_NAME}
    BUNDLE DESTINATION .
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

if(QT_VERSION_MAJOR EQUAL 6)
    qt_finalize_executable(${PROJECT_NAME})
endif()
