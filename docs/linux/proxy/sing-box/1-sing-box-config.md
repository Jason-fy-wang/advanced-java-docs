---
tags:
  - sing-box
  - config
  - sing-box-config
---


```json
{
  "log": {
    "level": "debug",
    "timestamp": true
  },
   "dns": {
    "servers": [
    {
      "type": "udp",
      "tag": "dns_resolver",
      "server": "223.5.5.5"
    },
    {
      "type": "h3",
      "tag": "dns_direct",
      "server": "dns.alidns.com",
      "path": "/dns-query",
      "domain_resolver": {
        "server": "dns_resolver",
        "strategy": "ipv4_only"
      }
    },
    {
      "type": "https",
      "tag": "dns_proxy",
      "server": "1.1.1.1",
      "server_port": 443,
      "path": "/dns-query",
      "domain_resolver": {
        "server": "dns_resolver",
        "strategy": "ipv4_only"
      },
      "detour": "proxy"
    },
    {
        "predefined": {
          "dns.google": [
            "8.8.8.8",
            "8.8.4.4",
            "2001:4860:4860::8888",
            "2001:4860:4860::8844"
          ],
          "dns.alidns.com": [
            "223.5.5.5",
            "223.6.6.6",
            "2400:3200::1",
            "2400:3200:baba::1"
          ],
          "one.one.one.one": [
            "1.1.1.1",
            "1.0.0.1",
            "2606:4700:4700::1111",
            "2606:4700:4700::1001"
          ],
          "1dot1dot1dot1.cloudflare-dns.com": [
            "1.1.1.1",
            "1.0.0.1",
            "2606:4700:4700::1111",
            "2606:4700:4700::1001"
          ],
          "cloudflare-dns.com": [
            "104.16.249.249",
            "104.16.248.249",
            "2606:4700::6810:f8f9",
            "2606:4700::6810:f9f9"
          ],
          "dns.cloudflare.com": [
            "162.159.61.8",
            "172.64.41.8",
            "2a06:98c1:52::8",
            "2803:f800:53::8"
          ],
          "dot.pub": [
            "1.12.12.12",
            "120.53.53.53"
          ],
          "doh.pub": [
            "1.12.12.12",
            "120.53.53.53"
          ],
          "dns.quad9.net": [
            "9.9.9.9",
            "149.112.112.112",
            "2620:fe::fe",
            "2620:fe::9"
          ],
          "dns.yandex.net": [
            "77.88.8.8",
            "77.88.8.1",
            "2a02:6b8::feed:0ff",
            "2a02:6b8:0:1::feed:0ff"
          ],
          "dns.sb": [
            "45.11.45.11",
            "185.222.222.222",
            "2a09::",
            "2a11::"
          ],
          "dns.umbrella.com": [
            "208.67.220.220",
            "208.67.222.222",
            "2620:119:35::35",
            "2620:119:53::53"
          ],
          "dns.sse.cisco.com": [
            "208.67.220.220",
            "208.67.222.222",
            "2620:119:35::35",
            "2620:119:53::53"
          ],
          "engage.cloudflareclient.com": [
            "162.159.192.1",
            "2606:4700:d0::a29f:c001"
          ]
        },
        "type": "hosts",
        "tag": "hosts_dns"
      }
  ],
    "rules": [
      {
        "domain_suffix": [
          "docker.io",
          "docker.com",
          "dockerusercontent.com",
          "cloudflare.docker.com",
          "production.cloudflare.docker.com",
          "registry-1.docker.io",
          "auth.docker.io"
        ],
        "action": "route",
        "server": "dns_proxy"
      },
      {
        "process_name": [
          "TencentMeeting",
          "NemoDesktop",
          "ToDesk",
          "ToDesk_Service",
          "WeChat",
          "Tailscale",
          "wireguard-go",
          "Tunnelblick",
          "softwareupdated",
          "kubectl"
        ],
        "action": "route",
        "server": "dns_direct"
      },
      {
        "domain_suffix": [
          "icloudnative.io",
          "fuckcloudnative.io",
          "shturl.cc",
          "cdn.jsdelivr.net"
        ],
        "action": "route",
        "server": "dns_direct"
      },
      {
        "process_name": [
          "DropboxMacUpdate",
          "Dropbox"
        ],
        "action": "route",
        "server": "dns_proxy"
      },
      {
        "package_name": [
          "com.google.android.youtube",
          "com.android.vending",
          "org.telegram.messenger",
          "org.telegram.plus"
        ],
        "action": "route",
        "server": "dns_proxy"
      },
      {
        "rule_set": [
          "geosite-cn"
        ],
        "action": "route",
        "server": "dns_proxy"
      }
    ],
    "final": "dns_direct",
    "strategy": "ipv4_only"
  },
  "inbounds": [
    {
      "type": "tun",
       "address": [
        "198.18.0.1/16"
      ],
      "auto_route": true,
      "strict_route": true,
      "endpoint_independent_nat": false,
      "stack": "system",
      "exclude_package": [
        "cmb.pb",
        "cn.gov.pbc.dcep",
        "com.MobileTicket",
        "com.adguard.android",
        "com.ainemo.dragoon",
        "com.alibaba.android.rimet",
        "com.alicloud.databox",
        "com.amazing.cloudisk.tv",
        "com.autonavi.minimap",
        "com.bilibili.app.in",
        "com.bishua666.luxxx1",
        "com.cainiao.wireless",
        "com.chebada",
        "com.chinamworld.main",
        "com.cmbchina.ccd.pluto.cmbActivity",
        "com.coolapk.market",
        "com.ctrip.ct",
        "com.dianping.v1",
        "com.douban.frodo",
        "com.eg.android.AlipayGphone",
        "com.farplace.qingzhuo",
        "com.hanweb.android.zhejiang.activity",
        "com.leoao.fitness",
        "com.lucinhu.bili_you",
        "com.mikrotik.android.tikapp",
        "com.moji.mjweather",
        "com.motorola.cn.calendar",
        "com.motorola.cn.lrhealth",
        "com.netease.cloudmusic",
        "com.sankuai.meituan",
        "com.sina.weibo",
        "com.smartisan.notes",
        "com.sohu.inputmethod.sogou.moto",
        "com.sonelli.juicessh",
        "com.ss.android.article.news",
        "com.ss.android.lark",
        "com.ss.android.ugc.aweme",
        "com.tailscale.ipn",
        "com.taobao.idlefish",
        "com.taobao.taobao",
        "com.tencent.mm",
        "com.tencent.mp",
        "com.tencent.soter.soterserver",
        "com.tencent.wemeet.app",
        "com.tencent.weread",
        "com.tencent.wework",
        "com.ttxapps.wifiadb",
        "com.unionpay",
        "com.unnoo.quan",
        "com.wireguard.android",
        "com.xingin.xhs",
        "com.xunmeng.pinduoduo",
        "com.zui.zhealthy",
        "ctrip.android.view",
        "io.kubenav.kubenav",
        "org.geekbang.geekTime",
        "tv.danmaku.bili"
      ]
    },
    {
      "type": "socks",
      "tag": "socks-in",
      "listen": "::",
      "listen_port": 5353
    },
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 7890
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "type": "vless",
      "server": "xxxx.hosts",
      "server_port": 443,
      "domain_resolver": {
        "server": "dns_direct",
        "strategy": "ipv4_only"
      },
      "uuid": "23619c65-7147-49a6-bbc8-6b41f32bfcee",
      "flow": "",
      "tls": {
        "enabled": true,
        "server_name": "xxxx.hosts.shop",
        "insecure": false
      },
      "transport": {
        "type": "ws",
        "path": "/yfjc/jp",
        "headers": {
          "Host": "xxxx.hosts.shop"
        }
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    }
  ],
  "route": {
     "auto_detect_interface": true,
     "default_domain_resolver": {
        "server": "dns_direct",
        "strategy": "ipv4_only"
      },
    "rules": [
       {
      "inbound": "tun-in",
      "action": "sniff"
    },
    {
      "inbound": "mixed-in",
      "outbound": "proxy"
    },
    {
      "domain": [
        "cf.yfjc.sbs",
        "yfjc-jp3.wangpan.shop"
      ],
      "outbound": "direct"
    },
     {
      "inbound": "socks-in",
      "outbound": "proxy"
      },
      {
        "domain_suffix": [
          "registry-1.docker.io",
          "auth.docker.io",
          "docker.io",
          "docker.com",
          "dockerusercontent.com"
        ],
        "outbound": "proxy"
      },
      {
        "port": [
          53
        ],
        "process_name": [
        ],
        "action": "hijack-dns"
      },
      {
        "outbound": "direct",
        "process_name": [
        ]
      },
      {
        "port": [
          53
        ],
        "action": "hijack-dns"
      },
      {
        "network": [
          "udp"
        ],
        "port": [
          443
        ],
        "action": "reject"
      },
      {
        "outbound": "proxy",
        "rule_set": [
          "geosite-google"
        ]
      },
      {
        "outbound": "direct",
        "ip_is_private": true
      },
      {
        "outbound": "direct",
        "rule_set": [
          "geosite-private"
        ]
      },
      {
        "outbound": "direct",
        "ip_cidr": [
          "223.5.5.5",
          "223.6.6.6",
          "2400:3200::1",
          "2400:3200:baba::1",
          "119.29.29.29",
          "1.12.12.12",
          "120.53.53.53",
          "2402:4e00::",
          "2402:4e00:1::",
          "180.76.76.76",
          "2400:da00::6666",
          "114.114.114.114",
          "114.114.115.115",
          "114.114.114.119",
          "114.114.115.119",
          "114.114.114.110",
          "114.114.115.110",
          "180.184.1.1",
          "180.184.2.2",
          "101.226.4.6",
          "218.30.118.6",
          "123.125.81.6",
          "140.207.198.6",
          "1.2.4.8",
          "210.2.4.8",
          "52.80.66.66",
          "117.50.22.22",
          "2400:7fc0:849e:200::4",
          "2404:c2c0:85d8:901::4",
          "117.50.10.10",
          "52.80.52.52",
          "2400:7fc0:849e:200::8",
          "2404:c2c0:85d8:901::8",
          "117.50.60.30",
          "52.80.60.30"
        ]
      },
      {
        "outbound": "direct",
        "domain_suffix": [
          "alidns.com",
          "doh.pub",
          "dot.pub",
          "360.cn",
          "onedns.net"
        ]
      },
      {
        "outbound": "direct",
        "rule_set": [
          "geoip-cn"
        ]
      },
      {
        "outbound": "direct",
        "rule_set": [
          "geosite-cn"
        ]
      }
    ],
    "rule_set": [
      {
        "tag": "geosite-google",
        "type": "local",
        "format": "binary",
        "path": "/etc/sing-box/rules/geosite-google.srs"
      },
      {
        "tag": "geosite-private",
        "type": "local",
        "format": "binary",
        "path": "/etc/sing-box/rules/geosite-private.srs"
      },
      {
        "tag": "geosite-cn",
        "type": "local",
        "format": "binary",
        "path": "/etc/sing-box/rules/geosite-cn.srs"
      },
      {
        "tag": "geoip-cn",
        "type": "local",
        "format": "binary",
        "path": "/etc/sing-box/rules/geoip-cn.srs"
      }
    ],
    "final": "proxy"
  },
  "experimental": {
    "cache_file": {
      "enabled": true,
      "path": "/var/lib/sing-box/cache.db"
    }
  }
}
```


