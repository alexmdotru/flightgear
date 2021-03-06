cmake_minimum_required (VERSION 3.0)

include (CheckFunctionExists)
include (CheckCSourceCompiles)
include (CheckCXXSourceCompiles)
include (CheckIncludeFile)

if(COMMAND cmake_policy)
   if(POLICY CMP0054)
       cmake_policy(SET CMP0054 NEW)
   endif()
   # Mac RPATH policy
   if(POLICY CMP0042)
       cmake_policy(SET CMP0042 NEW)
   endif()
endif()

message("CMAKE Build type: ${CMAKE_BUILD_TYPE}")
# Set a default build type if none was specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Debug' as none was specified.")
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()

if(NOT FG_BUILD_TYPE)
  message(STATUS "Setting build type to 'Dev' as none was specified.")
  set(FG_BUILD_TYPE Dev CACHE STRING "Choose the FlightGear build type" FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE FG_BUILD_TYPE PROPERTY STRINGS "Dev" "Nightly" "Release")
endif()

if(APPLE)
	set(CMAKE_INSTALL_RPATH "@loader_path/../Frameworks")
	# when building, don't use the install RPATH already
	# (but later on when installing)
	SET(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
endif()

# let's use & require C++11 - note these are only functional with CMake 3.1
# we do manual fallbacks for CMake 3.0 in the compilers section
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED YES)

project(FlightGear)

# We have some custom .cmake scripts not in the official distribution.
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/CMakeModules;${CMAKE_MODULE_PATH}")

# Warning when build is not an out-of-source build.
string(COMPARE EQUAL "${CMAKE_SOURCE_DIR}" "${CMAKE_BINARY_DIR}" InSourceBuild)
if(InSourceBuild)
    message(WARNING  "Avoid building inside the source tree!")
    message(WARNING  "Create a separate build directory instead (i.e. 'fgbuild') and call CMake from there: ")
    message(WARNING  "  mkdir ../fgbuild && cd ../fgbuild && cmake ${CMAKE_SOURCE_DIR}")
endif(InSourceBuild)

# using 10.7 because boost requires libc++ and 10.6 doesn't include it
# Cmake documentation says we must set this before calling project(), but
# it only seems to be picked up setting it /after/ the call to project()
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.7")

include(GNUInstallDirs)
# System detection/default settings
include( DetectDistro )
include( DetectBrowser )

set(CMAKE_DEBUG_POSTFIX          "d" CACHE STRING "add a postfix, usually d on windows")
set(CMAKE_RELEASE_POSTFIX        ""  CACHE STRING "add a postfix, usually empty on windows")
set(CMAKE_RELWITHDEBINFO_POSTFIX ""  CACHE STRING "add a postfix, usually empty on windows")
set(CMAKE_MINSIZEREL_POSTFIX     ""  CACHE STRING "add a postfix, usually empty on windows")

# read 'version' file into a variable (stripping any newlines or spaces)
file(READ version versionFile)
if (NOT versionFile)
    message(FATAL_ERROR "Unable to determine FlightGear version. Version file is missing.")
endif()
string(STRIP "${versionFile}" FLIGHTGEAR_VERSION)
# add a dependency on the versino file
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS version)

# FlightGear packaging (to build a source tarball)
include( ConfigureCPack )

# FlightGear base package path
if (FG_DATA_DIR)
    message(STATUS "Using explicit data directory for base package: ${FG_DATA_DIR}")
else()
    set(FG_DATA_DIR "${CMAKE_INSTALL_PREFIX}/lib/FlightGear" CACHE PATH "Default location where data files are located")
    message(STATUS "Using default data directory for base package:  ${FG_DATA_DIR}")
endif()

# Change the default build type to something fast
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING
        "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
        FORCE)
endif(NOT CMAKE_BUILD_TYPE)

if(NOT "$ENV{BUILD_ID}" STREQUAL "")
    set(HUDSON_BUILD_ID $ENV{BUILD_ID})
    set(HUDSON_BUILD_NUMBER $ENV{BUILD_NUMBER})
    message(STATUS "running under Hudson/Jenkins, build-number is ${HUDSON_BUILD_NUMBER}")
else()
    set(HUDSON_BUILD_NUMBER 0)
    set(HUDSON_BUILD_ID "none")
endif()

#####################################################################################
# Configure library search paths
#####################################################################################

IF(APPLE)
    set(EVENT_INPUT_DEFAULT 1)

    find_library(CORESERVICES_LIBRARY CoreServices)
    find_library(COCOA_LIBRARY Cocoa)
    list(APPEND PLATFORM_LIBS ${COCOA_LIBRARY} ${CORESERVICES_LIBRARY})
