cmake_minimum_required(VERSION 2.6)
project(mGBA C)
set(BINARY_NAME mgba CACHE INTERNAL "Name of output binaries")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -std=gnu99")
set(USE_CLI_DEBUGGER ON CACHE BOOL "Whether or not to enable the CLI-mode ARM debugger")
set(USE_GDB_STUB OFF CACHE BOOL "Whether or not to enable the GDB stub ARM debugger")
set(USE_FFMPEG ON CACHE BOOL "Whether or not to enable FFmpeg support")
set(USE_PNG ON CACHE BOOL "Whether or not to enable PNG support")
set(USE_LIBZIP ON CACHE BOOL "Whether or not to enable ZIP support")
set(USE_MAGICK ON CACHE BOOL "Whether or not to enable ImageMagick support")
set(BUILD_QT ON CACHE BOOL "Build Qt frontend")
set(BUILD_SDL ON CACHE BOOL "Build SDL frontend")
set(BUILD_PERF OFF CACHE BOOL "Build performance profiling tool")
file(GLOB ARM_SRC ${CMAKE_SOURCE_DIR}/src/arm/*.c)
file(GLOB GBA_SRC ${CMAKE_SOURCE_DIR}/src/gba/*.c)
file(GLOB UTIL_SRC ${CMAKE_SOURCE_DIR}/src/util/*.[cSs])
file(GLOB VFS_SRC ${CMAKE_SOURCE_DIR}/src/util/vfs/*.c)
file(GLOB RENDERER_SRC ${CMAKE_SOURCE_DIR}/src/gba/renderers/video-software.c)
file(GLOB THIRD_PARTY_SRC ${CMAKE_SOURCE_DIR}/src/third-party/inih/*.c)
list(APPEND UTIL_SRC ${CMAKE_SOURCE_DIR}/src/platform/commandline.c)
source_group("ARM core" FILES ${ARM_SRC})
source_group("GBA board" FILES ${GBA_SRC} ${RENDERER_SRC})
source_group("Utilities" FILES ${UTIL_SRC} ${VFS_SRC}})
include_directories(${CMAKE_SOURCE_DIR}/src/arm)
include_directories(${CMAKE_SOURCE_DIR}/src/gba)
include_directories(${CMAKE_SOURCE_DIR}/src)

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type (e.g. Release or Debug)" FORCE)
endif()

include(GNUInstallDirs)

# Function definitions
include(FindPkgConfig)
function(find_feature FEATURE_NAME FEATURE_REQUIRES)
	if (NOT ${FEATURE_NAME})
		return()
	endif()
	foreach(REQUIRE ${FEATURE_REQUIRES})
		if(NOT ${REQUIRE}_FOUND)
			find_package(${REQUIRE} QUIET)
			if(NOT ${REQUIRE}_FOUND)
				pkg_search_module(${REQUIRE} ${REQUIRE})
				if (NOT ${REQUIRE}_FOUND)
					message(WARNING "Requested module ${REQUIRE} missing for feature ${FEATURE_NAME}. Feature disabled.")
					set(${FEATURE_NAME} OFF PARENT_SCOPE)
					return()
				endif()
			endif()
		endif()
		string(TOUPPER ${REQUIRE} UREQUIRE)
		set(${UREQUIRE}_CFLAGS_OTHER ${${REQUIRE}_CFLAGS_OTHER} PARENT_SCOPE)
		set(${UREQUIRE}_FOUND ${${REQUIRE}_FOUND} PARENT_SCOPE)
		set(${UREQUIRE}_INCLUDE_DIRS ${${REQUIRE}_INCLUDE_DIRS} PARENT_SCOPE)
		set(${UREQUIRE}_LIBRARIES ${${REQUIRE}_LIBRARIES} PARENT_SCOPE)
		set(${UREQUIRE}_LIBRARY_DIRS ${${REQUIRE}_LIBRARY_DIRS} PARENT_SCOPE)
		set(${UREQUIRE}_LDFLAGS_OTHER ${${REQUIRE}_LDFLAGS_OTHER} PARENT_SCOPE)
	endforeach()
endfunction()

# Version information
set(LIB_VERSION_MAJOR 0)
set(LIB_VERSION_MINOR 1)
set(LIB_VERSION_PATCH 1)
set(LIB_VERSION_ABI 0.1)
set(LIB_VERSION_STRING ${LIB_VERSION_MAJOR}.${LIB_VERSION_MINOR}.${LIB_VERSION_PATCH})

# Advanced settings
set(BUILD_PGO CACHE BOOL "Build with profiling-guided optimization")
set(PGO_STAGE_2 CACHE BOOL "Rebuild for profiling-guided optimization after profiles have been generated")
set(PGO_DIR "/tmp/gba-pgo/" CACHE PATH "Profiling-guided optimization profiles path")
mark_as_advanced(BUILD_PGO PGO_STAGE_2 PGO_DIR)
set(PGO_PRE_FLAGS "-pg -fprofile-generate=${PGO_DIR} -fprofile-arcs")
set(PGO_POST_FLAGS "-fprofile-use=${PGO_DIR} -fbranch-probabilities")

if(BUILD_PGO AND NOT PGO_STAGE_2)
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${PGO_PRE_FLAGS}")
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${PGO_PRE_FLAGS}")
	set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${PGO_PRE_FLAGS}")
elseif(BUILD_PGO AND PGO_STAGE_2)
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${PGO_POST_FLAGS}")
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${PGO_POST_FLAGS}")
	set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${PGO_POST_FLAGS}")
endif()

add_definitions(-DBINARY_NAME="${BINARY_NAME}" -DPROJECT_NAME="${PROJECT_NAME}" -DPROJECT_VERSION="${LIB_VERSION_STRING}")

# Feature dependencies
find_feature(USE_CLI_DEBUGGER "libedit")
find_feature(USE_FFMPEG "libavcodec;libavformat;libavresample;libavutil;libswscale")
find_feature(USE_PNG "ZLIB;PNG")
find_feature(USE_LIBZIP "libzip")
find_feature(USE_MAGICK "MagickWand")

include(CheckFunctionExists)
check_function_exists(strndup HAVE_STRNDUP)

if(HAVE_STRNDUP)
	add_definitions(-DHAVE_STRNDUP)
endif()

# Platform support
if(WIN32)
	add_definitions(-D_WIN32_WINNT=0x0600)
	list(APPEND OS_LIB ws2_32)
	file(GLOB OS_SRC ${CMAKE_SOURCE_DIR}/src/platform/windows/*.c)
	source_group("Windows-specific code" FILES ${OS_SRC})
else()
	add_definitions(-DUSE_PTHREADS)
	list(APPEND OS_LIB pthread)
	file(GLOB OS_SRC ${CMAKE_SOURCE_DIR}/src/platform/posix/*.c)
	source_group("POSIX-specific code" FILES ${OS_SRC})
endif()

if(APPLE)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mmacosx-version-min=10.6")
endif()

if(BUILD_BBB OR BUILD_RASPI)
	enable_language(ASM)
	if(NOT BUILD_EGL)
		add_definitions(-DCOLOR_16_BIT -DCOLOR_5_6_5)
	endif()
endif()

# Features
set(DEBUGGER_SRC ${CMAKE_SOURCE_DIR}/src/debugger/debugger.c ${CMAKE_SOURCE_DIR}/src/debugger/memory-debugger.c)
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libc6")

if(USE_CLI_DEBUGGER)
	add_definitions(-DUSE_CLI_DEBUGGER)
	list(APPEND DEBUGGER_SRC ${CMAKE_SOURCE_DIR}/src/debugger/cli-debugger.c)
	list(APPEND DEBUGGER_SRC ${CMAKE_SOURCE_DIR}/src/debugger/parser.c)
	list(APPEND DEBUGGER_SRC ${CMAKE_SOURCE_DIR}/src/gba/gba-cli.c)
	include_directories(AFTER ${LIBEDIT_INCLUDE_DIRS})
	link_directories(${LIBEDIT_LIBRARY_DIRS})
	set(DEBUGGER_LIB ${LIBEDIT_LIBRARIES})
	set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS},libedit2")
else()
	set(DEBUGGER_LIB "")
endif()

if(USE_GDB_STUB)
	add_definitions(-DUSE_GDB_STUB)
	list(APPEND DEBUGGER_SRC ${CMAKE_SOURCE_DIR}/src/debugger/gdb-stub.c)
endif()
source_group("ARM debugger" FILES ${DEBUGGER_SRC})

if(USE_FFMPEG)
	add_definitions(-DUSE_FFMPEG)
	pkg_search_module(LIBSWRESAMPLE QUIET libswresample)
	if(NOT LIBSWRESAMPLE_FOUND)
		add_definitions(-DUSE_LIBAV)
	endif()
	include_directories(AFTER ${LIBAVCODEC_INCLUDE_DIRS} ${LIBAVFORMAT_INCLUDE_DIRS} ${LIBAVRESAMPLE_INCLUDE_DIRS} ${LIBAVUTIL_INCLUDE_DIRS} ${LIBSWSCALE_INCLUDE_DIRS})
	link_directories(${LIBAVCODEC_LIBRARY_DIRS} ${LIBAVFORMAT_LIBRARY_DIRS} ${LIBAVRESAMPLE_LIBRARY_DIRS} ${LIBAVUTIL_LIBRARY_DIRS} ${LIBSWSCALE_LIBRARY_DIRS})
	list(APPEND UTIL_SRC "${CMAKE_SOURCE_DIR}/src/platform/ffmpeg/ffmpeg-encoder.c")
	string(REGEX MATCH "^[0-9]+" LIBAVCODEC_VERSION_MAJOR ${libavcodec_VERSION})
	string(REGEX MATCH "^[0-9]+" LIBAVFORMAT_VERSION_MAJOR ${libavformat_VERSION})
	string(REGEX MATCH "^[0-9]+" LIBAVRESAMPLE_VERSION_MAJOR ${libavresample_VERSION})
	string(REGEX MATCH "^[0-9]+" LIBAVUTIL_VERSION_MAJOR ${libavutil_VERSION})
	string(REGEX MATCH "^[0-9]+" LIBSWSCALE_VERSION_MAJOR ${libswscale_VERSION})
	list(APPEND DEPENDENCY_LIB ${LIBAVCODEC_LIBRARIES} ${LIBAVFORMAT_LIBRARIES} ${LIBAVRESAMPLE_LIBRARIES} ${LIBAVUTIL_LIBRARIES} ${LIBSWSCALE_LIBRARIES})
	set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS},libavcodec${LIBAVCODEC_VERSION_MAJOR},libavformat${LIBAVFORMAT_VERSION_MAJOR},libavresample${LIBAVRESAMPLE_VERSION_MAJOR},libavutil${LIBAVUTIL_VERSION_MAJOR},libswscale${LIBSWSCALE_VERSION_MAJOR}")
	set(CPACK_DEBIAN_PACKAGE_RECOMMENDS "libavcodec-extra")
endif()

if(USE_MAGICK)
	add_definitions(-DUSE_MAGICK)
	include_directories(AFTER ${MAGICKWAND_INCLUDE_DIRS})
	link_directories(${MAGICKWAND_LIBRARY_DIRS})
	list(APPEND UTIL_SRC "${CMAKE_SOURCE_DIR}/src/platform/imagemagick/imagemagick-gif-encoder.c")
	list(APPEND DEPENDENCY_LIB ${MAGICKWAND_LIBRARIES})
	string(REGEX MATCH "^[0-9]+\\.[0-9]+" MAGICKWAND_VERSION_PARTIAL ${MagickWand_VERSION})
	if(${MAGICKWAND_VERSION_PARTIAL} EQUAL "6.7")
		set(MAGICKWAND_DEB_VERSION "5")
	else()
		set(MAGICKWAND_DEB_VERSION "-6.q16-2")
	endif()
	set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS},libmagickwand${MAGICKWAND_DEB_VERSION}")
endif()

if(USE_PNG)
	add_definitions(-DUSE_PNG)
	include_directories(AFTER ${PNG_INCLUDE_DIRS})
	list(APPEND DEPENDENCY_LIB ${PNG_LIBRARIES} ${ZLIB_LIBRARIES})
	set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS},libpng12-0,zlib1g")
endif()

if(USE_LIBZIP)
	include_directories(AFTER ${LIBZIP_INCLUDE_DIRS})
	link_directories(${LIBZIP_LIBRARY_DIRS})
	list(APPEND DEPENDENCY_LIB ${LIBZIP_LIBRARIES})
	add_definitions(-DENABLE_LIBZIP)
	set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS},libzip2")
endif()

# Binaries
add_library(${BINARY_NAME} SHARED
	${ARM_SRC}
	${GBA_SRC}
	${DEBUGGER_SRC}
	${RENDERER_SRC}
	${UTIL_SRC}
	${VFS_SRC}
	${OS_SRC}
	${THIRD_PARTY_SRC})
target_link_libraries(${BINARY_NAME} m ${DEBUGGER_LIB} ${OS_LIB} ${DEPENDENCY_LIB})
install(TARGETS ${BINARY_NAME} DESTINATION lib COMPONENT lib${BINARY_NAME})
set_target_properties(${BINARY_NAME} PROPERTIES VERSION ${LIB_VERSION_STRING} SOVERSION ${LIB_VERSION_ABI})

if(BUILD_SDL)
	add_definitions(-DBUILD_SDL)
	add_subdirectory(${CMAKE_SOURCE_DIR}/src/platform/sdl ${CMAKE_BINARY_DIR}/sdl)
endif()

if(BUILD_QT)
	add_subdirectory(${CMAKE_SOURCE_DIR}/src/platform/qt ${CMAKE_BINARY_DIR}/qt)
endif()

if(BUILD_PERF)
	set(PERF_SRC ${CMAKE_SOURCE_DIR}/src/platform/perf-main.c)
	if(UNIX AND NOT APPLE)
		list(APPEND PERF_LIB rt)
	endif()

	add_executable(${BINARY_NAME}-perf ${PERF_SRC})
	target_link_libraries(${BINARY_NAME}-perf ${BINARY_NAME} ${PERF_LIB})
	install(TARGETS ${BINARY_NAME}-perf DESTINATION bin COMPONENT ${BINARY_NAME}-perf)
	install(FILES ${CMAKE_SOURCE_DIR}/tools/perf.py DESTINATION "${CMAKE_INSTALL_LIBDIR}/${BINARY_NAME}" COMPONENT ${BINARY_NAME}-perf)
endif()

# Packaging
set(CPACK_PACKAGE_VERSION_MAJOR ${LIB_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${LIB_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${LIB_VERSION_PATCH})
set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_SOURCE_DIR}/LICENSE)
set(CPACK_RESOURCE_FILE_README ${CMAKE_SOURCE_DIR}/LICENSE)

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "mGBA Game Boy Advance Emulator")
set(CPACK_PACKAGE_VENDOR "Jeffrey Pfau")
set(CPACK_PACKAGE_CONTACT "Jeffrey Pfau <jeffrey@endrift.com>")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_SOURCE_DIR}/README.md")

set(CPACK_STRIP_FILES ${BINARY_NAME})

include(CPack)

# Summaries
message(STATUS "Feature summary:")
message(STATUS "	CLI debugger: ${USE_CLI_DEBUGGER}")
message(STATUS "	GDB stub: ${USE_GDB_STUB}")
message(STATUS "	Video recording: ${USE_FFMPEG}")
message(STATUS "	GIF recording: ${USE_MAGICK}")
message(STATUS "	Screenshot/advanced savestate support: ${USE_PNG}")
message(STATUS "	ZIP support: ${USE_LIBZIP}")
message(STATUS "Frontend summary:")
message(STATUS "	Qt: ${BUILD_QT}")
message(STATUS "	SDL (${SDL_VERSION}): ${BUILD_SDL}")
message(STATUS "	Profiling: ${BUILD_PERF}")
