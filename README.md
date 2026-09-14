# CTF Writeups

个人 CTF 题解记录 — 每一道题都是一次真实的学习。

## 题解索引

| 赛事 | 题目 | 类型 | 核心考点 | 链接 |
|---|---|---|---|---|
| 2026 HUSEC | ctf26 | Web / PHP | 运算符优先级 + 变量变量 + parse_str 键转换 | [题解](2026-HUSEC/php-var-cover.md) |

## 环境

- Windows 10 + Git Bash
- PHP 7.4.30（靶机）
- 工具：curl / Burp Suite / 浏览器 DevTools

## 题解规范

每篇题解包含四部分：

1. **题目源码** — 还原原始代码
2. **审计** — 逐点标注缺陷
3. **构造** — 利用链的思路推导
4. **Payload** — 可直接复现的完整请求