elseif(WIN32)
    list(APPEND PLATFORM_LIBS "Shlwapi.lib")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux" OR
	${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
    find_package(Threads REQUIRED)
    find_package(X11 REQUIRED)

    set(USE_DBUS_DEFAULT 1)

    find_package(UDev)
    if(UDEV_FOUND)
        set(EVENT_INPUT_DEFAULT 1)
    endif(UDEV_FOUND)

    find_package(Speex)
    find_package(Speexdsp)
    if(SPEEX_FOUND AND SPEEXDSP_FOUND)
        set(SYSTEM_SPEEX_DEFAULT 1)
    endif(SPEEX_FOUND AND SPEEXDSP_FOUND)

    find_package(Gsm)
    if(GSM_FOUND)
        set(SYSTEM_GSM_DEFAULT 1)
    endif(GSM_FOUND)

    find_package(Flite)
    if(FLITE_FOUND)
        set(SYSTEM_FLITE_DEFAULT 1)
    endif()

    find_package(HtsEngine)
    if(HTS_ENGINE_FOUND)
        set(SYSTEM_HTS_ENGINE_DEFAULT 1)
    endif()
endif()

find_package(Git)
if (GIT_FOUND)
    execute_process(COMMAND git --git-dir ${PROJECT_SOURCE_DIR}/.git rev-parse  HEAD
        OUTPUT_VARIABLE REVISION
        OUTPUT_STRIP_TRAILING_WHITESPACE)
    message(STATUS "Git revision is ${REVISION}")
else()
    set(REVISION "none")
endif()

# FlightGear build options
option(LOGGING           "Set to ON to build FlightGear with logging support (default)" ON)
option(JSBSIM_TERRAIN    "Set to ON to build FlightGear with JSBSim terrain handling code" ON)
option(SP_FDMS           "Set to ON to build FlightGear with special-purpose FDMs" ON)
option(ENABLE_UIUC_MODEL "Set to ON to build FlightGear with UIUCModel FDM" ON)
option(ENABLE_LARCSIM    "Set to ON to build FlightGear with LaRCsim FDM" ON)
option(ENABLE_YASIM      "Set to ON to build FlightGear with YASIM FDM (default)" ON)
option(ENABLE_JSBSIM     "Set to ON to build FlightGear with JSBSim FDM (default)" ON)
option(EVENT_INPUT       "Set to ON to build FlightGear with event-based Input support" ${EVENT_INPUT_DEFAULT})
option(ENABLE_RTI        "Set to ON to build FlightGear with RTI support" OFF)
option(ENABLE_PROFILE    "Set to ON to build FlightGear with gperftools profiling support" OFF)
option(SYSTEM_SQLITE     "Set to ON to build FlightGear with the system's SQLite3 library" OFF)
option(ENABLE_IAX        "Set to ON to build FlightGear with IAXClient/fgcom built-in (default)" ON)
option(USE_DBUS          "Set to ON to build FlightGear with DBus screensaver interaction (default on Linux)" ${USE_DBUS_DEFAULT})
option(USE_AEONWAVE      "Set to ON to use AeonWave instead of OpenAL" OFF)
option(SYSTEM_SPEEX      "Set to ON to build IAXClient with the system's speex and speexdsp library" ${SYSTEM_SPEEX_DEFAULT})
option(SYSTEM_GSM        "Set to ON to build IAXClient with the system's GSM library" ${SYSTEM_GSM_DEFAULT})
option(SYSTEM_FLITE      "Set to ON to build Flightgear with the system's Flite library" ${SYSTEM_FLITE_DEFAULT})
option(SYSTEM_HTS_ENGINE "Set to ON to build Flightgear with the system's HTS Engine library" ${SYSTEM_HTS_ENGINE_DEFAULT})

# additional utilities
option(ENABLE_FGELEV     "Set to ON to build the fgelev application (default)" ON)
option(WITH_FGPANEL      "Set to ON to build the fgpanel application (default)" ON)
option(ENABLE_FGVIEWER   "Set to ON to build the fgviewer application (default)" ON)
option(ENABLE_GPSSMOOTH  "Set to ON to build the GPSsmooth application (default)" ON)
option(ENABLE_TERRASYNC  "Set to ON to build the terrasync application (default)" ON)
option(ENABLE_FGJS       "Set to ON to build the fgjs application (default)" ON)
option(ENABLE_JS_DEMO    "Set to ON to build the js_demo application (default)" ON)
option(ENABLE_METAR      "Set to ON to build the metar application (default)" ON)
option(ENABLE_TESTS      "Set to ON to build test applications (default)" ON)
option(ENABLE_FGCOM      "Set to ON to build the FGCom application (default)" ON)
option(ENABLE_FLITE      "Set to ON to build the Flite text-to-speech module" ON)
option(ENABLE_QT         "Set to ON to build the internal Qt launcher" ON)
option(ENABLE_TRAFFIC    "Set to ON to build the external traffic generator modules" ON)
option(ENABLE_FGQCANVAS  "Set to ON to build the Qt-based remote canvas application" OFF)
option(ENABLE_DEMCONVERT "Set to ON to build the dem conversion tool (default)" ON)

include (DetectArch)

# when building an OSG with commit 15ec7e2ae7a8b983ecc44e1ce7363a9a9fa7da95
# applied, we can use better link options
option(OSG_FSTREAM_EXPORT_FIXED "Set to ON if the osgDB fstream export patch is applied" OFF)

if(LOGGING)
    # nothing
else()
    set(FG_NDEBUG 1)
endif()

if(JSBSIM_TERRAIN)
   set(JSBSIM_USE_GROUNDREACTIONS 1)
endif()

if(SP_FDMS)
    set(ENABLE_SP_FDM 1)
endif()

if(ENABLE_FGCOM)
  set(ENABLE_IAX 1)
  include_directories(${PROJECT_SOURCE_DIR}/3rdparty/iaxclient/lib ) # for iaxclient.h
endif()

# Setup MSVC 3rd party directories
include( ConfigureMsvc3rdParty )

if(EVENT_INPUT)
    if(APPLE)
      add_definitions(-DWITH_EVENTINPUT)
    elseif(CMAKE_SYSTEM_NAME MATCHES "Linux")
        if(NOT UDEV_FOUND)
            message(WARNING "UDev not found, event input is disabled!")
            set(EVENT_INPUT 0)
        else()
            add_definitions(-DWITH_EVENTINPUT)
            set(EVENT_INPUT_LIBRARIES ${UDEV_LIBRARIES})
            message(STATUS "event-based input enabled. Using ${UDEV_LIBRARIES}")
        endif()
    else()
        message(WARNING "event-based input is not supported on this platform yet")
    endif()

    # Keep PLIB INPUT enabled as long as EventInput does not replace current joystick configurations.
    set(ENABLE_PLIB_JOYSTICK 1)
else(EVENT_INPUT)
    set(ENABLE_PLIB_JOYSTICK 1)
endif(EVENT_INPUT)

# check required dependencies
find_package(Boost REQUIRED)
find_package(OpenGL  REQUIRED)
find_package(OpenSceneGraph 3.2.0 REQUIRED
  osgText
  osgSim
  osgDB
  osgParticle
  osgFX
  osgUtil
  osgViewer
  osgGA
)

if (MSVC)
	find_package(CrashRpt)
	if (CRASHRPT_FOUND)
		set(HAVE_CRASHRPT 1)
		message(STATUS "Using CrashRpt")
    include_directories( ${CRASHRPT_INCLUDE_DIR})
	endif()
endif()

##############################################################################
## Sqlite3 setup

if (SYSTEM_SQLITE)
    find_package(SQLite3 REQUIRED)
    message(STATUS "Using system SQLite3 library")
else()
    set(SQLITE3_INCLUDED_DIR "${CMAKE_SOURCE_DIR}/3rdparty/sqlite3")
    # this target is defined in src/Navaids/CMakeLists.txt
    list(APPEND SQLITE3_LIBRARY fgsqlite3)
endif (SYSTEM_SQLITE)

# Sqlite always depends on the threading lib
list(APPEND SQLITE3_LIBRARY ${CMAKE_THREAD_LIBS_INIT})

##############################################################################
## DBus setup

if (USE_DBUS)
    include(FindPkgConfig)
    if (PKG_CONFIG_FOUND)
        pkg_check_modules(DBUS dbus-1)
    endif (PKG_CONFIG_FOUND) #if we don't have pkg-config, assume we don't have libdbus-1-dev either http://packages.debian.org/sid/libdbus-1-dev
    if (DBUS_FOUND)
        set(HAVE_DBUS 1)
        message(STATUS "Using DBus")
        include_directories( ${DBUS_INCLUDE_DIRS})
    else()
        message(STATUS "DBus not found, screensaver control disabled")
    endif (DBUS_FOUND)
else()
endif (USE_DBUS)

##############################################################################
## Qt5 setup setup
if (ENABLE_QT)
	message(STATUS "Qt launcher enabled, checking for Qt 5.1 / qmake")
        find_package(Qt5 5.1 COMPONENTS Widgets Network)
	if (Qt5Widgets_FOUND)
		message(STATUS "Will enable Qt launcher GUI")
		message(STATUS "  Qt5Widgets version: ${Qt5Widgets_VERSION_STRING}")
		message(STATUS "  Qt5Widgets include dir: ${Qt5Widgets_INCLUDE_DIRS}")
		set(HAVE_QT 1)
  else()
    # don't try to build FGQCanvas if Qt wasn't found correctly
    set(ENABLE_FGQCANVAS OFF)
	endif()
else()
  message(STATUS "Qt support disabled")
  set(ENABLE_FGQCANVAS OFF)
endif (ENABLE_QT)
##############################################################################

find_package(PLIB REQUIRED puaux pu js fnt)

# FlightGear and SimGear versions need to match major + minor
# split version string into components, note CMAKE_MATCH_0 is the entire regexp match
string(REGEX MATCH "([0-9]+)\\.([0-9]+)\\.([0-9]+)" VERSION_REGEX ${FLIGHTGEAR_VERSION} )
set(FG_VERSION_MAJOR ${CMAKE_MATCH_1})
set(FG_VERSION_MINOR ${CMAKE_MATCH_2})
set(FG_VERSION_PATCH ${CMAKE_MATCH_3})

set(MIN_SIMGEAR_VERSION "${FG_VERSION_MAJOR}.${FG_VERSION_MINOR}.0")
message(STATUS "Min Simgear version is ${MIN_SIMGEAR_VERSION}")

find_package(SimGear ${MIN_SIMGEAR_VERSION} CONFIG REQUIRED)

##############################################################################

check_include_file(unistd.h   HAVE_UNISTD_H)
check_include_file(sys/time.h HAVE_SYS_TIME_H)
check_include_file(windows.h  HAVE_WINDOWS_H)

if(ENABLE_PROFILE)
    find_package(GooglePerfTools REQUIRED)
    set(FG_HAVE_GPERFTOOLS 1)
    message(STATUS "Built-in profiler using gperftools available")
endif()

if(ENABLE_RTI)
    message(STATUS "RTI: ENABLED")
    find_package(RTI)
    if(RTI_FOUND)
        set(FG_HAVE_HLA 1)
    endif(RTI_FOUND)
else()
    message(STATUS "RTI: DISABLED")
endif(ENABLE_RTI)

if(ENABLE_GDAL)
    find_package(GDAL 2.0.0 REQUIRED)
endif(ENABLE_GDAL)
include_directories(${GDAL_INCLUDE_DIR})

if (ENABLE_OPENMP)
    find_package(OpenMP)
    if(OPENMP_FOUND)
        message(STATUS "OpenMP: ENABLED")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    else()
        message(STATUS "OpenMP: NOT FOUND")
    endif()
else()
    message(STATUS "OpenMP: DISABLED")
endif()

if (ENABLE_SIMD)
  message(STATUS "SSE/SSE2 support: ENABLED")
else()
  message(STATUS "SSE/SSE2 support: DISABLED")
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
    set(WARNING_FLAGS_CXX "-Wall")
    set(WARNING_FLAGS_C   "-Wall")

    if (CMAKE_VERSION VERSION_LESS 3.1)
       set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    endif()

    if(ENABLE_SIMD)
      if (X86 OR X86_64)
        set(CMAKE_C_FLAGS_RELEASE "-O3 -msse2 -mfpmath=sse")
        set(CMAKE_CXX_FLAGS_RELEASE "-O3 -msse2 -mfpmath=sse")
      endif()
    endif()
endif(CMAKE_COMPILER_IS_GNUCXX)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang" )
   set(WARNING_FLAGS_CXX "-Wall -Wno-overloaded-virtual \
     -Wno-redeclared-class-member \
     -Wno-inconsistent-missing-override \
     -Wno-unused-local-typedef")

     if (CMAKE_VERSION VERSION_LESS 3.1)
       set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
     endif()

     if(ENABLE_SIMD)
      if (X86 OR X86_64)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse3 -mfpmath=sse")
      endif()
    endif()

  set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LIBRARY "libc++")
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
   set(WARNING_FLAGS_C "-Wall")
