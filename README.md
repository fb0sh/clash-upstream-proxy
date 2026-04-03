# clash-upstream-proxy
mihomo dialer上游代理

必须使用mihomo内核，支持dialer-proxy

```js
function main(config) {
  // ================= 1. 核心底座配置 =================
  const schoolSocksNode = {
    name: "🔒 校园网SOCKS5出口",
    server: "xxx.xxx.xxx.xxx", // 学校跳板IP
    port: 8080,
    username: "floatctf",
    password: "floatctf",
    type: "socks5",
    udp: false,
    "skip-cert-verify": true
  };

  const groupFinalName = "🚀 最终出口选择";
  const groupChainedName = "🏫 校内穿透池 (走SOCKS5)";
  const groupRawName = "✈️ 机场原始池 (直连机场)";

  // ================= 2. 基础直连规则 (防止死循环) =================
  const optimizedDirectRules = [
    "GEOSITE,private,DIRECT",
    `IP-CIDR,${schoolSocksNode.server}/32,DIRECT,no-resolve`, 
    "IP-CIDR,127.0.0.0/8,DIRECT,no-resolve",
    "GEOSITE,cn,DIRECT",
    "GEOIP,CN,DIRECT",
  ];

  // ================= 3. 提取并克隆节点 =================
  if (!config.proxies) config.proxies = [];
  config.proxies.unshift(schoolSocksNode);

  const rawAirportNodes = [];
  const chainedAirportNodes = [];
  const validNodeTypes = ["ss", "vmess", "vless", "trojan", "hysteria2", "ssr", "socks5"];

  config.proxies.forEach((p) => {
    if (p.name !== schoolSocksNode.name && validNodeTypes.includes(p.type)) {
      rawAirportNodes.push(p.name);
      const clonedNode = {
        ...p,
        name: p.name + " [S5中转]", 
        "dialer-proxy": schoolSocksNode.name
      };
      config.proxies.push(clonedNode);
      chainedAirportNodes.push(clonedNode.name);
    }
  });

  // ================= 4. 分组逻辑处理 (核心修复点) =================
  
  // A. 保存旧的代理组名称，防止 Rules 报错
  const oldGroupNames = config["proxy-groups"] ? config["proxy-groups"].map(g => g.name) : [];

  // B. 定义我们的核心管理组
  const masterGroups = [
    {
      name: groupFinalName,
      type: "select",
      proxies: [groupChainedName, groupRawName, schoolSocksNode.name, "DIRECT"]
    },
    {
      name: groupChainedName,
      type: "url-test",
      url: "http://www.gstatic.com/generate_204",
      interval: 300,
      proxies: chainedAirportNodes.length > 0 ? chainedAirportNodes : ["DIRECT"]
    },
    {
      name: groupRawName,
      type: "select",
      proxies: rawAirportNodes.length > 0 ? rawAirportNodes : ["DIRECT"]
    }
  ];

  // C. 关键修复：修改旧的代理组，让它们指向 groupFinalName
  // 这样无论规则是找 YouTube 还是 Netflix，最终都会流向我们的总开关
  if (config["proxy-groups"]) {
    config["proxy-groups"].forEach(group => {
      // 如果这个组不是我们新建的三个主控组，就清空它的成员，只指向 Final
      if (![groupFinalName, groupChainedName, groupRawName].includes(group.name)) {
        group.proxies = [groupFinalName, "DIRECT"];
        group.type = "select"; // 强制改为 select 提高稳定性
      }
    });
    // 把主控组塞到最前面
    config["proxy-groups"] = [...masterGroups, ...config["proxy-groups"]];
  } else {
    config["proxy-groups"] = masterGroups;
  }

  // ================= 5. 规则逻辑重定向 =================
  const finalRules = [...optimizedDirectRules];
  if (config.rules) {
    config.rules.forEach((rule) => {
      const parts = rule.split(",");
      if (parts.length < 2) return;
      const originalPolicy = parts[parts.length - 1].trim();
      
      // 如果规则原本就是直连或拒绝，保留之
      if (originalPolicy.toUpperCase() === "DIRECT" || originalPolicy.toUpperCase().startsWith("REJECT")) {
        finalRules.push(rule);
      } else {
        // 如果原本是走某个组，保留原样（因为我们在第4步已经把那些组重定向到 Final 了）
        finalRules.push(rule);
      }
    });
  }

  // 兜底规则
  if (!finalRules.some(r => r.startsWith("MATCH"))) {
    finalRules.push(`MATCH,${groupFinalName}`);
  }
  config.rules = finalRules;

  return config;
}
```
