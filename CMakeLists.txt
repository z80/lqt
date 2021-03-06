#  Copyright (c) 2008 Peter Kuemmel 
#  Copyright (c) 2008 Mauro Iazzi
#  Copyright (c) 2011 Michal Kottman
#  
#  Permission is hereby granted, free of charge, to any person
#  obtaining a copy of this software and associated documentation
#  files (the "Software"), to deal in the Software without
#  restriction, including without limitation the rights to use,
#  copy, modify, merge, publish, distribute, sublicense, and/or sell
#  copies of the Software, and to permit persons to whom the
#  Software is furnished to do so, subject to the following
#  conditions:
#  
#  The above copyright notice and this permission notice shall be
#  included in all copies or substantial portions of the Software.
#  
#  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
#  EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES
#  OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
#  NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
#  HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
#  WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
#  FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
#  OTHER DEALINGS IN THE SOFTWARE.

cmake_minimum_required(VERSION 2.6)

project(lqt)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY  ${CMAKE_BINARY_DIR}/bin) 
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY  ${CMAKE_BINARY_DIR}/lib) 
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY  ${CMAKE_BINARY_DIR}/lib) 

find_package(Qt4 REQUIRED)
set(QT_USE_QTCORE 1)
set(QT_USE_QTGUI 1)
set(QT_USE_QTNETWORK 1)
set(QT_USE_QTWEBKIT 1)
set(QT_USE_QTOPENGL 1)
set(QT_USE_QTSCRIPT 1)
set(QT_USE_QTSVG 1)
set(QT_USE_QTUITOOLS 1) 
set(QT_USE_QTXML 1)
set(QT_USE_QTSQL 1)
set(QT_USE_QTXMLPATTERNS 1)
include(${QT_USE_FILE})

if(noqt)
	set(QT_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/test)
	#set(nomerge TRUE)
endif()

#find_package(Lua51 REQUIRED)
add_subdirectory( "./lua" )
#find_program(LUA_CMD NAMES luajit lua lua5.1 lua51)
set( LUA_CMD         lua )
set( LUA_LIBRARIES   liblua )
set( LUA_INCLUDE_DIR ./lua/src )
set( CMAKE_INSTALL_PREFIX ./app )

if(WIN32)
	if(MSVC)
		set(CMAKE_SUPPRESS_REGENERATION TRUE)
                # -DLUA_BUILD_AS_DLL
		add_definitions(-D_CRT_SECURE_NO_DEPRECATE -D_CRT_SECURE_NO_WARNINGS -D_SCL_SECURE_NO_WARNINGS /bigobj)
		set(CMAKE_DEBUG_POSTFIX)
	endif()
	if(MINGW)
		set(CMAKE_SHARED_LINKER_FLAGS "-Wl,-add-stdcall-alias")
	endif()
	set(LUA_LIBS_FOR_MODULE ${LUA_LIBRARIES})
else(WIN32)
	set(LUA_LIBS_FOR_MODULE )
endif(WIN32)

if(APPLE)
   set(CMAKE_SHARED_MODULE_CREATE_CXX_FLAGS
       "${CMAKE_SHARED_MODULE_CREATE_CXX_FLAGS} -flat_namespace -undefined dynamic_lookup")
endif(APPLE)

add_subdirectory(cpptoxml)

