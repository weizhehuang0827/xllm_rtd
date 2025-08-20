# MTP投机推理

## 背景
MTP是一种创新的推理阶段加速技术，专注于解决大语言模型生成过程中的效率瓶颈。MTP的本质是通过预训练阶段的特殊设计，为推理阶段提供高效的草稿token预测能力，从而显著提升模型的生成速度。其核心价值在于平衡推理效率与输出质量，为大语言模型的长序列生成问题提供了一种高效的解决方案，最终实现推理性能的优化。

## 功能介绍
MTP在推理加速方面具有以下核心功能：

`高效草稿生成`：使用低成本的MTP结构快速生成草稿token，这些草稿token作为主模型验证的基础，大幅减少了传统自回归生成的计算开销。

`批量验证机制`：主模型能够同时批量验证多个MTP生成的草稿token，而不必逐个生成和验证，显著提升了推理速度。

`高采样准确率`：MTP解决了Eagle、Medusa等现有推理加速方法中的关键痛点——训练后生成的draft模块token采样率低的问题。由于MTP在预训练阶段就优化了草稿生成能力，其生成的token具有更高的准确率，减少了主模型的验证负担。

`推理延迟降低`：通过预先生成多个可能的后续token，MTP有效降低了模型生成长文本时的累积延迟，使用户体验更加流畅。

`资源消耗优化`：相比其他推理加速技术，MTP在保持加速效果的同时，对计算资源的额外需求更少，适合在资源受限环境下部署。

MTP技术为大语言模型的推理阶段提供了一种全新的效率优化方案，特别适合需要快速响应的实时应用场景，代表了语言模型推理优化的重要发展方向。

## 使用示例

### 导出模型
```bash
./tools/export_deepseek_mtp.py --input-dir /path/to/DeepSeek-V3 --output-dir /path/to/DeepSeek-V3-mtp
```
输入模型参考: [Deepseek-V3](https://huggingface.co/deepseek-ai/DeepSeek-V3)

### 启动脚本
```bash
MODEL_PATH="/models/DeepSeek-V3"
DRAFT_MODEL_PATH="/models/DeepSeek-V3-MTP"
MASTER_NODE_ADDR="127.0.0.1:42123"
START_PORT=13222
START_DEVICE=0
LOG_DIR="log"
NNODES=16

for (( i=0; i<$NNODES; i++ ))
do
  PORT=$((START_PORT + i))
  DEVICE=$((START_DEVICE + i))
  LOG_FILE="$LOG_DIR/node_$i.log"
  nohup ./xllm \
    --model $MODEL_PATH \
    --devices="npu:$DEVICE" \
    --port $PORT \
    --master_node_addr=$MASTER_NODE_ADDR \
    --nnodes=$NNODES \
    --draft_model $DRAFT_MODEL_PATH \
    --draft_devices="npu:$DEVICE" \
    --num_speculative_tokens 1 \
    --max_memory_utilization=0.90 \
    --max_tokens_per_batch=10000 \
    --max_seqs_per_batch=256 \
    --enable_mla=true \
    --block_size=128 \
    --ep_size=1 \
    --dp_size=1 \
    --enable_prefix_cache=false \
    --enable_chunked_prefill=false \
    --node_rank=$i > $LOG_FILE 2>&1 &
  sleep 0.5
done
```

# 性能数据