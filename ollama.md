#### 列出所有模型
ollama list

#### 跑模型
ollama run qwen3:30b-a3b

#### 看正在运行的模型
ollama ps

#### 停掉某个加载到内存的模型
ollama stop qwen3:30b-a3b

#### 删除模型
ollama rm qwen3:30b-a3b

#### 通过 API 调用(默认 11434 端口)
curl http://localhost:11434/api/generate -d '{"model":"qwen3:30b-a3b","prompt":"你好"}'


