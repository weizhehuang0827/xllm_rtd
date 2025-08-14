# 使用

## 快速使用
直接启动单节点的`xllm`服务：
```bash
./build/xllm/core/server/xllm \    # 启动 xllm 服务器程序
    --model=/path/to/your/qwen2-7b  \   # 指定模型路径（需替换为实际路径）
    --backend=llm \                # 指定后端类型为 LLM
    --port=9977 \                  # 设置服务端口为 9977
    --max_memory_utilization 0.90  # 设置最大内存利用率为 90
```
### Curl 调用
chat模式：
```bash
curl http://localhost:9977/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2-7B-Instruct",
    "max_tokens": 10,
    "temperature": 0,
    "stream": true,
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful assistant."
      },
      {
        "role": "user",
        "content": "hello xllm"
      }
    ]
  }'
```
completions模式：
```bash
curl http://127.0.0.1:9977/v1/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "Qwen2-7B-Instruct",
    "prompt": "hello xllm",
    "max_tokens": 10,
    "temperature": 0,
    "stream": true
  }'
```


### Python调用
```python
import requests
import json

url = f"http://localhost:9977/v1/chat/completions"
messages = [
    {'role': 'user', 'content': "列出三个国家和他的首都。"}
]

request_data = {
    "model": "Qwen2-7B-Instruct",
    "messages": messages,
    "stream": False, 
    "temperature": 0.6, 
    "max_tokens": 2048, 
}

response = requests.post(url, json=request_data)
if response.status_code != 200:
    print(response.status_code, response.text)
else:
    ans = json.loads(response.text)["choices"]
    print(ans[0]['message'])
```


## 多节点服务
启动服务:
```shell
bash start_qwen.sh
```
start_qwen.sh 脚本如下:
```bash
# 1. 环境变量设置
export PYTHON_INCLUDE_PATH="$(python3 -c 'from sysconfig import get_paths; print(get_paths()["include"])')"
export PYTHON_LIB_PATH="$(python3 -c 'from sysconfig import get_paths; print(get_paths()["include"])')"
export PYTORCH_NPU_INSTALL_PATH=/usr/local/libtorch_npu/  # NPU 版 PyTorch 路径
export PYTORCH_INSTALL_PATH="$(python3 -c 'import torch, os; print(os.path.dirname(os.path.abspath(torch.__file__)))')"  # PyTorch 安装路径
export LIBTORCH_ROOT="$PYTORCH_INSTALL_PATH"  # LibTorch 路径
export LD_LIBRARY_PATH=/usr/local/libtorch_npu/lib:$LD_LIBRARY_PATH  # 添加 NPU 库路径

# 2. 加载 Ascend 环境
source /usr/local/Ascend/ascend-toolkit/set_env.sh  # 加载 CANN 工具链
source /usr/local/Ascend/nnal/atb/set_env.sh       # 加载 ATB 加速库
export ASCEND_RT_VISIBLE_DEVICES=10,11             # 指定可见 NPU 设备（物理卡 10 和 11）
export ASDOPS_LOG_TO_STDOUT=1   # 将 ASDOPS 的日志输出到标准输出（终端）
export ASDOPS_LOG_LEVEL=ERROR   # 设置 ASDOPS 的日志级别，仅输出指定级别及以上的日志
export ASDOPS_LOG_TO_FILE=1     # 将 ASDOPS 的日志写入默认路径
# export HCCL_BUFFSIZE=1024
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True  # 允许显存动态扩展
export NPU_MEMORY_FRACTION=0.98                    # 显存利用率上限 98%
export ATB_WORKSPACE_MEM_ALLOC_ALG_TYPE=3          # ATB 内存分配算法
export ATB_WORKSPACE_MEM_ALLOC_GLOBAL=1            # 全局内存分配
export OMP_NUM_THREADS=12   # OpenMP 线程数（建议与 CPU 核数匹配）
export HCCL_CONNECT_TIMEOUT=7200    # HCCL 连接超时（2 小时）
export INF_NAN_MODE_ENABLE=0

# 3. 清理旧日志
\rm -rf /root/atb/log/
\rm -rf /root/ascend/log/
\rm -rf core.*

# 4. 启动分布式服务
MODEL_PATH="/path/to/your/Qwen2-7B-Instruct"  # 模型路径
MASTER_NODE_ADDR="127.0.0.1:9748"                  # Master 节点地址（需全局一致）
START_PORT=18000                                   # 服务起始端口
START_DEVICE=0                                     # 起始 NPU 逻辑设备号
LOG_DIR="log"                                      # 日志目录
NNODES=2                                           # 节点数（当前脚本启动 2 个进程）

export HCCL_IF_BASE_PORT=43432  # HCCL 通信基础端口
export FOLLY_DEBUG_MEMORYIDLER_DISABLE_UNMAP=1  # 禁用内存释放（提升稳定性）

for (( i=0; i<$NNODES; i++ ))
do
  PORT=$((START_PORT + i))
  DEVICE=$((START_DEVICE + i))
  LOG_FILE="$LOG_DIR/node_$i.log"
  ./xllm/build/xllm/core/server/xllm \
    --model $MODEL_PATH \
    --devices="npu:$DEVICE" \
    --port $PORT \
    --master_node_addr=$MASTER_NODE_ADDR \
    --nnodes=$NNODES \
    --max_memory_utilization=0.86 \
    --max_tokens_per_batch=40000 \
    --max_seqs_per_batch=256 \
    --enable_mla=false \
    --block_size=128 \
    --communication_backend="hccl" \
    --enable_prefix_cache=false \
    --enable_chunked_prefill=true \
    --enable_schedule_overlap=true \
    --node_rank=$i  &
done
```
这里使用了两个节点，可以通过 `--nnodes=$NNODES` 和`--node_rank=$i`来设置。
同时可以通过 `ASCEND_RT_VISIBLE_DEVICES` 环境变量设置NPU Device。

