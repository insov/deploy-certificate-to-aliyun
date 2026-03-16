# GitHub Actions 脚本说明文档

## 1. 脚本概述

本GitHub Actions脚本用于**自动续期和部署SSL证书到阿里云DCDN**，实现了从证书申请、获取到部署的全自动化流程。

### 核心功能
- 自动从Let's Encrypt获取/续期SSL证书
- 支持多域名批量处理
- 自动将证书部署到阿里云DCDN
- 定期执行，确保证书始终有效

## 2. 工作流文件结构

工作流文件路径：`.github/workflows/action.yml`

### 2.1 工作流名称
```yaml
name: Auto Renew and Deploy SSL Certificates
```

### 2.2 触发条件

该工作流在以下两种情况下触发：

1. **代码推送触发**：当代码推送到`main`分支时执行
   ```yaml
   push:
     branches:
       - main
   ```

2. **定时触发**：每两个月的第20天执行一次（用于自动续期证书）
   ```yaml
   schedule:
     - cron: '0 0 20 */2 *' # 每两个月的第二十天执行一次
   ```

### 2.3 作业配置

```yaml
jobs:
  renew-deploy-cert:
    runs-on: ubuntu-latest
```

- 作业名称：`renew-deploy-cert`
- 运行环境：`ubuntu-latest`

## 3. 工作流程步骤

### 3.1 步骤1：检出代码
```yaml
- name: Checkout repository
  uses: actions/checkout@v2
```
- 使用`actions/checkout@v2`动作将代码仓库检出到GitHub Actions运行器中

### 3.2 步骤2：设置Python环境
```yaml
- name: Set up Python
  uses: actions/setup-python@v2
  with:
    python-version: '3.8'
```
- 使用`actions/setup-python@v2`动作安装Python 3.8环境

### 3.3 步骤3：安装acme.sh
```yaml
- name: Install acme.sh
  env:
    EMAIL: ${{ secrets.EMAIL}}
  run: |
    sudo apt-get update
    sudo apt-get install -y socat
    curl https://get.acme.sh | sh -s email="${EMAIL}"
```
- 更新系统包列表
- 安装依赖`socat`
- 安装`acme.sh`工具，用于获取/续期Let's Encrypt证书
- 使用GitHub Secrets中的`EMAIL`变量作为Let's Encrypt注册邮箱

### 3.4 步骤4：准备acme.sh凭证
```yaml
- name: Prepare acme.sh credentials
  run: |
    mkdir -p ~/.acme.sh
    IFS=',' read -r -a domain_array <<< "${{ secrets.DOMAINS }}"
    for domain in "${domain_array[@]}"; do
      mkdir -p ~/certs/${domain}
    done
```
- 创建必要的目录结构
- 从GitHub Secrets中读取`DOMAINS`变量，创建对应的证书存储目录

### 3.5 步骤5：获取SSL证书
```yaml
- name: Obtain SSL Certificates
  env:
    DOMAINS: ${{ secrets.DOMAINS }}
    Ali_Key: ${{ secrets.ALIYUN_ACCESS_KEY_ID }}
    Ali_Secret: ${{ secrets.ALIYUN_ACCESS_KEY_SECRET }}
  run: |
    IFS=',' read -r -a domain_array <<< "${DOMAINS}"
    for domain in "${domain_array[@]}"; do
      ~/.acme.sh/acme.sh --issue --dns dns_ali -d "*.${domain}" \
      --key-file ~/certs/${domain}/privkey.pem --fullchain-file ~/certs/${domain}/fullchain.pem
    done
```
- 使用`acme.sh`工具，通过阿里云DNS验证方式获取通配符证书
- 证书文件保存路径：
  - 私钥：`~/certs/${domain}/privkey.pem`
  - 完整证书链：`~/certs/${domain}/fullchain.pem`

### 3.6 步骤6：安装Python依赖
```yaml
- name: Install Python dependencies
  run: pip install -r requirements.txt
```
- 安装Python脚本所需的依赖包

### 3.7 步骤7：上传证书到阿里云DCDN
```yaml
- name: Upload certificates to Alibaba Cloud DCDN
  env:
    ALIYUN_ACCESS_KEY_ID: ${{ secrets.ALIYUN_ACCESS_KEY_ID }}
    ALIYUN_ACCESS_KEY_SECRET: ${{ secrets.ALIYUN_ACCESS_KEY_SECRET }}
    DOMAINS: ${{ secrets.DOMAINS }}
    ALIYUN_DCDN_DOMAINS: ${{ secrets.ALIYUN_DCDN_DOMAINS }}
  
  run: python upload_certs_to_aliyun.py
```
- 运行Python脚本，将获取到的证书上传到阿里云DCDN
- 使用GitHub Secrets中配置的阿里云访问密钥

### 3.8 步骤8：清理工作
```yaml
- name: Clean up
  env:
    DOMAINS: ${{ secrets.DOMAINS }}
  run: |
    IFS=',' read -r -a domain_array <<< "${DOMAINS}"
    for domain in "${domain_array[@]}"; do
      rm -rf ~/certs/${domain}
    done
```
- 清理临时生成的证书文件

