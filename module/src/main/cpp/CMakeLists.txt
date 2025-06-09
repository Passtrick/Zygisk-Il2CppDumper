cmake_minimum_required(VERSION 3.18.1)

if (NOT DEFINED MODULE_NAME)
    message(FATAL_ERROR "MODULE_NAME is not set")
else ()
    project(${MODULE_NAME})
endif ()

message("Build type: ${CMAKE_BUILD_TYPE}")

# Configuración del estándar C++
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Flags de compilación optimizados para Zygisk
set(LINKER_FLAGS "-ffixed-x18 -Wl,--hash-style=both")
set(C_FLAGS "-Werror=format -fdata-sections -ffunction-sections")
set(CXX_FLAGS "${CXX_FLAGS} -fno-exceptions -fno-rtti")

# Configuración según tipo de build
if (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(C_FLAGS "${C_FLAGS} -O2 -fvisibility=hidden -fvisibility-inlines-hidden")
    set(LINKER_FLAGS "${LINKER_FLAGS} -Wl,-exclude-libs,ALL -Wl,--gc-sections -Wl,--strip-all")
else ()
    set(C_FLAGS "${C_FLAGS} -O0")
endif ()

# Aplicar flags
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${C_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${C_FLAGS} ${CXX_FLAGS}")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${LINKER_FLAGS}")

# Directorios de inclusión
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/Dobby
    ${CMAKE_CURRENT_SOURCE_DIR}/ImGui
    ${CMAKE_CURRENT_SOURCE_DIR}/ImGui/backends
)

# Configuración específica por arquitectura para Dobby
if(ANDROID_ABI STREQUAL "armeabi-v7a")
    add_library(libdobby STATIC IMPORTED)
    set_target_properties(libdobby PROPERTIES IMPORTED_LOCATION 
        ${CMAKE_CURRENT_SOURCE_DIR}/Dobby/libraries/armeabi-v7a/libdobby.a)
    
elseif(ANDROID_ABI STREQUAL "arm64-v8a")
    add_library(libdobby STATIC IMPORTED)
    set_target_properties(libdobby PROPERTIES IMPORTED_LOCATION 
        ${CMAKE_CURRENT_SOURCE_DIR}/Dobby/libraries/arm64-v8a/libdobby.a)
endif()

# Crear la biblioteca compartida del módulo Zygisk
add_library(${MODULE_NAME} SHARED
    # Archivos principales del módulo
    imgui_manager.cpp
    native-lib.cpp
    
    # Archivos de ImGui
    ImGui/imgui.cpp
    ImGui/imgui_draw.cpp
    ImGui/imgui_widgets.cpp
    ImGui/imgui_tables.cpp
    ImGui/imgui_demo.cpp
    ImGui/backends/imgui_impl_android.cpp
    ImGui/backends/imgui_impl_opengl3.cpp
)

# Definiciones de compilación específicas para ImGui
target_compile_definitions(${MODULE_NAME} PRIVATE
    -DIMGUI_IMPL_OPENGL_ES3
)

# Flags adicionales específicos para ImGui (manteniendo compatibilidad)
target_compile_options(${MODULE_NAME} PRIVATE
    -Wno-error=format-security
    -ffunction-sections
    -fdata-sections
    -w
    -fpermissive
    -Wno-error=c++11-narrowing
    -fms-extensions
)

# Enlazar todas las dependencias
target_link_libraries(${MODULE_NAME}
    libdobby
    log
    android
    GLESv2
    EGL
    GLESv3
)

# Strip automático en release para optimizar tamaño
if (NOT CMAKE_BUILD_TYPE STREQUAL "release")
    add_custom_command(TARGET ${MODULE_NAME} POST_BUILD
            COMMAND ${CMAKE_STRIP} --strip-all --remove-section=.comment "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/lib${MODULE_NAME}.so")
endif ()