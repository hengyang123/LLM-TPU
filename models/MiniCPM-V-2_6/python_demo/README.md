# 环境准备
> （python demo运行之前需要执行这个）
```
sudo apt-get update
sudo apt-get install pybind11-dev
pip3 install sentencepiece transformers==4.40.0
pip3 install gradio==3.39.0 mdtex2html==1.2.0 dfss
```

如果不打算自己编译模型，可以直接用下载好的模型
```
python3 -m dfss --url=open@sophgo.com:/ext_model_information/LLM/LLM-TPU/minicpmv26_bm1684x_int4_seq1024_imsize448.bmodel
```

编译库文件
```
mkdir build
cd build && cmake .. && make && cp *cpython* .. && cd ..
```

# python demo
```
python3 pipeline.py --model_path minicpmv26_bm1684x_int4_seq1024_imsize448.bmodel --processor_path ../support/processor_config/ --devid 0
```