## 4. Python脚本说明

### 4.1 脚本功能

`upload_certs_to_aliyun.py`脚本用于将获取到的SSL证书上传到阿里云DCDN服务。

### 4.2 核心函数

#### `get_env_var(key)`
- 功能：获取环境变量值，若不存在则抛出异常
- 参数：`key` - 环境变量名称
- 返回值：环境变量值

#### `file_exists_and_not_empty(file_path)`
- 功能：检查文件是否存在且不为空
- 参数：`file_path` - 文件路径
- 返回值：布尔值

#### `upload_certificate(client, domain_name, cert_path, key_path)`
- 功能：上传证书到阿里云DCDN
- 参数：
  - `client` - 阿里云SDK客户端
  - `domain_name` - DCDN加速域名
  - `cert_path` - 证书文件路径
  - `key_path` - 私钥文件路径

#### `main()`
- 功能：脚本主函数，协调证书上传流程

### 4.3 执行流程

1. 获取环境变量：阿里云访问密钥、域名列表、DCDN域名列表
2. 初始化阿里云SDK客户端
3. 遍历域名列表，将每个域名的证书上传到对应的DCDN域名

## 5. 环境变量配置

| 环境变量名称 | 类型 | 说明 |
|------------|------|------|
| `EMAIL` | Secret | Let's Encrypt注册邮箱，用于接收证书到期提醒 |
| `DOMAINS` | Secret | 要申请证书的主域名，多个域名用英文逗号分隔<br>示例：`example.com,test.com` |
| `ALIYUN_ACCESS_KEY_ID` | Secret | 阿里云访问密钥ID，需要具有DNS管理和DCDN管理权限 |
| `ALIYUN_ACCESS_KEY_SECRET` | Secret | 阿里云访问密钥Secret |
| `ALIYUN_DCDN_DOMAINS` | Secret | 阿里云DCDN域名，与`DOMAINS`一一对应，用英文逗号分隔<br>示例：`dcdn.example.com,dcdn.test.com` |

## 6. 阿里云权限配置

### 6.1 RAM用户权限

需要为阿里云RAM用户配置以下权限：

1. **DNS管理权限**：用于验证域名所有权（AliyunDNSFullAccess）
2. **DCDN管理权限**：用于上传和配置证书（AliyunDCDNFullAccess）

### 6.2 DNS验证

脚本使用阿里云DNS API自动添加TXT记录进行域名验证，无需手动干预。

## 7. 证书续期机制

- 脚本每两个月的第20天自动执行一次，提前续期证书
- Let's Encrypt证书有效期为90天，每两个月续期一次可确保证书始终有效
- 当代码推送到main分支时也会触发执行，方便手动触发证书更新

## 8. 使用方法

### 8.1 首次配置

1. 克隆仓库到本地
2. 在GitHub仓库的`Settings` > `Secrets and variables` > `Actions`中配置所需的环境变量
3. 推送代码到main分支，触发首次执行

### 8.2 手动触发

可以通过以下方式手动触发工作流：

1. 推送代码到main分支
2. 在GitHub仓库的`Actions`标签页中手动触发工作流

### 8.3 查看执行日志

在GitHub仓库的`Actions`标签页中可以查看工作流的执行日志，了解每一步的执行情况。

## 9. 注意事项

1. **域名对应关系**：`DOMAINS`和`ALIYUN_DCDN_DOMAINS`中的域名必须一一对应，顺序不能出错
2. **权限配置**：阿里云RAM用户必须具有足够的权限，否则会导致证书获取或上传失败
3. **证书有效期**：Let's Encrypt证书有效期为90天，脚本每两个月执行一次，确保证书始终有效
4. **DNS解析**：确保域名的DNS服务器使用阿里云DNS，否则无法自动添加TXT记录进行验证
5. **存储安全**：证书文件仅在工作流执行期间临时存储，执行完成后会被清理，确保安全

## 10. 故障排除

### 10.1 证书获取失败

- 检查阿里云访问密钥是否正确
- 检查RAM用户是否具有DNS管理权限
- 检查域名是否使用阿里云DNS

### 10.2 证书上传失败

- 检查RAM用户是否具有DCDN管理权限
- 检查`DOMAINS`和`ALIYUN_DCDN_DOMAINS`的对应关系是否正确
- 检查DCDN域名是否存在且状态正常

### 10.3 工作流执行失败

- 查看GitHub Actions执行日志，定位具体失败步骤
- 检查环境变量配置是否正确
- 检查Python脚本是否存在语法错误

## 11. 依赖说明

### 11.1 Python依赖

依赖文件：`requirements.txt`

内容：
```
aliyun-python-sdk-core
```

### 11.2 系统依赖

- `socat`：用于acme.sh的HTTP验证
- `curl`：用于下载acme.sh

## 12. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0 | 2024-01-01 | 初始版本，实现基本功能 |

## 13. 许可证

本项目采用MIT许可证，详见`LICENSE`文件。

## 14. 联系方式

如有问题或建议，请通过GitHub Issues提交。
