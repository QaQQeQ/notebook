```bash
#!/bin/bash

# ==============================================================================
# Script Name: setup_ssh_key_batch.sh
# Description: Rocky Linux 9.6 免密登录配置脚本 (支持批量 IP + SELinux 简化版)
# Usage: 
#   ./setup_ssh_key_batch.sh root@192.168.1.10
#   ./setup_ssh_key_batch.sh root@192.168.1.10 root@192.168.1.11
# ==============================================================================

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# 记录原始 SELinux 状态
SELINUX_STATUS=""

check_selinux_status() {
    if command -v sestatus &> /dev/null; then
        SELINUX_STATUS=$(sestatus | grep "Current mode" | awk '{print $3}')
        echo -e "${BLUE}[INFO] 检测到 SELinux 当前状态: ${SELINUX_STATUS}${NC}"
        
        if [ "$SELINUX_STATUS" == "Enforcing" ]; then
            echo -e "${YELLOW}[提示] 为了简化配置，脚本将临时关闭 SELinux...${NC}"
            setenforce 0
            if [ $? -eq 0 ]; then
                echo -e "${GREEN}[成功] SELinux 已临时设置为 Permissive 模式。${NC}"
            else
                echo -e "${RED}[错误] 无法修改 SELinux 状态，请确保以 root 权限运行。${NC}"
                exit 1
            fi
        fi
    else
        echo -e "${YELLOW}[警告] 未找到 sestatus 命令，跳过 SELinux 处理。${NC}"
    fi
}

restore_selinux_status() {
    if [ "$SELINUX_STATUS" == "Enforcing" ]; then
        echo -e "${BLUE}[INFO] 正在恢复 SELinux 状态为 Enforcing...${NC}"
        setenforce 1
        echo -e "${GREEN}[成功] SELinux 已恢复。${NC}"
    fi
}

check_and_generate_local_key() {
    if [ ! -f ~/.ssh/id_rsa ] && [ ! -f ~/.ssh/id_ed25519 ]; then
        echo -e "${YELLOW}[提示] 检测到本地未生成 SSH 密钥对，正在生成...${NC}"
        ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" || ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
        echo -e "${GREEN}[成功] 密钥对已生成。${NC}"
    else
        echo -e "${GREEN}[检查] 本地已存在 SSH 密钥。${NC}"
    fi
    chmod 700 ~/.ssh
}

# 配置单个目标的函数
setup_single_target() {
    local target=$1
    local target_user=$(echo "$target" | cut -d'@' -f1)
    local target_host=$(echo "$target" | cut -d'@' -f2)
    
    if [ -z "$target_host" ]; then
        echo -e "${RED}[错误] 无效的目标格式: $target (应为 user@host)${NC}"
        return 1
    fi

    echo -e "\n${BLUE}>>> 正在处理: $target <<<${NC}"

    # 确定公钥文件路径
    if [ -f ~/.ssh/id_ed25519.pub ]; then
        PUB_KEY_FILE=~/.ssh/id_ed25519.pub
    elif [ -f ~/.ssh/id_rsa.pub ]; then
        PUB_KEY_FILE=~/.ssh/id_rsa.pub
    else
        echo -e "${RED}[错误] 未找到本地公钥文件。${NC}"
        return 1
    fi

    echo -e "${YELLOW}[步骤 1] 在目标机器创建 .ssh 目录并设置权限...${NC}"
    ssh -o StrictHostKeyChecking=no "$target" "
        mkdir -p ~/.ssh
        chmod 700 ~/.ssh
        touch ~/.ssh/authorized_keys
        chmod 600 ~/.ssh/authorized_keys
        echo '目录权限已设置。'
    " 2>/dev/null

    if [ $? -ne 0 ]; then
        echo -e "${RED}[错误] 无法连接到 $target 。请检查网络、密码或主机名。${NC}"
        return 1
    fi

    echo -e "${YELLOW}[步骤 2] 追加公钥到目标机器...${NC}"
    ssh -o StrictHostKeyChecking=no "$target" "
        if grep -q '$(cat $PUB_KEY_FILE | cut -d' ' -f2)' ~/.ssh/authorized_keys; then
            echo '公钥已存在，跳过添加。'
        else
            cat >> ~/.ssh/authorized_keys << EOK
$(cat $PUB_KEY_FILE)
EOK
            echo '公钥添加成功。'
        fi
    " 2>/dev/null

    if [ $? -eq 0 ]; then
        echo -e "${GREEN}[成功] 免密登录配置完成！${NC}"
        # 尝试快速测试
        ssh -o StrictHostKeyChecking=no -o ConnectTimeout=3 "$target" "echo 'Test OK: $(hostname)'" 2>/dev/null
    else
        echo -e "${RED}[失败] 配置过程中出现错误。${NC}"
    fi
}

main() {
    echo "============================================================"
    echo "   Rocky Linux 9.6 SSH 免密登录配置工具 (批量版)"
    echo "============================================================"

    # 权限检查
    if [ "$EUID" -ne 0 ]; then
        echo -e "${RED}[错误] 本脚本需要 root 权限来修改 SELinux 状态。${NC}"
        echo -e "${YELLOW}[提示] 请使用: sudo $0 ${1:-user@host}${NC}"
        exit 1
    fi

    # 检查并临时关闭 SELinux
    check_selinux_status

    # 检查本地密钥
    check_and_generate_local_key

    # 获取目标列表
    declare -a TARGET_LIST
    if [ $# -gt 0 ]; then
        # 使用传入的参数
        TARGET_LIST=("$@")
    else
        # 交互式输入
        echo -e "\n${YELLOW}[提示] 请输入目标主机信息，多个主机用空格隔开${NC}"
        echo -e "${YELLOW}[示例] root@192.168.1.10 root@192.168.1.11${NC}"
        read -p "请输入: " INPUT_STR
        
        # 将输入的字符串转换为数组
        if [ -n "$INPUT_STR" ]; then
            TARGET_LIST=($INPUT_STR)
        fi
    fi

    if [ ${#TARGET_LIST[@]} -eq 0 ]; then
        echo -e "${RED}[错误] 未指定任何目标主机。${NC}"
        restore_selinux_status
        exit 1
    fi

    echo -e "\n${BLUE}[INFO] 将配置 ${#TARGET_LIST[@]} 个目标主机...${NC}"

    # 循环处理每个目标
    for target in "${TARGET_LIST[@]}"; do
        setup_single_target "$target"
    done

    # 恢复 SELinux
    restore_selinux_status
}

main "$@"

```