客户端测试命令和上面一样。

## PD分离服务

`xllm`支持PD分离部署，这需要与我们的另一个开源库[xllm service](https://github.com/jd-opensource/xllm-service)配套使用。
### 依赖安装编译
首先，我们下载安装`xllm service`，与安装编译`xllm`类似：
```bash
git clone https://github.com/jd-opensource/xllm-service
cd xllm_service
git submodule init
git submodule update
```
`xllm_service`编译运行依赖[vcpkg](https://github.com/microsoft/vcpkg)和[etcd](https://github.com/etcd-io/etcd)。先确保在前面[编译xllm](./compile-and-run.md)时已经进行了`vcpkg`的安装且设置了`vcpkg`的路径：
```bash
export VCPKG_ROOT=/your/path/to/vcpkg
```
使用etcd官方提供的[安装脚本](https://github.com/etcd-io/etcd/releases)进行安装，其脚本提供的默认安装路径是`/tmp/etcd-download-test/etcd`，我们可以手动修改其脚本中的安装路径，也可以运行完脚本之后手动迁移：
```bash
mv /tmp/etcd-download-test/etcd /path/to/your/etcd
```
再执行编译:
```bash
mkdir -p build
cd build
cmake ..
make -j 8
cd ..
```
> 这里能会遇到关于`boost-locale`和`boost-interprocess`的安装错误：`vcpkg-src/packages/boost-locale_x64-linux/include: No such file or directory`,`/vcpkg-src/packages/boost-interprocess_x64-linux/include: No such file or directory`
我们使用`vcpkg`重新安装这些包:
```bash
/path/to/vcpkg remove boost-locale boost-interprocess
/path/to/vcpkg install boost-locale:x64-linux
/path/to/vcpkg install boost-interprocess:x64-linux
```
### PD分离运行
启动etcd:
```bash
./etcd-download-test/etcd --listen-peer-urls 'http://localhost:2390'  --listen-client-urls 'http://localhost:2389' --advertise-client-urls  'http://localhost:2391'
```
启动xllm service:
```bash
ENABLE_DECODE_RESPONSE_TO_SERVICE=0 \    # 环境变量：禁用解码响应到服务
ENABLE_XLLM_DEBUG_LOG=1 \                # 环境变量：启用 XLLM 调试日志
./build/xllm_service/xllm_master_serving \  # 启动 xllm_master_serving 服务
    --etcd_addr="127.0.0.1:2389" \       # 指定 etcd 服务地址（默认本地 2389 端口）
    --http_server_port=9888 \            # 设置 HTTP 服务端口为 9888
    --rpc_server_port=9889              # 设置 RPC 服务端口为 9889
```
启动Prefill节点：
```bash
bash start_pd.sh
```
启动Decode节点：
```bash
bash start_pd.sh decode
```
start_pd.sh脚本如下:
```bash
export PYTHON_INCLUDE_PATH="$(python3 -c 'from sysconfig import get_paths; print(get_paths()["include"])')"
export PYTHON_LIB_PATH="$(python3 -c 'from sysconfig import get_paths; print(get_paths()["include"])')"
export PYTORCH_NPU_INSTALL_PATH=/usr/local/libtorch_npu/  # NPU 版 PyTorch 路径
export PYTORCH_INSTALL_PATH="$(python3 -c 'import torch, os; print(os.path.dirname(os.path.abspath(torch.__file__)))')"  # PyTorch 安装路径
export LIBTORCH_ROOT="$(python3 -c 'import torch, os; print(os.path.dirname(os.path.abspath(torch.__file__)))')"  # LibTorch 路径
source /usr/local/Ascend/ascend-toolkit/set_env.sh  # Ascend Toolkit 环境变量
source /usr/local/Ascend/nnal/atb/set_env.sh  # ATB（Ascend Tensor Boost）环境变量
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/libtorch_npu/lib  # 添加 NPU LibTorch 库路径

# 清理日志和临时文件
\rm -rf /root/atb/log/
\rm -rf /root/ascend/log/
\rm -rf core.*
\rm -rf ~/dynamic_profiling_socket_*

echo "$1"
if [ "$1" = "decode" ]; then
  # decode模式
  echo ">>>>> decode"
  export ASCEND_RT_VISIBLE_DEVICES=6,7  # 使用 NPU 设备 6 和 7

  ./xllm/build/xllm/core/server/xllm \
  --model /export/home/xiongjun3/models/Qwen2-7B-Instruct \
  --max_memory_utilization 0.90 \
  --devices="npu:1" \
  --instance_role DECODE \
  --enable_disagg_pd=true \
  --enable_cuda_graph=false \
  --enable_prefix_cache=false \
  --backend=llm \
  --port=9996  \
  --xservice_addr=127.0.0.1:9889  \
  --host=127.0.0.1 \
  --disagg_pd_port=7780 \
  --cluster_id=1 \
  --device_ip=11.86.23.217 \
  --transfer_listen_port=26001 \
  --enable_service_routing=true
else
  # prefill模式
  echo ">>>>> prefill"
  export ASCEND_RT_VISIBLE_DEVICES=6,7  # 使用 NPU 设备 6 和 7 
  
  ./xllm/build/xllm/core/server/xllm \
  --model /export/home/xiongjun3/models/Qwen2-7B-Instruct \
  --max_tokens_per_batch 102400  \
  --max_memory_utilization 0.90  \
  --devices="npu:0"  \
  --instance_role PREFILL \
  --enable_disagg_pd=true \
  --enable_cuda_graph=false \
  --enable_prefix_cache=false \
  --backend=llm \
  --port=9997  \
  --xservice_addr=127.0.0.1:9889  \
  --host=127.0.0.1 \
  --cluster_id=0 \
  --device_ip=11.86.23.216 \
  --transfer_listen_port=26000 \
  --disagg_pd_port=7781 \
  --enable_service_routing=true
fi
```
需要注意：
- PD分离在指定NPU Device的时候，需要对应的`device_ip`，这个每张卡是不一样的，具体的可以在非容器环境下的物理机器上执行下面命令看到,其呈现的`address_{i}=`后面的值就是对应`NPU {i}`的`device_ip`。
```bash
sudo cat /etc/hccn.conf
```
- `xservice_addr`需与`xllm_service`的`rpc_server_port`相同

测试命令和上面类似，注意`curl http://localhost:{PORT}/v1/chat/completions ...`的`PORT`选择为prefill节点的`port`或者`xllm service`的`http_server_port`。