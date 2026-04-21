cmake_minimum_required(VERSION 3.15)
project(texture0 CXX)

# C++ 標準の設定
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

# macOS の Xcode で開いた際に Run で起動できるようにスキームを生成する
set(CMAKE_XCODE_GENERATE_SCHEME ON)

# ソースファイルとヘッダー
set(SOURCES main.cpp trackball.cpp)
set(HEADERS trackball.h)
set(RESOURCES tire.raw)

# ターゲットの追加
if(APPLE)
    list(APPEND RESOURCES opengl.icns)
    add_executable(texture0 MACOSX_BUNDLE ${SOURCES} ${HEADERS} ${RESOURCES})
else()
    add_executable(texture0 ${SOURCES} ${HEADERS})
endif()

if(WIN32)
    # freeglut をダウンロードしてビルド
    include(FetchContent)
    
    set(FREEGLUT_BUILD_DEMOS OFF CACHE BOOL "" FORCE)
    set(FREEGLUT_BUILD_SHARED_LIBS ON CACHE BOOL "" FORCE)
    set(FREEGLUT_BUILD_STATIC_LIBS OFF CACHE BOOL "" FORCE)
    
    FetchContent_Declare(
        freeglut
        GIT_REPOSITORY https://github.com/FreeGLUTProject/freeglut.git
        GIT_TAG master
    )
    FetchContent_MakeAvailable(freeglut)

    # glext.h と khrplatform.h をダウンロード
    set(GLEXT_DIR ${CMAKE_BINARY_DIR}/third_party/glext)
    if(NOT EXISTS ${GLEXT_DIR}/GL/glext.h)
        file(DOWNLOAD https://www.khronos.org/registry/OpenGL/api/GL/glext.h ${GLEXT_DIR}/GL/glext.h)
    endif()
    if(NOT EXISTS ${GLEXT_DIR}/KHR/khrplatform.h)
        file(DOWNLOAD https://www.khronos.org/registry/EGL/api/KHR/khrplatform.h ${GLEXT_DIR}/KHR/khrplatform.h)
    endif()
    
    target_include_directories(texture0 PRIVATE ${GLEXT_DIR})
    
    # 共有ライブラリと OpenGL をリンク
    target_link_libraries(texture0 PRIVATE freeglut opengl32.lib glu32.lib)
    
    # DLL と リソースを実行ディレクトリへコピー
    add_custom_command(
        TARGET texture0 POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different "$<TARGET_FILE:freeglut>" "$<TARGET_FILE_DIR:texture0>"
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${CMAKE_CURRENT_SOURCE_DIR}/tire.raw"
            "$<TARGET_FILE_DIR:texture0>"
    )

    # Visual Studio のスタートアッププロジェクトを texture0 に設定
    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT texture0)    
    set_target_properties(texture0 PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "$<TARGET_FILE_DIR:texture0>")

elseif(APPLE)
    # GLUT.framework と OpenGL.framework を検索
    find_library(OPENGL_LIBRARY OpenGL REQUIRED)
    find_library(GLUT_LIBRARY GLUT REQUIRED)
    
    target_link_libraries(texture0 PRIVATE ${OPENGL_LIBRARY} ${GLUT_LIBRARY})

    # アプリケーションバンドルとして設定し、Info.plist を指定
    set_target_properties(texture0 PROPERTIES
        MACOSX_BUNDLE TRUE
        MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/Info.plist
    )
    
    # tire.raw 及び opengl.icns をバンドル内の Resources に配置する
    set_source_files_properties(${RESOURCES} PROPERTIES MACOSX_PACKAGE_LOCATION Resources)

elseif(UNIX AND NOT APPLE)
    # pkg-config を使用して freeglut を検索
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(FREEGLUT REQUIRED freeglut)
    
    target_compile_options(texture0 PRIVATE ${FREEGLUT_CFLAGS})
    target_link_libraries(texture0 PRIVATE ${FREEGLUT_LIBRARIES} GL m)
    
    # 実行時に読み込めるよう、ビルド時に tire.raw を実行ファイルと同じ場所へコピー
    add_custom_command(
        TARGET texture0 POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_CURRENT_SOURCE_DIR}/tire.raw
            $<TARGET_FILE_DIR:texture0>
    )
endif()
