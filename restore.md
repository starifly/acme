  | 目录          | 路径                 | 用途                                    |
  |-------------|--------------------|---------------------------------------|
  | .acme.sh    | /root/.acme.sh/    | acme.sh 程序本体 + 证书源文件 + 账户信息 + 域名配置    |
  | .cert       | /root/.cert/       | 安装后的证书文件 (private.key, fullchain.crt) |
  | .ssl_config | /root/.ssl_config/ | 域名配置 (域名、CF Token、邮箱等)                |

  ---
  恢复步骤

  第 1 步：安装依赖

  # Debian/Ubuntu
  apt update && apt install -y cron curl jq socat

  # CentOS
  yum install -y cronie curl jq socat

  确保 cron 服务启动：

  # Debian/Ubuntu
  systemctl enable cron && systemctl start cron

  # CentOS
  systemctl enable crond && systemctl start crond

  第 2 步：恢复备份目录

  将三个目录恢复到 /root/ 下：

  cp -a /path/to/backup/.acme.sh  /root/.acme.sh
  cp -a /path/to/backup/.cert     /root/.cert
  cp -a /path/to/backup/.ssl_config /root/.ssl_config

  用 cp -a 保留文件权限和属性。

  第 3 步：设置证书文件权限

  # 遍历所有域名的证书目录，确保权限正确
  for dir in /root/.cert/*/; do
    chmod 600 "$dir/private.key" 2>/dev/null
    chmod 644 "$dir/fullchain.crt" 2>/dev/null
  done

  第 4 步：恢复 acme.sh 的 shell 环境集成

  acme.sh 安装时会在 .bashrc 中加入 source 行，重装系统后丢失了，需要手动添加回来：

  echo '. "/root/.acme.sh/acme.sh.env"' >> /root/.bashrc
  source /root/.bashrc

  第 5 步：验证 acme.sh 可正常运行

  /root/.acme.sh/acme.sh --list

  应该能看到你之前申请的域名证书列表。如果报错，检查 .acme.sh/acme.sh 是否有执行权限：

  chmod +x /root/.acme.sh/acme.sh

  第 6 步：重新注册 install-cert（关键步骤）

  acme.sh 的 --install-cert 命令会在内部配置中记录证书安装路径。由于是恢复的备份，这些记录应该还在 .acme.sh/<域名>/ 目录下的配置文件中。验证一下：

  # 替换为你的实际域名
  DOMAIN="your.domain.com"
  cat /root/.acme.sh/${DOMAIN}_ecc/${DOMAIN}.conf | grep -E "Le_RealKeyPath|Le_RealFullChainPath"

  应该看到类似：

  Le_RealKeyPath='/root/.cert/your.domain.com/private.key'
  Le_RealFullChainPath='/root/.cert/your.domain.com/fullchain.crt'

  如果这些路径正确且存在，续期时 acme.sh 会自动把新证书复制到这些位置。

  如果路径为空或不正确，需要重新执行 install-cert：

  DOMAIN="your.domain.com"
  /root/.acme.sh/acme.sh --install-cert -d "$DOMAIN" \
    --key-file "/root/.cert/$DOMAIN/private.key" \
    --fullchain-file "/root/.cert/$DOMAIN/fullchain.crt" \
    --ecc

  第 7 步：恢复 cron 自动续期任务

  脚本中定义的 cron 任务是：

  crontab -l 2>/dev/null | grep -q "acme.sh.*--cron" || \
    (crontab -l 2>/dev/null; echo "0 0 * * * /root/.acme.sh/acme.sh --cron --home /root/.acme.sh > /dev/null") | crontab -

  执行后验证：

  crontab -l

  应看到包含 acme.sh --cron 的行。

  第 8 步：手动测试续期是否正常

  /root/.acme.sh/acme.sh --cron --home /root/.acme.sh

  如果证书未到期，会提示 "Skip, Next renewal time is ..."，这表示一切正常。

  如果想强制续期测试（可选）：

  DOMAIN="your.domain.com"
  /root/.acme.sh/acme.sh --renew -d "$DOMAIN" --force --ecc

  ---
  总结检查清单

  | 检查项               | 命令                                                  |
  |-------------------|-----------------------------------------------------|
  | acme.sh 可运行       | /root/.acme.sh/acme.sh --version                    |
  | 证书列表正常            | /root/.acme.sh/acme.sh --list                       |
  | 证书文件存在            | ls -la /root/.cert/你的域名/                            |
  | install-cert 路径正确 | grep Le_Real /root/.acme.sh/你的域名_ecc/你的域名.conf      |
  | cron 任务存在         | crontab -l | grep acme                              |
  | 续期测试通过            | /root/.acme.sh/acme.sh --cron --home /root/.acme.sh |
  | cron 服务运行中        | systemctl status cron                               |

  按以上步骤完成后，SSL 证书即恢复完成，无需重新申请，且自动续期会照常工作。
