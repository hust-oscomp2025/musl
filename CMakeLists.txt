cmake_minimum_required(VERSION 3.10)
project(musl C)

# musl构建配置
set(MUSL_INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/install)

# 从原来的构建命令转换
include(ExternalProject)

# 获取处理器核心数
execute_process(COMMAND nproc OUTPUT_VARIABLE NUM_PROCESSORS OUTPUT_STRIP_TRAILING_WHITESPACE)

# 创建简化的构建脚本，直接复制原始shell脚本的行为
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/build_musl.sh
"#!/bin/bash
set -ex

# 进入源码目录
cd \"${CMAKE_CURRENT_SOURCE_DIR}\"

# 确保安装目录存在
mkdir -p \"${MUSL_INSTALL_DIR}\"

# 配置musl
echo \"配置 musl...\"
./configure \\
  --prefix=\"${MUSL_INSTALL_DIR}\" \\
  --target=riscv64-unknown-elf \\
  --disable-shared \\
  CFLAGS=\"-g3 -O0 -fno-omit-frame-pointer -mabi=lp64d\" \\
  CC=riscv64-unknown-elf-gcc

# 构建musl
echo \"构建 musl...\"
make clean
make -j${NUM_PROCESSORS}

# 安装musl
echo \"安装 musl...\"
make install

echo \"musl构建和安装完成\"
")

# 设置脚本权限
execute_process(COMMAND chmod +x ${CMAKE_CURRENT_BINARY_DIR}/build_musl.sh)

# 简化ExternalProject配置，只使用一个脚本处理全部步骤
ExternalProject_Add(musl_build
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}
    DOWNLOAD_COMMAND ""
    CONFIGURE_COMMAND ""  # 不使用单独的配置命令
    BUILD_COMMAND ${CMAKE_CURRENT_BINARY_DIR}/build_musl.sh  # 使用我们的构建脚本
    INSTALL_COMMAND ""  # 不使用单独的安装命令，安装已经包含在构建脚本中
    BUILD_IN_SOURCE 0  # 不使用build-in-source，我们的脚本自己会进入源码目录
    LOG_BUILD TRUE     # 记录构建日志
)

# 创建一个接口库，以便在CMake中使用
add_library(musl_lib INTERFACE)
add_dependencies(musl_lib musl_build)

# 设置包含目录和库路径
target_include_directories(musl_lib INTERFACE 
    ${MUSL_INSTALL_DIR}/include
)

target_link_directories(musl_lib INTERFACE 
    ${MUSL_INSTALL_DIR}/lib
)

# 将头文件安装到主项目的include目录
install(DIRECTORY ${MUSL_INSTALL_DIR}/include/
        DESTINATION ${CMAKE_SOURCE_DIR}/include/musl
        FILES_MATCHING PATTERN "*.h")

# 添加自定义目标来执行清理
add_custom_target(clean_musl
    COMMAND rm -rf ${MUSL_INSTALL_DIR}
    COMMAND cd ${CMAKE_CURRENT_SOURCE_DIR} && make clean
    COMMENT "Cleaning musl build directories"
)

# 导出安装目录
set(MUSL_INSTALL_DIR ${MUSL_INSTALL_DIR} PARENT_SCOPE)