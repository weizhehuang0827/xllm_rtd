# 编译执行

## 安装
进入到容器后，使用我们的[官方仓库](https://github.com/jd-opensource/xllm)下载编译：
```bash
git clone https://github.com/jd-opensource/xllm
cd xllm 
git submodule init
git submodule update
```
编译依赖[vcpkg](https://github.com/microsoft/vcpkg)，我们编译的时候会默认下载`vcpkg`，也可以先提前下载`vcpkg`，然后设置环境变量:
```bash
git clone https://github.com/microsoft/vcpkg.git
export VCPKG_ROOT=/your/path/to/vcpkg
```
然后下载安装python依赖:
```
cd xllm
pip install -r cibuild/requirements-dev.txt -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
pip install --upgrade setuptools wheel
```
## 编译
执行编译，在`build/`下生成可执行文件`build/xllm/core/server/xllm`：
```bash
python setup.py build
```
也可以直接用以下命令编译&&在`dist/`下生成whl包: 
```bash
python setup.py bdist_wheel
```
## 执行
之后可以运行例如如下命令启动`xllm`引擎：
```bash
./build/xllm/core/server/xllm \    # 启动 xllm 服务器程序
    --model=/path/to/your/llm  \   # 指定模型路径（需替换为实际路径）
    --backend=llm \                # 指定后端类型为 LLM
    --port=9977 \                  # 设置服务端口为 9977
    --max_memory_utilization 0.90  # 设置最大内存利用率为 90
```
更多的传入参数信息可以参考。。。