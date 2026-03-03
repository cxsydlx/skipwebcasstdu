# skipcasstdu
用openwrt连接石铁大的校园网（student）
---

## 〇、准备工作
在校园网登录页面（如 `1.1.1.1`）抓包获取 `quickauth` 参数：

1. 打开 Edge 浏览器，进入校园网登录页面
2. 按 `F12` 打开开发者工具 → 网络
3. 勾选 **保留日志**
4. 正常登录校园网
5. 在请求列表中找到：`quickauth.do?xxx`
6. 切换到 **负载 / 参数**，复制所有字段备用

---

## 一、准备信息（抓包后填写）
```text
认证账号：

认证密码：

无线接口：

本机 MAC：

AC 控制器 IP（wlanacIp）：

认证地址：
```

---

## 二、在 OpenWrt 终端执行

### 1. 一键创建脚本
**你需要对应你的抓包信息修改以下第11到第14行所有字符串参数，wlanaclp参数在第74行，也需要修改**

```bash
# 一键创建带详细注释的校园网认证脚本
cat > /root/campus_auto_login.sh << 'END_OF_SCRIPT'
#!/bin/sh
# ==============================================
# 脚本核心功能：校园网Portal自动认证（适配phy1-sta0接口）
# 适配环境：OpenWrt，接口phy1-sta0，MAC a2:21:aa:c1:cd:0b
# 核心逻辑：获取接口IP→生成动态参数→发送认证请求→验证结果
# ==============================================
 
# --------------------------
# 第一步：配置项（仅需维护这里）
# --------------------------
USER_ID="00000000"          # 校园网账号（固定）
PASSWD="000000"             # 校园网密码（固定）
CLONE_MAC="00:00:00:00:00:00" # 真实接口MAC（从ip addr获取）
WLAN_IFACE="phy1-example"      # 真实无线客户端接口（从ip addr获取）
 
# --------------------------
# 第二步：打印脚本启动信息（可视化回执）
# --------------------------
echo -e "\n========================================"
echo "📅 校园网认证脚本启动 | $(date '+%Y-%m-%d %H:%M:%S')"
echo "========================================"
 
# --------------------------
# 第三步：打印基础配置（方便核对）
# --------------------------
echo -e "\n🔧 基础配置信息（核对是否正确）："
echo "   - 认证账号：$USER_ID"          # 打印账号
echo "   - 接口MAC：$CLONE_MAC"         # 打印MAC
echo "   - 无线接口：$WLAN_IFACE"       # 打印接口名
 
# --------------------------
# 第四步：自动获取接口IP（核心步骤）
# --------------------------
# ip addr show：读取接口的IP信息
# grep 'inet '：筛选出IPv4地址行
# awk '{print $2}'：提取IP+子网掩码段（如10.2.96.231/23）
# cut -d '/' -f1：去掉子网掩码，只保留纯IP
WAN_IP=$(ip addr show $WLAN_IFACE | grep 'inet ' | awk '{print $2}' | cut -d '/' -f1)
 
# 打印获取到的IP（直观确认）
echo -e "\n🌐 接口IP获取结果："
echo "   - 成功获取 $WLAN_IFACE 接口IP：$WAN_IP"
 
# --------------------------
# 第五步：生成动态参数（适配校园网要求）
# --------------------------
# date +%s%3N：生成当前时间的毫秒级时间戳（校园网必填）
TIMESTAMP=$(date +%s%3N)
# cat /proc/sys/kernel/random/uuid：生成随机UUID（校园网必填）
UUID=$(cat /proc/sys/kernel/random/uuid)
 
# 打印动态参数（方便排查）
echo -e "\n🎲 动态认证参数（校园网要求）："
echo "   - 毫秒时间戳(timestamp)：$TIMESTAMP"
echo "   - 唯一标识(uuid)：$UUID"
 
# --------------------------
# 第六步：发送认证请求（核心操作）
# --------------------------
echo -e "\n🚀 开始向校园网网关发送认证请求..."
# curl参数说明：
# -G：以GET方式发送请求（适配校园网quickauth.do）
# --data-urlencode：对参数进行URL编码（避免特殊字符报错）
# -H：设置请求头（模拟浏览器，避免校园网拦截）
# -b：设置Cookie（携带macAuth，校园网绑定MAC用）
# --insecure：忽略HTTPS证书（校园网是HTTP，不影响）
# -s：静默模式（不打印curl自身的调试信息）
RESULT=$(curl "http://192.168.251.75/quickauth.do" \
  -G \
  --data-urlencode "userid=${USER_ID}" \
  --data-urlencode "passwd=${PASSWD}" \
  --data-urlencode "wlanuserip=${WAN_IP}" \
  --data-urlencode "wlanacname=NFV-BASE" \
  --data-urlencode "wlanacIp=000.000.000.000" \
  --data-urlencode "vlan=0" \
  --data-urlencode "mac=${CLONE_MAC}" \
  --data-urlencode "version=0" \
  --data-urlencode "portalpageid=1" \
  --data-urlencode "timestamp=${TIMESTAMP}" \
  --data-urlencode "uuid=${UUID}" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36 Edg/145.0.0.0" \
  -H "X-Requested-With: XMLHttpRequest" \
  -b "macAuth=${CLONE_MAC}" \
  --insecure \
  -s)
 
# --------------------------
# 第七步：打印认证返回结果（排查关键）
# --------------------------
echo -e "\n📜 校园网网关完整返回数据："
echo "$RESULT"  # 打印原始JSON，方便定位错误码
 
# --------------------------
# 第八步：验证认证结果（核心判断）
# --------------------------
echo -e "\n✅ 认证结果验证："
# grep -q '"code":"0"'：静默检查返回结果中是否有code=0（认证成功标识）
if echo "$RESULT" | grep -q '"code":"0"'; then
  echo "   ✅ 校园网认证成功！"
  # 测试外网连通性（ping百度2次，静默模式）
  if ping -c 2 baidu.com >/dev/null 2>&1; then
    # 提取平均延迟并打印
    PING_LATENCY=$(ping -c 2 baidu.com | grep 'avg' | awk -F '/' '{print $5}')
    echo "   🌐 外网访问正常，百度平均延迟：${PING_LATENCY} ms"
  else
    echo "   ⚠️  认证成功但外网不通，请检查路由/防火墙配置！"
  fi
else
  echo "   ❌ 校园网认证失败！"
  echo "   📌 失败原因排查："
  echo "      1. 账号/密码错误（当前：$USER_ID/$PASSWD）"
  echo "      2. MAC地址不匹配（当前：$CLONE_MAC）"
  echo "      3. IP地址不在校园网段（当前：$WAN_IP）"
fi
 
# --------------------------
# 第九步：脚本结束收尾
# --------------------------
echo -e "\n========================================"
echo "📝 认证脚本执行完成 | $(date '+%Y-%m-%d %H:%M:%S')"
echo "========================================\n"
END_OF_SCRIPT
 
# 一键赋予脚本执行权限
#chmod +x /root/campus_auto_login.sh
 
# 验证脚本创建成功（打印前10行确认）
echo -e "\n✅ 带详细注释的脚本创建成功！脚本前10行内容："
head -10 /root/campus_auto_login.sh
```

### 2. 赋予执行权限
```bash
chmod +x /root/campus_auto_login.sh
```

### 3. 测试脚本
```bash
/root/campus_auto_login.sh
```

### 4. 设置开机自启
```bash
# 一键添加开机自启任务
(crontab -l 2>/dev/null; echo "@reboot sleep 15 && /root/campus_auto_login.sh") | crontab -
 
# 验证自启任务是否添加成功
echo -e "\n✅ 开机自启任务已添加！当前计划任务列表："
crontab -l
```

### 5. 查看自启是否生效
```bash
crontab -l
```

## 三、版权声明
本文为原创教程，遵循 **CC 4.0 BY-SA** 版权协议，转载请注明出处。
版权声明：本文为CSDN博主「HOPE71」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/2403_88761894/article/details/158622725
