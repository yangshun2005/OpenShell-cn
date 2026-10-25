---
title: OPENSHELL
section: 1
header: OpenShell Manual
footer: openshell
date: 2026
---

# NAME

openshell - 用于管理 OpenShell 沙箱、网关注册和提供者的 CLI

# SYNOPSIS

**openshell** \[*OPTIONS*\] *COMMAND* \[*ARGS*\]

# DESCRIPTION

**openshell** 是 OpenShell 的命令行界面，OpenShell 是一个为自主 AI 代理提供安全沙箱运行时的平台。它管理网关注册、沙箱生命周期、凭据提供者、网络策略和提供者挂载。

该 CLI 通过 gRPC 与网关服务器通信。网关可作为包管理的 systemd 用户服务、Helm 部署、开发任务或云反向代理后端运行。

# COMMANDS

## 网关管理

**gateway add** *ENDPOINT* \[**--local**\] \[**--name** *NAME*\] \[**--remote** *USER@HOST*\]
:   向 CLI 注册现有网关。

**gateway remove** \[*NAME*\]
:   移除本地 CLI 注册及存储的认证令牌。不会停止或销毁网关服务。

**gateway select** \[*NAME*\]
:   列出已注册网关或切换活动网关。

**gateway info** \[**--name** *NAME*\]
:   显示网关的注册详情。

**gateway list**
:   列出已注册网关。

**gateway login**
:   重新认证云网关。

**gateway logout**
:   清除网关存储的认证凭据。

**status**
:   检查活动网关的健康状态。

## 沙箱管理

**sandbox create** \[**--from** *IMAGE*\] \[**--policy** *FILE*\] \[**--provider** *NAME*\] \[**--gpu**\] \[**--upload** *SRC:DST*\] \[**--forward** *PORT*\] \[**--** *COMMAND*\]
:   在活动网关上创建新沙箱。

**sandbox list** \[**--selector** *LABEL*\]
:   列出活动网关上的所有沙箱。

**sandbox get** *NAME*
:   显示沙箱详情。

**sandbox delete** *NAME* \| **--all**
:   删除一个或所有沙箱。

**sandbox connect** *NAME* \[**--editor** *EDITOR*\]
:   SSH 进入运行中的沙箱。

**sandbox exec** **-n** *NAME* \[**--workdir** *DIR*\] **--** *COMMAND*
:   在沙箱中执行命令。

**sandbox upload** *NAME* *LOCAL* *REMOTE*
:   上传文件到沙箱。

**sandbox download** *NAME* *REMOTE* *LOCAL*
:   从沙箱下载文件。

## 策略管理

**policy get** *SANDBOX* \[**--full**\]
:   显示沙箱的活动策略。

**policy set** *SANDBOX* **--policy** *FILE* \[**--wait**\]
:   将策略应用到沙箱。

**policy update** *SANDBOX* \[**--add-endpoint** *SPEC*\] \[**--add-allow** *RULE*\]
:   增量更新沙箱策略。

**policy list** *SANDBOX*
:   显示策略修订历史。

**policy prove** **--policy** *FILE* \[**--credentials** *FILE*\]
:   验证策略属性。

## 提供者管理

**provider create** **--name** *NAME* **--type** *TYPE* \[**--from-existing**\] \[**--credential** *KEY=VALUE*\]
:   创建凭据提供者。

**provider list**
:   列出所有提供者。

**provider get** *NAME*
:   显示提供者详情。

**provider update** *NAME* \[**--from-existing**\] \[**--credential** *KEY=VALUE*\]
:   更新提供者凭据。

**provider delete** *NAME*
:   删除提供者。

**sandbox provider attach** *SANDBOX* *PROVIDER*
:   将提供者挂载到运行中的沙箱。新进程将获得其端点绑定的凭据占位符和提供者派生的策略。

**sandbox provider detach** *SANDBOX* *PROVIDER*
:   从运行中的沙箱撤销提供者挂载。

## 其他

**logs** *SANDBOX* \[**--tail**\]
:   查看沙箱日志。

**forward start** *PORT* *SANDBOX* \[**-d**\]
:   启动到沙箱的端口转发。

**forward stop** *PORT*
:   停止端口转发。

**forward list**
:   列出活动端口转发。

**term**
:   打开实时 TUI 仪表盘。

**doctor check**
:   验证独立网关开发的本地 Docker 前置条件。对于包管理的网关，请优先使用 systemd、journalctl、kubectl 或 Helm 诊断。

**completions** *SHELL*
:   生成 shell 补全（bash、zsh、fish）。

# GLOBAL OPTIONS

**-g**, **--gateway** *NAME*
:   按名称指定目标网关。

**--gateway-endpoint** *URL*
:   直接通过 URL 连接网关。

**-h**, **--help**
:   打印帮助信息。

**-V**, **--version**
:   打印版本。

# ENVIRONMENT

**OPENSHELL_GATEWAY**
:   默认网关名称（覆盖活动网关）。

**OPENSHELL_GATEWAY_ENDPOINT**
:   直接网关 URL（绕过元数据查找）。

**ANTHROPIC_API_KEY**、**OPENAI_API_KEY**、**OPENROUTER_API_KEY**
:   自动创建提供者时发现的 API 密钥。

**GITHUB_TOKEN**、**GH_TOKEN**
:   用于提供者自动发现的 GitHub 凭据。

# FILES

*~/.config/openshell/gateways/*
:   网关元数据和 mTLS 证书。

*~/.config/openshell/active_gateway*
:   当前活动网关的名称。

# EXAMPLES

注册本地 RPM 网关并创建沙箱：

    openshell gateway add --local https://127.0.0.1:17670
    openshell sandbox create -- claude

列出沙箱并连接其中一个：

    openshell sandbox list
    openshell sandbox connect my-sandbox

从本地环境变量创建提供者：

    openshell provider create --name openai --type openai --from-existing

检查网关健康状态：

    openshell status

# SEE ALSO

**openshell-gateway**(8)

完整文档：*https://docs.nvidia.com/openshell/*

运行 **openshell** *COMMAND* **--help** 获取任意命令的详细帮助。