endif()

if(WIN32)

    if(MSVC)
        set(MSVC_FLAGS "-DNOMINMAX -D_USE_MATH_DEFINES -D_CRT_SECURE_NO_WARNINGS -D_SCL_SECURE_NO_WARNINGS -D__CRT_NONSTDC_NO_WARNINGS /MP /O2")
        if(ENABLE_SIMD)
          if (X86)
            SET(CMAKE_C_FLAGS_RELEASE "/O2 /arch:SSE /arch:SSE2")
            SET(CMAKE_CXX_FLAGS_RELEASE "/O2 /arch:SSE /arch:SSE2")
          else()
            SET(CMAKE_C_FLAGS_RELEASE "/O2")
            SET(CMAKE_CXX_FLAGS_RELEASE "/O2")
          endif()
        endif()

        if (NOT OSG_FSTREAM_EXPORT_FIXED AND ${MSVC_VERSION} GREATER 1599)
          message(STATUS "For better linking performance, use OSG with fixed fstream header")
          # needed to avoid link errors on multiply-defined standard C++
          # symbols. This issue was fixed in OSG commit 15ec7e2ae7a8b983ecc44e1ce7363a9a9fa7da95
          set( MSVC_LD_FLAGS "/FORCE:MULTIPLE" )
        endif ()

        if (${MSVC_VERSION} GREATER 1899)
            # needed for debug builds with VS2015
            set( MSVC_FLAGS "${MSVC_FLAGS} /bigobj" )
        endif()
    endif(MSVC)

    set(NOMINMAX 1)
