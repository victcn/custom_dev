# openclaw Gateway是什么

Gateway 是openclaw的控制层，主要负责：
1. 入口层：接入大部分的IM软件，以及做认证、设备匹配、allowlist等
2. 识别上下文：判断消息来自哪个channel 、哪个用户的哪个session
3. 调度agent-runtime，收到消息触发agent run 
4. 推送过程：把sse推送过去
5. 输出结果：把消息推送到原始渠道


 OpenClaw = Gateway daemon + Agent Runtime +
  State/Workspace/Session/Memory + Tools/Plugins