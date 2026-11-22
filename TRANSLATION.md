# 中文本地化说明

本项目是 [ollama-gui](https://github.com/HelgeSverre/ollama-gui) 的简体中文分支（MIT 许可）。

## 翻译原则

界面分两类处理，避免出现"最大词元"这类生硬翻译。

### 保持英文

- **参数名**：`Temperature`、`Top P`、`Top K`、`Context length (num_ctx)`、`Max tokens (num_predict)`、`Repeat penalty`、`Seed`
- **功能开关名**：`Thinking`、`Keep alive`
- **选项值**：`Off` / `On`
- **模型元数据**：`Family`、`Parameters`、`Quantization`、`Capabilities`
- **产品与协议**：`Ollama`、`Ollama library`、`Ollama GUI`、`Markdown`、`Gravatar`

### 翻译成中文

导航、按钮、动作、状态提示、说明文字。例如：新建对话、归档、导出为 Markdown、已安装、已加载、仅发送给支持 Thinking 的模型。

## 与上游的差异

1. 全部界面文案中文化（约 180 处）
2. 对话标题生成提示词改为**跟随对话语言**，中文对话得到中文标题
3. 新增设置项 **标题生成模型**（`设置 → 对话`）：
   可指定一个更小更快的模型专门生成标题，避免用 35B 这类大模型聊天时，标题生成拖慢或阻塞主对话。留空则跟随当前对话模型。

## 同步上游

```bash
git remote add upstream https://github.com/HelgeSverre/ollama-gui.git
git fetch upstream
git merge upstream/main   # 文案冲突集中在 .vue 模板，按本文件的原则取舍
```