function(build_qt_lib_binding BINDING_HEADER)

	string(TOLOWER ${BINDING_HEADER} BINDING_NAME)

	# hardcoded in generator.lua
	set (LQT_GEN_DIR ${CMAKE_BINARY_DIR}/${BINDING_NAME}_src)
	file(MAKE_DIRECTORY ${LQT_GEN_DIR})

	set (BINDING_SRC ${LQT_GEN_DIR}/${BINDING_NAME}_merged_build.cpp)
	set_source_files_properties(${BINDING_SRC} PROPERTIES  GENERATED TRUE)

	if (UNIX)
		set (PP_CONFIG ${CMAKE_SOURCE_DIR}/cpptoxml/parser/rpp/pp-qt-configuration)
	else()
		set (PP_CONFIG ${CMAKE_SOURCE_DIR}/cpptoxml/parser/rpp/pp-qt-configuration-win)
	endif()

	if(NOT REUSE_FILES)
		set (GEN_XML ${LQT_GEN_DIR}/${BINDING_NAME}.xml)
		add_custom_target(generate_${BINDING_NAME}_xml
							COMMAND cpptoxml
								${BINDING_HEADER}
								-C ${PP_CONFIG}
								-Q ${QT_INCLUDE_DIR} 
								-o ${GEN_XML}
							COMMENT "Generating XML: running cpptoxml on ${BINDING_NAME} "
							VERBATIM)

		math(EXPR MAX ${ARGC}-1)
		if(MAX)
			foreach(I RANGE 1 ${MAX})
				string(TOLOWER ${ARGV${I}} DEPNAME)
				set(${BINDING_NAME}_TYPES ${${BINDING_NAME}_TYPES} -t ${DEPNAME}_src/${DEPNAME}_types.lua)
				set(${BINDING_NAME}_DEPENDS ${${BINDING_NAME}_DEPENDS} generate_${DEPNAME}_cpp)
			endforeach()
		endif()
		
		add_custom_target(generate_${BINDING_NAME}_cpp
							COMMAND ${LUA_CMD}
								"${CMAKE_SOURCE_DIR}/generator/generator.lua" 
								${GEN_XML} 
								-i ${BINDING_HEADER} -i lqt_qt.hpp 
								-n ${BINDING_NAME} 
								-t ${CMAKE_SOURCE_DIR}/generator/qtypes.lua
								${${BINDING_NAME}_TYPES}
								-f ${CMAKE_SOURCE_DIR}/generator/qt_internal.lua
							COMMENT "Generating binding code with Lua"
							VERBATIM)
	else()
		add_custom_target(generate_${BINDING_NAME}_xml 	COMMAND
							COMMENT "Reusing old XML on ${BINDING_NAME} "
							VERBATIM)
		add_custom_target(generate_${BINDING_NAME}_cpp
							COMMENT "Reusing old binding code with Lua"
							VERBATIM)
	endif()

	set(COMMON COMMON_${BINDING_NAME})
	file(GLOB ${COMMON}_SRC ${CMAKE_SOURCE_DIR}/common/*.cpp)
	file(GLOB ${COMMON}_HPP ${CMAKE_SOURCE_DIR}/common/*.hpp)


	set(${BINDING_NAME}_GENERATED ${LQT_GEN_DIR}/${BINDING_NAME}_enum.cpp
									${LQT_GEN_DIR}/${BINDING_NAME}_meta.cpp
									${LQT_GEN_DIR}/${BINDING_NAME}_slot.cpp)
	set_source_files_properties(${${BINDING_NAME}_GENERATED} PROPERTIES GENERATED TRUE)

	if(nomerge)
		#TODO fix this build process:
		#	1. cmake ../lqt -Dnomerge=1
		#	2. make -> will generate cpp files but they are not part of the Makefile (GLOB was at 1.)
		# 	3. cmake ../lqt -Dnomerge=1 -> GLOB now finds the generated cpp files
		#	4. make
		file(GLOB BINDING_SRC ${LQT_GEN_DIR}/*.cpp) 
		set(${COMMON}_SRC)
		list(REMOVE_ITEM BINDING_SRC ${LQT_GEN_DIR}/${BINDING_NAME}_merged_build.cpp)
	else()				
		set(${COMMON}_SRC ${${COMMON}_SRC}	${${BINDING_NAME}_GENERATED})
	endif()

	qt4_wrap_cpp(${COMMON}_MOC ${LQT_GEN_DIR}/${BINDING_NAME}_slot.hpp)
	
	include_directories(${CMAKE_SOURCE_DIR}/common ${LUA_INCLUDE_DIR} ${QT_INCLUDES})

	add_library(${BINDING_NAME} MODULE ${BINDING_SRC}
											${${COMMON}_SRC}
											${${COMMON}_HPP} 
											${${COMMON}_MOC})
											
	target_link_libraries(${BINDING_NAME} ${QT_LIBRARIES} ${LUA_LIBS_FOR_MODULE})
	set_target_properties(${BINDING_NAME} PROPERTIES PREFIX "" COMPILE_DEFINITIONS "MODULE_${BINDING_NAME}")

	if(LQT_BUILDIN_CPPTOXML)
		add_dependencies(generate_${BINDING_NAME}_xml  cpptoxml)
	endif()

	add_dependencies(generate_${BINDING_NAME}_cpp  generate_${BINDING_NAME}_xml ${${BINDING_NAME}_DEPENDS})
	add_dependencies(${BINDING_NAME} generate_${BINDING_NAME}_cpp)
endfunction()

if(noqt)
	include_directories(${CMAKE_CURRENT_SOURCE_DIR}/test/noqt)
	build_qt_lib_binding(noqt)
else()
	build_qt_lib_binding(QtCore)
	build_qt_lib_binding(QtNetwork QtCore)
	build_qt_lib_binding(QtGui QtNetwork QtCore)
	build_qt_lib_binding(QtScript QtGui QtNetwork QtCore)
	build_qt_lib_binding(QtOpenGL QtGui QtNetwork QtCore)
	build_qt_lib_binding(QtSvg QtScript QtGui QtNetwork QtCore)
	build_qt_lib_binding(QtXml QtSvg QtScript QtGui QtNetwork QtCore)
	build_qt_lib_binding(QtXmlPatterns QtCore QtXml)
	build_qt_lib_binding(QtUiTools QtCore QtGui QtXml) 
	build_qt_lib_binding(QtWebKit QtXmlPatterns QtSvg QtScript QtOpenGL QtGui QtNetwork QtCore QtXml)
	build_qt_lib_binding(QtSql QtCore)
	# TODO
	#build_qt_lib_binding(QtScriptTools) # 4.5 only
	#build_qt_lib_binding(Phonon)
endif()







