# TVM_secret
新增tvm_secret資料夾，內有version1 ~ version4  
version1為加密測試階段  
version2為加密各frontend之.py檔案  
version3為測試5個tflite之top1、top5  
version4為TVMC

----------------------
###version2:  
1. `rsa_keygen.py`產生rsa key: `rsa_private_key.bin`、`rsa_public_key.pem`。  
2. `from_pytorch.py`、`from_tensorflow.py`、`from_tflite.py`、`from_onnx.py`、`from_mxnet.py`  
  為原tutorials/frontend底下的.py檔加上encrypt、decrypt功能、並有計時功能。  
3. `pytorch_time.txt`、`tensorflow_time.txt`、`tflite_time.txt`、`onnx_time.txt`、`mxnet_time.txt`  
  為計時結果，記錄每次運行*relay.frontend.from_pytorch()*、*relay.build()*、*graph_executor.Execution()* 之時間，  
  可以用`time_counter.py`計算和查看個別的平均運行時間。
  
  
