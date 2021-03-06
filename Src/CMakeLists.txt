cmake_minimum_required(VERSION 2.8)

if(POLICY CMP0083)
    cmake_policy(SET CMP0083 NEW)
endif()

if(POLICY CMP0020)
    cmake_policy(SET CMP0020 NEW)
endif()

project(FaceRecognizer)

# Set install data prefix
if(ANDROID)
    set(INSTALL_DATA_PREFIX assets/data)
else()
    set(INSTALL_DATA_PREFIX data)
endif()
    
OPTION(BUILD_PERFORMANCE "Set to ON to build performance)." OFF)
if(BUILD_PERFORMANCE)
    set(LIBS_DEFINES ${LIBS_DEFINES} BUILD_PERFORMANCE)
    set(SOURCES_FILES ${SOURCES_FILES} Performance.cpp)
endif()

option(USE_YUV "Use libyuv" ON)
if(USE_YUV)
    #Use: https://github.com/KangLin/libyuv
    find_package(YUV)
    if(YUV_FOUND)
        set(LIBS_DEFINES ${LIBS_DEFINES} HAVE_LIBYUV)
        set(LIBS ${LIBS} YUV)
        if(ANDROID)
            INSTALL(FILES $<TARGET_FILE:YUV>
                    DESTINATION "libs/${ANDROID_ABI}")
        else()
            INSTALL(FILES $<TARGET_FILE:YUV>
                    DESTINATION "${CMAKE_INSTALL_BINDIR}")
        endif()
    endif()
endif(USE_YUV)

option(USE_FFMPEG "Use ffmpeg" ON)
if(USE_FFMPEG)
    find_package(FFMPEG)
endif(USE_FFMPEG)

include(OpenCV)

include(Seeta)

set(SOURCES_FILES
    ${SOURCES_FILES}
    CameraQtCaptureVideoFrame.cpp
    ImageTool.cpp
    Log.cpp
    ParameterFactory.cpp
    Parameter.cpp
    ParameterDetector.cpp
    ParameterLandmark.cpp
    ParameterRecognizer.cpp
    ParameterFaceTools.cpp
    FactoryFace.cpp
    Face.cpp
    Detector.cpp
    Tracker.cpp
    Landmarker.cpp
    Recognizer.cpp
    FaceTools.cpp
    database/Database.cpp
    database/DataRegister.cpp
    database/TableRegister.cpp
    )

set(INSTALLHEADER_FILES
    ImageTool.h
    CameraQtCaptureVideoFrame.h
    FactoryFace.h
    Face.h
    Detector.h
    Tracker.h
    Landmarker.h
    Recognizer.h
    FaceTools.h
    Performance.h
    ParameterFactory.h
    Parameter.h
    ParameterDetector.h
    ParameterLandmark.h
    ParameterRecognizer.h
    ParameterFaceTools.h
    )

set(HEADER_FILES
    ${HEADER_FILES}
    ${INSTALLHEADER_FILES}
    Log.h
    database/Database.h
    database/DataRegister.h
    database/TableRegister.h
    )

#翻译
SET(TRANSLATIONS_NAME ${PROJECT_NAME})
include(${CMAKE_SOURCE_DIR}/cmake/Translations.cmake)

SET(RCC_FILES
    ../Resource/Resource.qrc
    ${TRANSLATIONS_RESOURCE_FILES})

string(TOLOWER ${CMAKE_BUILD_TYPE} BUILD_TYPE)
if(BUILD_TYPE STREQUAL "debug")
    set(LIBS_DEFINES ${LIBS_DEFINES} DEBUG)
    SET(RCC_FILES ${RCC_FILES} ../Resource/ResourceSql.qrc)
endif()

add_library(${PROJECT_NAME} ${SOURCES_FILES} ${HEADER_FILES} ${RCC_FILES})
set_target_properties(${PROJECT_NAME} PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
    ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
    PUBLIC_HEADER "${INSTALLHEADER_FILES}" # Install head files
    VERSION ${BUILD_VERSION})

target_compile_definitions(${PROJECT_NAME} PUBLIC RABBITCOMMON ${LIBS_DEFINES})
target_link_libraries(${PROJECT_NAME} PUBLIC RabbitCommon ${QT_LIBRARIES} ${LIBS})
target_include_directories(${PROJECT_NAME}
    PRIVATE ${RabbitCommon_DIR}/Src
    ${CMAKE_BINARY_DIR}
    ${INCLUDE_DIRS})

