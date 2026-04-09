# Mainland China Root Domains — Hostname Routes Reference

A curated list of mainland China root domains for use with Cloudflare Warp Reverse Hostname Routes. Adding a root domain automatically covers all its subdomains via SNI-based matching.

> **Usage:** Copy domain groups relevant to your use case into the bulk upload script in [warp-reverse-solution.md](warp-reverse-solution.md#api-recommended-for-bulk-upload).

---

## Quick-Copy: Full List (bash array format)

```bash
CHINA_DOMAINS=(
  # ── Search & Content ──────────────────────────────────────
  "baidu.com"
  "bdstatic.com"
  "bcebos.com"
  "bing.com"
  "sogou.com"
  "so.com"
  "360.cn"
  "sm.cn"

  # ── Social & Messaging ────────────────────────────────────
  "qq.com"
  "weixin.qq.com"
  "wechat.com"
  "weibo.com"
  "weibo.cn"
  "sinaimg.cn"
  "sina.com.cn"
  "douyin.com"
  "tiktok.com"
  "ixigua.com"
  "kuaishou.com"
  "kwai.com"
  "xiaohongshu.com"
  "xhscdn.com"
  "zhihu.com"
  "zhimg.com"
  "tieba.baidu.com"
  "bilibili.com"
  "bilivideo.com"
  "hdslb.com"

  # ── E-Commerce ────────────────────────────────────────────
  "taobao.com"
  "tmall.com"
  "alipay.com"
  "alicdn.com"
  "tbcdn.cn"
  "jd.com"
  "jd.hk"
  "360buyimg.com"
  "pinduoduo.com"
  "yangkeduo.com"
  "xianyu.taobao.com"
  "meituan.com"
  "meituan.net"
  "eleme.cn"
  "ele.me"
  "kaola.com"
  "vip.com"
  "suning.com"
  "dangdang.com"

  # ── Cloud & Infrastructure (China) ────────────────────────
  "aliyun.com"
  "alibabacloud.com"
  "aliyuncs.com"
  "aliyundrive.com"
  "yuque.com"
  "amazonaws.cn"
  "aws.amazon.com.cn"
  "tencentcloud.com"
  "myqcloud.com"
  "qcloud.com"
  "cloud.tencent.com"
  "huaweicloud.com"
  "obs.cn-north-4.myhuaweicloud.com"
  "baidu.com"
  "baidubce.com"
  "baiduyun.com"
  "baidupan.com"
  "pan.baidu.com"
  "jdcloud.com"
  "ctyun.cn"
  "ucloud.cn"
  "qiniu.com"
  "qiniucdn.com"
  "qiniudn.com"
  "upyun.com"
  "upcdn.net"
  "ksyun.com"
  "ksyuncs.com"
  "21vianet.com"
  "scloud.cn"

  # ── Finance & Banking ──────────────────────────────────────
  "icbc.com.cn"
  "ccb.com"
  "abchina.com"
  "boc.cn"
  "bankofchina.com"
  "cmbchina.com"
  "spdb.com.cn"
  "cib.com.cn"
  "hxb.com.cn"
  "pingan.com"
  "pingan.com.cn"
  "paipai.com"
  "unionpay.com"
  "95516.com"
  "cnaps.com.cn"
  "cebbank.com"
  "psbc.com"
  "chinaums.com"
  "99bill.com"
  "lakala.com"

  # ── Travel & Maps ─────────────────────────────────────────
  "ctrip.com"
  "trip.com"
  "qunar.com"
  "elong.com"
  "ly.com"
  "amap.com"
  "gaode.com"
  "map.baidu.com"
  "maps.baidu.com"
  "ditu.baidu.com"
  "12306.cn"
  "timaticweb2.com"
  "caac.gov.cn"

  # ── Video & Entertainment ─────────────────────────────────
  "youku.com"
  "iqiyi.com"
  "qiyi.com"
  "71.am"
  "mgtv.com"
  "le.com"
  "pptv.com"
  "acfun.cn"
  "huya.com"
  "douyu.com"
  "zhanqi.tv"
  "yy.com"
  "lizhi.fm"
  "ximalaya.com"
  "kugou.com"
  "kuwo.cn"
  "music.163.com"
  "netease.com"
  "yinyuetai.com"
  "migu.cn"

  # ── News & Media ──────────────────────────────────────────
  "xinhuanet.com"
  "people.com.cn"
  "cctv.com"
  "cctv.cn"
  "cntv.cn"
  "chinadaily.com.cn"
  "china.com.cn"
  "ifeng.com"
  "163.com"
  "126.com"
  "yeah.net"
  "sohu.com"
  "sina.com"
  "eastmoney.com"
  "hexun.com"
  "caixin.com"
  "yicai.com"

  # ── Telecom ───────────────────────────────────────────────
  "10086.cn"
  "chinamobile.com"
  "cmpassport.com"
  "10010.com"
  "chinaunicom.com"
  "wo.com.cn"
  "189.cn"
  "chinatelecom.com.cn"
  "189.com.cn"
  "21cn.com"

  # ── Government & Tax ──────────────────────────────────────
  "gov.cn"
  "chinatax.gov.cn"
  "mofcom.gov.cn"
  "safe.gov.cn"
  "samr.gov.cn"
  "gsxt.gov.cn"
  "beian.gov.cn"
  "miit.gov.cn"
  "mps.gov.cn"
  "customs.gov.cn"
  "nia.gov.cn"
  "pbc.gov.cn"
  "csrc.gov.cn"
  "cbirc.gov.cn"
  "cnki.net"
  "12366.gov.cn"
  "chinainfo.gov.cn"

  # ── Education ─────────────────────────────────────────────
  "edu.cn"
  "tsinghua.edu.cn"
  "pku.edu.cn"
  "fudan.edu.cn"
  "sjtu.edu.cn"
  "zju.edu.cn"
  "gaokao.cn"
  "koolearn.com"
  "xueersi.com"
  "zuoyebang.com"
  "yuanfudao.com"
  "huohua.cn"

  # ── Tech & Developer ──────────────────────────────────────
  "csdn.net"
  "oschina.net"
  "gitee.com"
  "coding.net"
  "segmentfault.com"
  "juejin.cn"
  "cnblogs.com"
  "51cto.com"
  "iteye.com"
  "infoq.cn"
  "jikexueyuan.com"

  # ── Logistics & Express ───────────────────────────────────
  "sf-express.com"
  "shunfeng.com"
  "zto.com"
  "yto.net.cn"
  "sto.cn"
  "ems.com.cn"
  "yunda.com"
  "deppon.com"
  "jd.com"

  # ── Lifestyle & Local Services ────────────────────────────
  "dianping.com"
  "dazhongdianping.com"
  "58.com"
  "lianjia.com"
  "ke.com"
  "anjuke.com"
  "fang.com"
  "beike.com"
  "autohome.com.cn"
  "bitauto.com"
  "che168.com"
  "pcauto.com.cn"
  "xcar.com.cn"
  "pcpop.com"
  "smzdm.com"
  "zol.com.cn"
  "pconline.com.cn"
  "it168.com"
  "yesky.com"
  "myapp.com"
)
```

---

## By Category

### 🛒 E-Commerce & Payments

| Domain | Service |
|--------|---------|
| `taobao.com` | Taobao marketplace |
| `tmall.com` | Tmall B2C |
| `alipay.com` | Alipay payment |
| `alicdn.com` | Alibaba CDN |
| `jd.com` | JD.com |
| `360buyimg.com` | JD image CDN |
| `pinduoduo.com` | Pinduoduo |
| `meituan.com` | Meituan food/services |
| `eleme.cn` | Ele.me delivery |
| `vip.com` | VIP flash sales |

### ☁️ Cloud Services (China Regions)

| Domain | Provider |
|--------|---------|
| `aliyuncs.com` | Alibaba Cloud OSS/CDN |
| `amazonaws.cn` | AWS China |
| `myqcloud.com` | Tencent Cloud |
| `huaweicloud.com` | Huawei Cloud |
| `baidubce.com` | Baidu Cloud |
| `jdcloud.com` | JD Cloud |
| `ctyun.cn` | China Telecom Cloud |
| `qiniu.com` | Qiniu Cloud Storage |
| `upyun.com` | UpYun CDN |

### 🏛️ Government & Compliance

| Domain | Service |
|--------|---------|
| `chinatax.gov.cn` | State Tax Administration |
| `gsxt.gov.cn` | National Enterprise Credit |
| `beian.gov.cn` | ICP/Beian filing |
| `safe.gov.cn` | SAFE (forex) |
| `samr.gov.cn` | Market supervision |
| `pbc.gov.cn` | People's Bank of China |
| `csrc.gov.cn` | Securities regulator |
| `customs.gov.cn` | General Administration of Customs |

### 💬 Social & Communication

| Domain | Service |
|--------|---------|
| `qq.com` | QQ / WeChat infrastructure |
| `weibo.com` | Weibo |
| `douyin.com` | Douyin (TikTok China) |
| `xiaohongshu.com` | RED (小红书) |
| `zhihu.com` | Zhihu Q&A |
| `bilibili.com` | Bilibili video |

---

## Notes

- **Root domain coverage:** Adding `qq.com` covers `im.qq.com`, `v.qq.com`, `music.qq.com`, etc. You do not need to add subdomains separately.
- **CDN domains:** Include CDN domains (e.g., `alicdn.com`, `qiniu.com`) alongside the main service domain for complete coverage — many sites load static assets from separate CDN roots.
- **`gov.cn` wildcard:** Adding `gov.cn` covers all `*.gov.cn` subdomains. Evaluate whether broad government domain coverage is appropriate for your use case.
- **Dynamic IP changes:** Warp Reverse uses Hostname Routes (SNI-based), not IP routes. This means it handles dynamic IP changes at the origin automatically — no maintenance required.
- **Testing:** After adding routes, verify with `dig <domain>` while WARP is connected. The resolved IP should fall in the `100.64.0.0/10` range, confirming the Magic IP routing is active.

---

*Last updated: 2026-04 | Contributions welcome via PR*
