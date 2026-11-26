# CTF Writeups

个人 CTF 刷题记录与 writeup 归档。

## 目录结构

```
ctf-writeups/
├─ README.md          本文件
├─ .gitignore
└─ writeups/          每篇 writeup 一个文件，命名：<平台>-<题目>-<漏洞类型>.md
```

## 写作约定

每篇 writeup 至少包含四段：

1. **题目信息**：平台、题目名、分值、难度
2. **环境 / 入口**：给了什么、从哪开始
3. **利用过程**：复现步骤，关键 payload 和响应
4. **原理与修复**：漏洞成因、真实场景怎么防

## 进度

- [ ] 环境搭建（Kali / WSL、Burp Suite）
- [ ] SQL 注入（sqli-labs 1-20）
- [ ] XSS（三型 + DVWA）
- [ ] 文件上传（upload-labs）
- [ ] 反序列化（PHP POP 链）
- [ ] 综合靶场（BUUCTF）

## 速查图表

| 主题 | 图片 | 文字版 |
|---|---|---|
| SQL 注入（MySQL） | [SQL注入速查.png](cheatsheets/SQL注入速查.png) | [SQL注入速查.md](cheatsheets/SQL注入速查.md) |