# 工作笔记总索引

庾韬的折腾记录。每个主题一个仓库，这里只放索引和结论。

## 仓库索引

| 主题 | 仓库 | 说明 |
|---|---|---|
| 抖音模块（FreedomPlus 定制） | [freedomplus-custom](https://github.com/yehuzhucha/freedomplus-custom) | 36.8.0 适配、续火花修复、经验教训 |
| 小红书模块 | [xhs-module-notes](https://github.com/yehuzhucha/xhs-module-notes) | 免 root 补丁版登录结论、QuietBox 结构 |
| 论文降 AIGC | [paperpass-aigc-playbook](https://github.com/yehuzhucha/paperpass-aigc-playbook) | 提示词 + 13 版实测数据 |

## 核心结论（2026-10-30）

### 抖音续火花

**症状**：日志显示 9/9 全部成功，实际没人收到。

**根因**：只给消息构造器塞了会话 ID 字符串，抖音 IM 收下请求但不投递，
消息永远卡在 `status=1（发送中）`。

**正确链路**（36.8.0 实测）：

```
ConversationListModel.Companion.inst.getConversation(id)   → 会话对象
TextContent().setText() → GsonUtil.toJson()                → 内容
X.12pl.LIZ()                                               → IMessageUtils
utils.sendMessage(msg, Map, IRequestListener)              → 15 秒回调
```

**关键发现**：开源项目 SparkFlame / doumcp 用的是 `MultiInstanceHelperKt`，
但 36.8.0 把工厂挪到了 `X.12pl`。**照抄别的版本必踩坑，换版本必须重新反查 dex。**

**速度**：修好后间隔 10~20 秒连发 5 个全部成功。之前「发太快被限流」的判断是错的。

详细过程：[freedomplus-custom/docs/续火花排查记录-2026-10-30.md](https://github.com/yehuzhucha/freedomplus-custom/blob/main/docs/续火花排查记录-2026-10-30.md)

### 小红书

**补丁版登录不了，这是硬结论。**

- 9.50 / 9.51 / 8.57 全部失败，官方原版正常
- 签名校验在 `libsecurebase.so`（native 层），Java 层 0 处签名校验
- NPatch L2~L5 + hidelibs 全试过，绕不过
- 社区所有能用的模块都是给 LSPosed（root）的

详见 [xhs-module-notes](https://github.com/yehuzhucha/xhs-module-notes)。

### 论文降 AIGC

**提示词只改语体，不改统计指纹，压到 44.7% 就到底了。**

- 71% → 44.7%（提示词能做到的极限）
- 再往下只能靠：注入具体数字/场景、换段落骨架、句长突变
- 13 版实测数据 + 可用提示词在 [paperpass-aigc-playbook](https://github.com/yehuzhucha/paperpass-aigc-playbook)

## 通用方法论

1. **先隔离变量**：无模块包 / 有模块包、官方版 / 补丁版
2. **先证明服务端**：日志、回调、返回码，再怀疑网络和设备
3. **日志必须上报服务器**：只写本地文件，用户不发给你等于没有
4. **成功判定要来自宿主回调**，不是「没抛异常」
5. **开源项目给思路，不给常量**：类名、方法名、入口都可能随版本变
6. **一次抓全**：诊断版把所有候选路径的结果都上报，别一版试一个
7. **长任务防被杀**：缩短单轮时长，保持前台

## 服务器备忘

```
甲骨文  4 OCPU / 24 GB（Always Free 上限，不扣费）
        www.ccios.de → 64.181.247.149
        91 视频站 :9191（systemd video-site-91）
DMIT    154.17.236.55:48222（ssh -i ~/.ssh/dmit_key）
        x-ui 节点 9 条入站；IPv6 出站通、入站不通
续火花配置  ~/spark-config（:47200）→ https://www.ccios.de/spark/?token=***
```

## 时间线

### 2026-10-29
- 抖音 36.8.0 长按菜单 / 双击不点赞 / 视频信息 / 无水印下载适配
- 续火花首次跑通（10/10）
- 论文降 AIGC 第二次实战：71% → 44.7%
- 降 AIGC 改写台上线（www.ccios.de/paper/）

### 2026-10-30
- 续火花「假成功」排查 → 定位发送链路 → v19 修好
- 小红书补丁版登录结论：不可行
- 抖音模块经验教训整理
