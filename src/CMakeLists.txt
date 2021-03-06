cmake_minimum_required ( VERSION 2.8 )

FOREACH ( policy CMP0054 CMP0057 CMP0042 )
	IF ( POLICY ${policy} )
		CMAKE_POLICY ( SET ${policy} NEW )
	ENDIF ()
ENDFOREACH ()

# main bunch of sources (for libsphinx)
set ( LIBSPHINX_SRCS sphinx.cpp sphinxexcerpt.cpp
		sphinxquery.cpp sphinxsoundex.cpp sphinxmetaphone.cpp
		sphinxstemen.cpp sphinxstemru.cpp sphinxstemru.inl sphinxstemcz.cpp
		sphinxstemar.cpp sphinxutils.cpp sphinxstd.cpp
		sphinxsort.cpp sphinxexpr.cpp sphinxfilter.cpp
		sphinxsearch.cpp sphinxrt.cpp sphinxjson.cpp
		sphinxaot.cpp sphinxplugin.cpp sphinxudf.c
		sphinxqcache.cpp sphinxjsonquery.cpp
		attribute.cpp secondaryindex.cpp killlist.cpp searchnode.cpp json/cJSON.c
		sphinxpq.cpp icu.cpp global_idf.cpp docstore.cpp lz4/lz4.c lz4/lz4hc.c
		searchdexpr.cpp snippetfunctor.cpp snippetindex.cpp snippetstream.cpp
		threadutils.cpp sphinxversion.cpp indexcheck.cpp datareader.cpp
		indexformat.cpp indexsettings.cpp fileutils.cpp )
set ( INDEXER_SRCS indexer.cpp )
set ( INDEXTOOL_SRCS indextool.cpp )
set ( SEARCHD_SRCS_TESTABLE searchdha.cpp http/http_parser.c searchdhttp.cpp
		searchdreplication.cpp  searchdtask.cpp taskping.cpp
		taskmalloctrim.cpp taskoptimize.cpp taskglobalidf.cpp tasksavestate.cpp
		taskflushbinlog.cpp taskflushattrs.cpp taskflushmutable.cpp taskpreread.cpp
		searchdaemon.cpp searchdssl.cpp searchdfields.cpp searchdconfig.cpp )
set ( SEARCHD_SRCS searchd.cpp ${SEARCHD_SRCS_TESTABLE} )
set ( SPELLDUMP_SRCS spelldump.cpp )
set ( TESTS_SRCS tests.cpp )
set ( WORDBREAKER_SRCS sphinxstd.cpp wordbreaker.cpp)
set ( INDEX_CONVERTER_SRCS sphinxstd.cpp index_converter.cpp)
set ( LIBSPHINX_BISON sphinxexpr.y sphinxselect.y sphinxquery.y sphinxjson.y )
set ( LIBSPHINX_FLEX sphinxjson.l )
set ( SEARCHD_BISON sphinxql.y )
set ( SEARCHD_FLEX sphinxql.l )
set ( INDEX_CONV_SRCS index_converter.cpp )
set ( UDFEXAMPLE_SRC udfexample.c )

# The headers is not neccessary for executable build, but nesessary for MSVC and other projects
# all the (non-generated) headers
# So if you add headers to the project and NOT see them in visual studio solution - just list them here!
file ( GLOB HEADERS "sphinx*.h" lz4.h lz4hc.h http/http_parser.h secondaryindex.h searchnode.h killlist.h
		attribute.h accumulator.h global_idf.h, optional.h, event.h)
file ( GLOB SEARCHD_H "searchd*.h" "task*.h" threadutils.h )

if ( UNIX AND NOT APPLE )
	add_subdirectory ( replication )
	# add_compile_definitions( HAVE_WSREP ) # need cmake >3.12, let's skip it for a while..
    add_definitions ( -DHAVE_WSREP )
	set ( WSREP "wsrep")
	LIST (APPEND SEARCHD_H "replication/wsrep_api.h" )
else()
	set ( WSREP "" )
	LIST ( APPEND SEARCHD_H "replication/wsrep_api_stub.h" )
endif()

file ( GLOB GHEADERS "yysphinx*.h" )
source_group ( "Generated Header Files" FILES ${GHEADERS} )
source_group ( "Grammar sources" FILES ${LIBSPHINX_BISON} ${SEARCHD_BISON} )
source_group ( "Lexer sources" FILES ${LIBSPHINX_FLEX} ${SEARCHD_FLEX} )

include ( genheader )

