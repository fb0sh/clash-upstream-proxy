# clash-upstream-proxy
mihomo dialer上游代理

必须使用mihomo内核，支持dialer-proxy
```bash
DIRECT          [本地直连] 电脑 -> 目标网站 (用于国内流量)
跳板原生出口    [跳板直连] 电脑 -> (TCP隧道) -> 跳板机 -> 目标网站
机场原始池      [常规出海] 电脑 -> 机场节点 -> 目标网站 (未受限环境使用)
链式穿透出海    [链式代理] 电脑 -> (TCP隧道) -> 跳板机 -> 机场节点 -> 目标网站 (核心：破解UDP封锁)
```

## Server
```bash
ssserver --server-addr 0.0.0.0:8080 --encrypt-method chacha20-ietf-poly1305 --password floatctf
```

## client
```js
function main(config) {
  // ================= 1. 核心底座配置 =================
  // 将原先的校园网SOCKS5，替换为你的 FloatCTF SS 节点
  const jumpNode = {
    name: "🛡️ FloatCTF TCP跳板机",
    type: "ss",
    server: "120.95.81.151",
    port: 8080,
    cipher: "chacha20-ietf-poly1305",
    password: "password",
    udp: true,
    "udp-over-tcp": true  // 核心大招：强制把后续节点的 UDP 请求打包进 TCP 发送，破解封锁
  };

  const groupFinalName = "🚀 最终出口选择";
  const groupChainedName = "⛓️ 链式穿透池 (经跳板机出海)";
  const groupRawName = "✈️ 机场原始池 (直连机场出海)";

  // ================= 2. 基础直连规则 (防止死循环) =================
  const optimizedDirectRules = [
    "GEOSITE,private,DIRECT",
    `IP-CIDR,${jumpNode.server}/32,DIRECT,no-resolve`, // 确保跳板机自身直连，绝不能死循环
    "IP-CIDR,127.0.0.0/8,DIRECT,no-resolve",
    "GEOSITE,cn,DIRECT",
    "GEOIP,CN,DIRECT",
  ];

  // ================= 3. 提取并克隆节点 =================
  if (!config.proxies) config.proxies = [];
  config.proxies.unshift(jumpNode); // 把跳板机塞入节点列表

  const rawAirportNodes = [];
  const chainedAirportNodes = [];
  const validNodeTypes = ["ss", "vmess", "vless", "trojan", "hysteria2", "ssr", "socks5"];

  config.proxies.forEach((p) => {
    // 遍历所有订阅进来的机场节点
    if (p.name !== jumpNode.name && validNodeTypes.includes(p.type)) {
      rawAirportNodes.push(p.name);
      
      // 生成克隆节点并绑定 dialer-proxy
      const clonedNode = {
        ...p,
        name: p.name + " [跳板中转]", 
        "dialer-proxy": jumpNode.name // 🌟 核心：强制这个节点通过 FloatCTF 节点连接
      };
      config.proxies.push(clonedNode);
      chainedAirportNodes.push(clonedNode.name);
    }
  });

  // ================= 4. 分组逻辑处理 (核心结构) =================
  const oldGroupNames = config["proxy-groups"] ? config["proxy-groups"].map(g => g.name) : [];

  const masterGroups = [
    {
      name: groupFinalName,
      type: "select",
      proxies: [groupChainedName, groupRawName, jumpNode.name, "DIRECT"]
    },
    {
      name: groupChainedName,
      type: "url-test", // 自动测速选择最快的中转链
      url: "http://www.gstatic.com/generate_204",
      interval: 300,
      proxies: chainedAirportNodes.length > 0 ? chainedAirportNodes : ["DIRECT"]
    },
    {
      name: groupRawName,
      type: "select", // 保留原始机场节点的手动选择功能
      proxies: rawAirportNodes.length > 0 ? rawAirportNodes : ["DIRECT"]
    }
  ];

  if (config["proxy-groups"]) {
    config["proxy-groups"].forEach(group => {
      // 劫持原生分组：除了我们的三大主控组，其他组全部导向 groupFinalName
      if (![groupFinalName, groupChainedName, groupRawName].includes(group.name)) {
        group.proxies = [groupFinalName, "DIRECT"];
        group.type = "select"; 
      }
    });
    // 置顶我们的主控组
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
      
      if (originalPolicy.toUpperCase() === "DIRECT" || originalPolicy.toUpperCase().startsWith("REJECT")) {
        finalRules.push(rule);
      } else {
        finalRules.push(rule);
      }
    });
  }

  // 兜底规则匹配
  if (!finalRules.some(r => r.startsWith("MATCH"))) {
    finalRules.push(`MATCH,${groupFinalName}`);
  }
  config.rules = finalRules;

  return config;
}
```