if(BUILD_VERSION)
    target_compile_definitions(${PROJECT_NAME} PRIVATE -DBUILD_VERSION="${BUILD_VERSION}")
    set_target_properties(${PROJECT_NAME} PROPERTIES VERSION ${BUILD_VERSION})
endif()

GENERATE_EXPORT_HEADER(${PROJECT_NAME})
string(TOLOWER ${PROJECT_NAME} LOWER_PROJECT_NAME)
set(INSTALLHEADER_FILES ${INSTALLHEADER_FILES} 
    ${CMAKE_CURRENT_BINARY_DIR}/${LOWER_PROJECT_NAME}_export.h)
file(COPY ${CMAKE_CURRENT_BINARY_DIR}/${LOWER_PROJECT_NAME}_export.h
    DESTINATION ${CMAKE_BINARY_DIR})

if(FFMPEG_FOUND)
    target_compile_definitions(${PROJECT_NAME} PUBLIC ${FFMPEG_DEFINITIONS})
    target_link_libraries(${PROJECT_NAME} PUBLIC ${FFMPEG_LIBRARIES})
    target_include_directories(${PROJECT_NAME} PUBLIC ${FFMPEG_INCLUDE_DIRS})
endif()

if(ANDROID)
    # Install target
    INSTALL(TARGETS ${PROJECT_NAME}
            EXPORT ${PROJECT_NAME}Config
            RUNTIME DESTINATION "libs/${ANDROID_ABI}"
                COMPONENT Runtime
            LIBRARY DESTINATION "libs/${ANDROID_ABI}"
                COMPONENT Runtime
            ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
            PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/FaceReconginzer
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
            )    
else()
    # Install target
    if(WIN32)
        INSTALL(TARGETS ${PROJECT_NAME}
            EXPORT ${PROJECT_NAME}Config
            RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
                COMPONENT Runtime
            LIBRARY DESTINATION "${CMAKE_INSTALL_BINDIR}"
                COMPONENT Runtime
            ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
            PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/FaceReconginzer
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
            )
    else()
        INSTALL(TARGETS ${PROJECT_NAME}
            EXPORT ${PROJECT_NAME}Config
            RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
                COMPONENT Runtime
            LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
                COMPONENT Runtime
            ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
            PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/FaceReconginzer
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
            )
    endif()
endif()

export(TARGETS ${PROJECT_NAME}
       APPEND FILE ${CMAKE_BINARY_DIR}/${PROJECT_NAME}Config.cmake
)
# Install cmake configure files
install(EXPORT ${PROJECT_NAME}Config
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake"
        )
write_basic_package_version_file(
    "${CMAKE_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${BUILD_VERSION}
    COMPATIBILITY AnyNewerVersion)
install(FILES "${CMAKE_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake" DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake")

# Install sql file
INSTALL(FILES ../Resource/db/database.sql
    DESTINATION ${INSTALL_DATA_PREFIX}/db
    COMPONENT Runtime)

IF( ("Release" STREQUAL CMAKE_BUILD_TYPE)
    OR ("MinSizeRel" STREQUAL CMAKE_BUILD_TYPE)
    AND WIN32 AND BUILD_SHARED_LIBS)
    IF( ("Release" STREQUAL CMAKE_BUILD_TYPE)
        OR ("MinSizeRel" STREQUAL CMAKE_BUILD_TYPE)
        AND MINGW)
        #windeployqt 分发时，是根据是否 strip 来判断是否是 DEBUG 版本,而用mingw编译时,qt没有自动 strip
        add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
            COMMAND strip "$<TARGET_FILE:${PROJECT_NAME}>"
            )
    ENDIF()

    #注意 需要把 ${QT_INSTALL_DIR}/bin 加到环境变量PATH中  
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND "${QT_INSTALL_DIR}/bin/windeployqt"
        --compiler-runtime
        --verbose 7
        "$<TARGET_FILE:${PROJECT_NAME}>"
        )
    #INSTALL(DIRECTORY "$<TARGET_FILE_DIR:${PROJECT_NAME}>/" DESTINATION "${CMAKE_INSTALL_BINDIR}")
ENDIF()

message("== OpenCV:${OpenCV_FOUND}")
message("== FFMPGE:${FFMPEG_FOUND}")
message("== LIBYUV:${YUV_FOUND}")
message("== SeetaFac:${SeetaFace_FOUND}")
message("== Build PERFORMANCE:${BUILD_PERFORMANCE}")