# compile globalaliases.h
set ( CHARSET_FILTER "charsets/*.txt" )
set ( CHARSET_TEMPLATE "charsets/globalaliases.h.in" )
hsnippetfolder ( "${CHARSET_FILTER}" "${CHARSET_TEMPLATE}" )

# populate charset files for IDE
file ( GLOB CHARSET_FILES "${CHARSET_FILTER}" )
source_group ( "Charset sources" FILES ${CHARSET_FILES} ${CHARSET_TEMPLATE} )

add_custom_target (
		vergen
		COMMAND
		${CMAKE_COMMAND}
			-D SOURCE_DIR=${CMAKE_SOURCE_DIR}
			-D BINARY_DIR=${CMAKE_BINARY_DIR}
			-D SPHINX_TAG=${SPHINX_TAG}
			-D CPACK_SUFFIX=${SPHINX_PACKAGE_SUFFIX}
			-D DISTR_SUFFIX=${DISTR_SUFFIX}
			-D PACKAGE_NAME=${PACKAGE_NAME}
			-P ${CMAKE_SOURCE_DIR}/sphinxrev.cmake
		COMMENT "Generate version from sources"
		VERBATIM
		SOURCES ${CMAKE_SOURCE_DIR}/sphinxrev.cmake
			${CMAKE_SOURCE_DIR}/src/sphinxversion.h.in
			${CMAKE_SOURCE_DIR}/dist/CPackOptions.cmake.in
)
include_directories ( "${MANTICORE_BINARY_DIR}/config" )
add_definitions ( "-DBUILD_WITH_CMAKE" )

# add the extra targets in the case we want on-the-fly grammar compiler
if ( USE_BISON )

	set ( BISON_DIR "${MANTICORE_BINARY_DIR}/config" )

	BISON_TARGET ( ExprParser "${CMAKE_CURRENT_SOURCE_DIR}/sphinxexpr.y" ${BISON_DIR}/bissphinxexpr.c )
	BISON_TARGET ( SelectParser "${CMAKE_CURRENT_SOURCE_DIR}/sphinxselect.y" ${BISON_DIR}/bissphinxselect.c )
	BISON_TARGET ( QueryParser "${CMAKE_CURRENT_SOURCE_DIR}/sphinxquery.y" ${BISON_DIR}/bissphinxquery.c )
	BISON_TARGET ( SQLParser "${CMAKE_CURRENT_SOURCE_DIR}/sphinxql.y" ${BISON_DIR}/bissphinxql.c )
	BISON_TARGET ( JsonParser "${CMAKE_CURRENT_SOURCE_DIR}/sphinxjson.y" ${BISON_DIR}/bissphinxjson.c )

	set_property ( SOURCE sphinxexpr.cpp APPEND PROPERTY
			OBJECT_DEPENDS ${BISON_ExprParser_OUTPUT_SOURCE} )
	set_property ( SOURCE sphinx.cpp APPEND PROPERTY
			OBJECT_DEPENDS ${BISON_SelectParser_OUTPUT_SOURCE} )
	set_property ( SOURCE sphinxquery.cpp APPEND PROPERTY
			OBJECT_DEPENDS ${BISON_QueryParser_OUTPUT_SOURCE} )
	set_property ( SOURCE searchd.cpp gtests_searchd.cpp APPEND PROPERTY
			OBJECT_DEPENDS ${BISON_SQLParser_OUTPUT_SOURCE} )
	set_property ( SOURCE sphinxjson.cpp APPEND PROPERTY
			OBJECT_DEPENDS ${BISON_JsonParser_OUTPUT_SOURCE} )


	# also add dependencies for tarball (to always have fresh grammar inside)
	list (APPEND TARBALLDEPENDS ${BISON_ExprParser_OUTPUTS})
	list ( APPEND TARBALLDEPENDS ${BISON_SelectParser_OUTPUTS} )
	list ( APPEND TARBALLDEPENDS ${BISON_QueryParser_OUTPUTS} )
	list ( APPEND TARBALLDEPENDS ${BISON_SQLParser_OUTPUTS} )
	list ( APPEND TARBALLDEPENDS ${BISON_JsonParser_OUTPUTS} )
