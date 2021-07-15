# TVM_secret

----------------------
新增tvm_secret資料夾，內有version1 ~ version4  
version1為加密測試階段  
version2為加密各frontend之.py檔案  
version3為測試5個tflite之top1、top5  
version4為TVMC

----------------------
### version2內容：  
1. `rsa_keygen.py`產生rsa key: `rsa_private_key.bin`、`rsa_public_key.pem`。  
2. `from_pytorch.py`、`from_tensorflow.py`、`from_tflite.py`、`from_onnx.py`、`from_mxnet.py`  
  為原tutorials/frontend底下的.py檔加上encrypt、decrypt功能、並有計時功能。  
3. `pytorch_time.txt`、`tensorflow_time.txt`、`tflite_time.txt`、`onnx_time.txt`、`mxnet_time.txt`  
  為計時結果，記錄每次運行*relay.frontend.from_{TYPE}()*、*relay.build()*、*graph_executor.Execution()* 之時間，  
  可以用`time_counter.py`計算和查看個別的平均運行時間。
  
----------------------
### 加密流程：  
* `from_pytorch.py`為模擬將模型加密後再傳給第三方，也就是先用`encrypt.py`將模型加密後存成`encrypted_model.crypt1`，再將這個加密後的檔案
  給第三方執行。  
* `from_tensorflow.py`、`from_tflite.py`、`from_onnx.py`、`from_mxnet.py`、為了測試方便，所以直接將model加密，沒有另外寫`encrypt.py`。  
* 以`from_tensorflow.py`為例，運行*relay.frontend.from_tensorflow()* 時會進入`python/tvm/relay/frontend/tensorflow.py`裡的*from_tensorflow()* 函式，  
  在進入函式後就先解密model等必要資訊，然後運行原本的動作後再加密傳回去，所以使用者只會看到回傳後加密後的資訊。
* *relay.build()* 會進入`python/tvm/relay/build_module.py`裡的*build()* 函式，一樣先解密必要資訊，然後運行原本的動作，但是原本回傳的**executor_factory** 不好
  加密，所以改為加密前一步的**tophub_context** ，並回傳一個**enc_list** ，裡面有加密後的各個重要參數。  
* *graph_executor.Execution()* 是自行撰寫的函式，主要功能是將原本的runtime部分隱藏起來，也包括*build()* 的最後一部份(**tophub_context** )，  
  進入`python/tvm/contrib/graph_executor.py`裡的*Execution()* 函式，先將**enc_list** 裡的參數解密後再跑原先*build()* 裡的**tophub_context** 得到  
  executor_factory後再產生實例、運行runtime，最後將結果**tvm_output** 回傳。  
 
----------------------
### 主要加解密function：
```
######################################################################
import pickle
import os
from Crypto.PublicKey import RSA
from Crypto.Cipher import AES, PKCS1_OAEP
from Crypto.Random import get_random_bytes

def Encrypt(data): 
    recipient_key = RSA.import_key(open('rsa_public_key.pem').read())

    session_key = get_random_bytes(16)
    # Encrypt the session key with the public RSA key
    cipher_rsa = PKCS1_OAEP.new(recipient_key)
    ###out_file.write(cipher_rsa.encrypt(session_key))
    # Encrypt the data with the AES session key
    cipher_aes = AES.new(session_key, AES.MODE_EAX)

    ciphertext, tag = cipher_aes.encrypt_and_digest(data)

    encrypt_data = cipher_rsa.encrypt(session_key) + cipher_aes.nonce + tag + ciphertext

    return encrypt_data

def Decrypt_1(encrypted_data):
    code = 'TVM'
    private_key = RSA.import_key(open('rsa_private_key.bin').read(), passphrase=code)
    enc_session_key, nonce, tag, ciphertext = [ encrypted_data[0 : private_key.size_in_bytes()], 
                                                encrypted_data[private_key.size_in_bytes() : private_key.size_in_bytes()+16],
                                                encrypted_data[private_key.size_in_bytes()+16 : private_key.size_in_bytes()+32],
                                                encrypted_data[private_key.size_in_bytes()+32 :]
                                              ]

    cipher_rsa = PKCS1_OAEP.new(private_key)
    session_key = cipher_rsa.decrypt(enc_session_key)
    cipher_aes = AES.new(session_key, AES.MODE_EAX, nonce)
    data = cipher_aes.decrypt_and_verify(ciphertext, tag)

    return data
######################################################################
```
