
set( BUILD_SHARED_LIBS False )
add_library( liblua  STATIC 
                       src/lapi.c src/lcode.c src/ldebug.c src/ldo.c 
                       src/ldump.c src/lfunc.c src/lgc.c src/llex.c
                       src/lmem.c src/lobject.c src/lopcodes.c 
                       src/lparser.c src/lstate.c src/lstring.c
                       src/ltable.c src/ltm.c src/lundump.c src/lvm.c 
                       src/lzio.c src/lauxlib.c src/lbaselib.c 
                       src/ldblib.c src/liolib.c src/lmathlib.c 
                       src/loslib.c src/ltablib.c src/lstrlib.c 
                       src/loadlib.c src/linit.c )
if( UNIX )
    target_link_libraries( liblua m )
endif()
add_executable( lua src/lua.c )
target_link_libraries( lua liblua )

add_executable( luac src/luac.c 
                     src/print.c )
target_link_libraries( luac liblua )


