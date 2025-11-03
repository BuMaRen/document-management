---
title: "模块异步通信"
date: 2025-11-03T22:50:03+08:00
description : "Description goes here..."
tags: ["异步", "多进程", "系统设计"]
image : "/img/posts/2025/20251103/middle-service.jpg"
draft: True
---

# 模块间异步通信

一个Server中的模块A要向模块B推送消息。模块A先将消息推送给服务M，M将消息存到DB中，然后给模块A回复确认，在回复之前模块A会定期重试。M随后将消息发送给B，B收到消息后记下该消息唯一的id，然后给M回复确认

