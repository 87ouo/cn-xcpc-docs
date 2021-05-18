# 外榜

## Domjudge 的静态榜单

`http://domjudge/public?static=true`


### 滚榜注意：

- 使用 2.2.415 版，在 Bash 下运行 ICPC_FONT="Microsoft Yahei" ./resolver.bat awards.json；
- 或者 Microsoft JhengHei，是个中文字体就行 貌似新版对中文字体的支持变好了，不用加环境变量了；
- 为啥我在 2.1.2100 测的时候也不用加环境变量（系统 ICPC_FONT 的环境变量已经删了）……
- 结束之后把打星队设置为隐藏就行了，不用去删什么；
- 要把下载下来的 event-feed 加上 .json 后缀名（加上之后感觉怎么都能跑）！！！
- 下榜地址 http://domjudge/api/contests/{id}/event-feed?stream=false；
- 要等 TOO-LATE 测评完（感觉其实不太重要）；
- 总感觉滚榜的时候统计过题数是挂了的（旧版没有显示过题数的功能）……
