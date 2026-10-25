# 内置 OCSF Schema

这些 Schema 内置自 [OCSF Schema Server](https://schema.ocsf.io/)，用于离线测试验证。

## 版本

- **OCSF v1.8.0** — 获取自 `https://schema.ocsf.io/api/1.8.0/`

## 内容

### 类（10）

- `network_activity` [4001]
- `http_activity` [4002]
- `ssh_activity` [4007]
- `process_activity` [1007]
- `detection_finding` [2004]
- `application_lifecycle` [6002]
- `device_config_state_change` [5019]
- `api_activity` [6003]
- `authentication` [3002]
- `base_event` [0]

### 对象（23）

- `metadata`、`network_endpoint`、`network_proxy`、`process`、`actor`
- `device`、`container`、`product`、`firewall_rule`、`finding_info`
- `evidences`、`http_request`、`http_response`、`url`、`attack`
- `remediation`、`connection_info`、`ai_model`、`api`、`user`、`service`、`os`、
  `resource_details`

### 配置文件（1）

- `ai_operation`

## 更新

更新到新的 OCSF 版本：

```bash
VERSION=1.8.0

for class in network_activity http_activity ssh_activity process_activity \
             detection_finding application_lifecycle device_config_state_change \
             api_activity authentication base_event; do
  curl -s "https://schema.ocsf.io/api/${VERSION}/classes/${class}" \
    | python3 -m json.tool > "classes/${class}.json"
done

for object in metadata network_endpoint network_proxy process actor device \
              container product firewall_rule finding_info evidences \
              http_request http_response url attack remediation connection_info \
              ai_model api user service os; do
  curl -s "https://schema.ocsf.io/api/${VERSION}/objects/${object}" \
    | python3 -m json.tool > "objects/${object}.json"
done

echo "${VERSION}" > VERSION
```

然后同步更新 `crates/openshell-ocsf/src/lib.rs` 中的 `OCSF_VERSION`。