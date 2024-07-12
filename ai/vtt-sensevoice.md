# 语音识别模型SenseVoice在MarsCode IDE的测试



## MarsCode运行环境

- Python :  3.12.2
- CPU:  Intel(R) Xeon(R) Platinum 8375C CPU @ 2.90GHz

- 内存 4G
- 磁盘 10G
- 内核: 5.10.217-205.860.amzn2.x86_64

- PRETTY_NAME="Debian GNU/Linux 11 (bullseye)



## 测试推理

- `pip install funasr`

  如果Debug 时提示:  `No module named 'pkg_resources'` , 就执行 `pip install --upgrade pip setuptools wheel`

  ```
  from funasr import AutoModel
  model_dir = "iic/SenseVoiceSmall"
  input_file = (
      "https://fun-audio-llm.github.io/audios/emotion/%E4%B8%AD%E7%AB%8B1.wav"
  )
  def modelscope():
      model = AutoModel(model=model_dir,
                    vad_model="fsmn-vad",
                    vad_kwargs={"max_single_segment_time": 30000},
                    trust_remote_code=True, device="cuda:0")
      res = model.generate(   
          input=input_file,
          cache={},
          language="zh", # "zn", "en", "yue", "ja", "ko", "nospeech"
          use_itn=False,
          batch_size_s=0,
      )
      print(res)
      
  if __name__ == '__main__':
  		modelscope()
  ```

  