else()
	if ( NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/yysphinxselect.h" )
		message ( SEND_ERROR "BISON is not active, and pre-generated parsers not found. Either install/activate bison, or build from 'tarball' generated on machine with bison installed")
	endif()
endif ( USE_BISON )

# add the extra targets in the case we want on-the-fly lex compiler
if ( USE_FLEX )
	set ( FLEX_DIR "${MANTICORE_BINARY_DIR}/config" )

	FLEX_TARGET ( SQLlex "${CMAKE_CURRENT_SOURCE_DIR}/sphinxql.l" ${FLEX_DIR}/flexsphinxql.c COMPILE_FLAGS "-i" )
	FLEX_TARGET ( Jsonlex "${CMAKE_CURRENT_SOURCE_DIR}/sphinxjson.l" ${FLEX_DIR}/flexsphinxjson.c COMPILE_FLAGS "-i -Pyy2" )

	set_property ( SOURCE searchd.cpp gtests_searchd.cpp APPEND PROPERTY
			OBJECT_DEPENDS ${FLEX_SQLlex_OUTPUTS} )
	set_property ( SOURCE sphinxjson.cpp APPEND PROPERTY
			OBJECT_DEPENDS ${FLEX_Jsonlex_OUTPUTS} )

	# also add dependencies for tarball (to always have fresh parsers inside)
	list ( APPEND TARBALLDEPENDS ${FLEX_SQLlex_OUTPUTS} )
	list ( APPEND TARBALLDEPENDS ${FLEX_Jsonlex_OUTPUTS} )
else ()
	if ( NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/llsphinxql.c" )
		message ( SEND_ERROR "FLEX is not active, and pre-generated parsers not found. Either install/activate flex, or build from 'tarball' generated on machine with flex installed" )
	endif ()
endif ( USE_FLEX )

add_custom_target (
		tarball
		COMMAND ${CMAKE_COMMAND}
		-D SOURCE_DIR=${CMAKE_SOURCE_DIR}
		-D BINARY_DIR=${CMAKE_BINARY_DIR}
		-D USE_BISON=${USE_BISON}
		-D USE_FLEX=${USE_FLEX}
		-D WITH_STEMMER=${WITH_STEMMER}
		-D STEMMER_BASEDIR=${STEMMER_BASEDIR}
		-D WITH_RE2=${WITH_RE2}
		-D RE2_BASEDIR=${RE2_BASEDIR}
		-P ${CMAKE_SOURCE_DIR}/cmake/mktarball.cmake
		COMMENT "Making tarball source archive"
		DEPENDS ${TARBALLDEPENDS}
		VERBATIM
		SOURCES ${CMAKE_SOURCE_DIR}/cmake/mktarball.cmake
)
add_dependencies ( tarball vergen )

# our mega-lib
add_library ( libsphinx STATIC ${LIBSPHINX_SRCS} ${HEADERS} ${GHEADERS} ${LIBSPHINX_BISON} ${LIBSPHINX_FLEX} ${CHARSET_FILES} ${CHARSET_TEMPLATE} "../misc/manticore.natvis" )
add_dependencies ( libsphinx vergen )

add_library ( lsearchd OBJECT ${SEARCHD_SRCS_TESTABLE} ${SEARCHD_H} ${SEARCHD_BISON} ${SEARCHD_FLEX} )
if ( USE_SSL )
	include_directories ( ${OPENSSL_INCLUDE_DIR} )
	target_link_libraries ( lsearchd OpenSSL::SSL )		
endif ( USE_SSL )

add_dependencies ( lsearchd vergen )

# our executables
add_executable ( indexer ${INDEXER_SRCS} )
add_executable ( indextool ${INDEXTOOL_SRCS} )
add_executable ( searchd searchd.cpp )
add_executable ( spelldump ${SPELLDUMP_SRCS} )
add_executable ( wordbreaker ${WORDBREAKER_SRCS} )
add_executable ( index_converter ${INDEX_CONV_SRCS} )
add_dependencies ( wordbreaker vergen )
target_link_libraries ( indexer libsphinx ${EXTRA_LIBRARIES} )
target_link_libraries ( indextool libsphinx ${EXTRA_LIBRARIES} )
target_link_libraries ( searchd lsearchd libsphinx ${EXTRA_LIBRARIES} ${EXTRA_SEARCHD_LIBRARIES} ${WSREP})
target_link_libraries ( spelldump libsphinx ${EXTRA_LIBRARIES} )
target_link_libraries ( wordbreaker libsphinx ${EXTRA_LIBRARIES} )
target_link_libraries ( index_converter libsphinx ${EXTRA_LIBRARIES} )

add_library ( udfexample SHARED ${UDFEXAMPLE_SRC} )

IF ( USE_GALERA )
    SET_SEARCHD_RPATH ()
endif ()

include ( helpers )


if ( SPLIT_SYMBOLS )
       split_dbg ( indexer )
       split_dbg ( indextool )
       split_dbg ( searchd )
       split_dbg ( spelldump )
       split_dbg ( wordbreaker )
       split_dbg ( index_converter )
endif()


INSTALL ( TARGETS indexer indextool spelldump wordbreaker RUNTIME DESTINATION ${BINPREFIX}bin COMPONENT tools )
INSTALL ( TARGETS searchd RUNTIME DESTINATION ${BINPREFIX}bin COMPONENT applications )

INSTALL ( TARGETS index_converter RUNTIME DESTINATION ${BINPREFIX}bin COMPONENT converter )
INSTALL(FILES sphinxudf.h  DESTINATION usr/include/manticore COMPONENT devel)

if ( NOT DISABLE_TESTING )

	if ( CMAKE_VERSION VERSION_GREATER 3.1.0 AND NOT DISABLE_GTESTS )
		add_subdirectory ( gtests )
	endif()

	add_executable ( tests ${TESTS_SRCS} )
	target_link_libraries ( tests libsphinx ${EXTRA_LIBRARIES} )

	# compile rtestconfig.h
	hsnippet ( rtestconfig testrt.conf "static" )

	set (DATAFLD "${MANTICORE_BINARY_DIR}/test/data")
	file ( MAKE_DIRECTORY ${DATAFLD} )
	add_definitions ( -DDATAFLD="${DATAFLD}/" )
	source_group ( "Testing config" FILES testrt.conf.in )
	add_executable ( testrt testrt.cpp testrt.conf )
	target_link_libraries ( testrt libsphinx ${EXTRA_LIBRARIES} )

	# testrt is alive, however it needs database 'lj' accessible
	# in order to work. So, avoid it's calling till the base is persist
	# in out CI env
	if (0)

	add_test ( NAME "\"Remove dump index\""
			COMMAND ${CMAKE_COMMAND} -E remove "${DATAFLD}/dump.*" )
	SET_TESTS_PROPERTIES ( "\"Remove dump index\"" PROPERTIES FIXTURES_SETUP TESTRT )

	add_test ( NAME "\"Internal rt src/testrt\""
			WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
			COMMAND testrt )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt\"" PROPERTIES FIXTURES_REQUIRED TESTRT )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt\"" PROPERTIES RESOURCE_LOCK DbAccess )

	add_test ( NAME "\"Internal rt src/testrt step 1000\""
			WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
			COMMAND testrt 1000 )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt step 1000\"" PROPERTIES FIXTURES_REQUIRED TESTRT )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt step 1000\"" PROPERTIES RESOURCE_LOCK DbAccess )

	add_test ( NAME "\"Internal rt src/testrt step 100\""
			WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
			COMMAND testrt 100 )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt step 100\"" PROPERTIES FIXTURES_REQUIRED TESTRT )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt step 1000\"" PROPERTIES RESOURCE_LOCK DbAccess )

	add_test ( NAME "\"Internal rt src/testrt step 10\""
			WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
			COMMAND testrt 10 )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt step 10\"" PROPERTIES FIXTURES_REQUIRED TESTRT )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt step 1000\"" PROPERTIES RESOURCE_LOCK DbAccess )

	add_test ( NAME "\"Internal rt src/testrt step 1\""
			WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
			COMMAND testrt 1 )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt step 1\"" PROPERTIES FIXTURES_REQUIRED TESTRT )
	SET_TESTS_PROPERTIES ( "\"Internal rt src/testrt step 1000\"" PROPERTIES RESOURCE_LOCK DbAccess )

	endif(0)

	# first check syntax
	find_package ( PythonInterp QUIET )
	if ( PYTHONINTERP_FOUND AND NOT DISABLE_GTESTS )
		set ( RESERVED_TEST "\"SphinxQL reserved keywords consistency\"" )
		message ( STATUS "python binary is ${PYTHON_EXECUTABLE}" ) # !COMMIT
		add_test ( NAME ${RESERVED_TEST}
				WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
				COMMAND ${PYTHON_EXECUTABLE} reserved.py )
		SET_TESTS_PROPERTIES ( ${RESERVED_TEST} PROPERTIES LABELS LINTER )
	endif()
	add_test ( NAME "\"Internal src/tests\""
			#    	CONFIGURATIONS Debug
			WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
			COMMAND "tests" )
endif()