endif(WIN32)

set (BOOST_CXX_FLAGS "-DBOOST_BIMAP_DISABLE_SERIALIZATION")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${WARNING_FLAGS_C} ${MSVC_FLAGS} -D_REENTRANT")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${WARNING_FLAGS_CXX} ${MSVC_FLAGS} -D_REENTRANT ${BOOST_CXX_FLAGS}")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${MSVC_LD_FLAGS}")

include_directories(${OPENSCENEGRAPH_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
    ${OPENGL_INCLUDE_DIR}
    ${SIMGEAR_INCLUDE_DIRS}
    ${PLIB_INCLUDE_DIR}
    ${SQLITE3_INCLUDED_DIR} )

if (USE_AEONWAVE)
    find_package(AAX COMPONENTS aax REQUIRED)
    include_directories(${OPENSCENEGRAPH_INCLUDE_DIRS}
        ${AAX_INCLUDE_DIR}
    )
else()
    include_directories(${OPENSCENEGRAPH_INCLUDE_DIRS}
        ${OPENAL_INCLUDE_DIR}
    )
endif()

include_directories(${PROJECT_SOURCE_DIR})
include_directories(${PROJECT_SOURCE_DIR}/src)
# following is needed, because config.h is include 'bare', whereas
# version.h is included as <Include/version.h> - this should be cleaned up
include_directories(${PROJECT_BINARY_DIR}/src)
include_directories(${PROJECT_BINARY_DIR}/src/Include)

if (ENABLE_FLITE)
    include_directories(${PROJECT_SOURCE_DIR}/3rdparty/hts_engine_API/include )
    include_directories(${PROJECT_SOURCE_DIR}/3rdparty/flite_hts_engine/include )
endif()

add_definitions(-DHAVE_CONFIG_H)

check_function_exists(mkfifo HAVE_MKFIFO)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file (
    "${PROJECT_SOURCE_DIR}/src/Include/config_cmake.h.in"
    "${PROJECT_BINARY_DIR}/src/Include/config.h"
)

#and the same for the version header
configure_file (
    "${PROJECT_SOURCE_DIR}/src/Include/version.h.cmake-in"
    "${PROJECT_BINARY_DIR}/src/Include/version.h"
)

add_subdirectory(3rdparty)
add_subdirectory(utils)

if(ENABLE_TESTS)
    # enable CTest / make test target
    message(STATUS "Tests: ENABLED")

    include (Dart)
    enable_testing()
    if(WIN32)
      # tests disabled until shared library export is fixed on Windows
      message(STATUS "Tests disabled on Windows for the moment")
    else()
      add_subdirectory(tests)
    endif()
else()
    message(STATUS "Tests: DISABLED")
endif(ENABLE_TESTS)

add_subdirectory(src)
add_subdirectory(man)

#-----------------------------------------------------------------------------
### uninstall target
#-----------------------------------------------------------------------------
CONFIGURE_FILE(
    "${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)
ADD_CUSTOM_TARGET(uninstall
    "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
