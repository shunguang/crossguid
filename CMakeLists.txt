########################################################################
# version requirements
########################################################################
if(NOT DEFINED required_cmake_ver)
    set(required_cmake_ver 3.5.1)
    message(STATUS "required_cmake_ver set to ${required_cmake_ver}")
endif()
cmake_minimum_required(VERSION ${required_cmake_ver})
if(NOT DEFINED required_cxx_ver)
    set(required_cxx_ver 17)
    message(STATUS "required_cxx_ver set to ${required_cxx_ver}")
endif()

########################################################################
# start project
########################################################################
project(CrossGuid VERSION 0.2.3)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake")
option(CROSSGUID_TESTS "Build test runner" ON)
set(CMAKE_CXX_STANDARD ${required_cxx_ver})
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_DEBUG_POSTFIX "-dgb")

########################################################################
# figure out pkg_dir, pkg_lib_dir, and pkg_bin_dir
########################################################################
if(NOT DEFINED pkg_dir)
	set(pkg_dir "${CMAKE_CURRENT_SOURCE_DIR}/../" )
endif()
if(NOT DEFINED pkg_lib_dir)
    set(pkg_lib_dir "${PROJECT_BINARY_DIR}/lib")
endif()
if(NOT DEFINED pkg_bin_dir)
    set(pkg_bin_dir "${PROJECT_BINARY_DIR}/bin")
endif()
message(pkg_dir="${pkg_dir}")
message(pkg_lib_dir="${pkg_lib_dir}")
message(pkg_bin_dir="${pkg_bin_dir}")

# CMAKE_ARCHIVE_OUTPUT_DIRECTORY control the output dir of static libs
# CMAKE_LIBRARY_OUTPUT_DIRECTORY control the output dir of dynamic/dll libs
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${pkg_lib_dir}")   
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${pkg_lib_dir}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${pkg_bin_dir}")


# Set the build type if not set
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif()

add_library(crossguid STATIC
	${CMAKE_CURRENT_SOURCE_DIR}/src/guid.cpp
)
set_property(TARGET crossguid PROPERTY POSITION_INDEPENDENT_CODE ON)
target_include_directories(crossguid PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>)

if(WIN32)
    target_compile_definitions(crossguid PRIVATE GUID_WINDOWS)
elseif(APPLE)
    find_library(CFLIB CoreFoundation)
    target_link_libraries(crossguid ${CFLIB})
    target_compile_definitions(crossguid PRIVATE GUID_CFUUID)
elseif(ANDROID)
    # GUID_ANDROID is used in the headers, so make PUBLIC
    target_compile_definitions(crossguid PUBLIC GUID_ANDROID)
else()
    find_package(Libuuid REQUIRED)
    if (NOT LIBUUID_FOUND)
        message(FATAL_ERROR
            "You might need to run 'sudo apt-get install uuid-dev' or similar")
    endif()
    target_include_directories(crossguid PRIVATE ${LIBUUID_INCLUDE_DIR})
    target_link_libraries(crossguid ${LIBUUID_LIBRARY})
    target_compile_definitions(crossguid PRIVATE GUID_LIBUUID)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    set(WARNINGS "-Werror" "-Wall")
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(WARNINGS "-Werror" "-Wall")
elseif(MSVC)
    set(WARNINGS "/WX" "/W4")
endif()
target_compile_options(crossguid PRIVATE ${WARNINGS})

set_target_properties(crossguid
					  PROPERTIES
					  VERSION ${PROJECT_VERSION}
					  SOVERSION ${PROJECT_VERSION_MAJOR}
					  DEBUG_POSTFIX ${CMAKE_DEBUG_POSTFIX})


if (0)
if (${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_CURRENT_SOURCE_DIR})
	include(GNUInstallDirs)

	set(CROSSGUID_INC_INSTALL_DIR "${CMAKE_INSTALL_INCLUDEDIR}")
	set(CROSSGUID_RUNTIME_INSTALL_DIR "${CMAKE_INSTALL_BINDIR}")
	set(CROSSGUID_LIBRARY_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}")
	set(CROSSGUID_ARCHIVE_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}")
	set(CROSSGUID_FRAMEWORK_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}")

	set(CROSSGUID_CMAKE_CONFIG_INSTALL_DIR "${CMAKE_INSTALL_DATADIR}/crossguid/cmake")
	set(CROSSGUID_ADDITIONAL_FILES_INSTALL_DIR "${CMAKE_INSTALL_DATADIR}/crossguid")

	message(CROSSGUID_INC_INSTALL_DIR="${CROSSGUID_INC_INSTALL_DIR}")
	
	# Install target
	install(TARGETS crossguid EXPORT crossguidTargets
		RUNTIME       DESTINATION ${CROSSGUID_RUNTIME_INSTALL_DIR}
		LIBRARY       DESTINATION ${CROSSGUID_LIBRARY_INSTALL_DIR}
		ARCHIVE       DESTINATION ${CROSSGUID_ARCHIVE_INSTALL_DIR}
		FRAMEWORK     DESTINATION ${CROSSGUID_FRAMEWORK_INSTALL_DIR})

	# Install headers
	install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/include/"
		DESTINATION ${CROSSGUID_INC_INSTALL_DIR})

	# Make cmake config files for all targets
	install(EXPORT  crossguidTargets
		DESTINATION ${CROSSGUID_CMAKE_CONFIG_INSTALL_DIR}
		FILE crossguid-config.cmake)

	# Install readme and license
	install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE" "${CMAKE_CURRENT_SOURCE_DIR}/README.md"
		DESTINATION ${CROSSGUID_ADDITIONAL_FILES_INSTALL_DIR})

	configure_file(crossguid.pc.in ${PROJECT_BINARY_DIR}/crossguid.pc @ONLY)
	install(FILES ${PROJECT_BINARY_DIR}/crossguid.pc DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR}/pkgconfig)
	endif()
endif()

if (CROSSGUID_TESTS)
    add_executable(crossguid-test test/TestMain.cpp test/Test.cpp)
    target_link_libraries(crossguid-test crossguid)
endif()
