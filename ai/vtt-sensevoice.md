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

## Step2 开启一个服务,接受URL 音频并输出文本

```
import http.server
import socketserver
from http import HTTPStatus
from urllib.parse import urlparse, parse_qs
import re

from funasr import AutoModel
from urllib3.filepost import b
model_dir = "iic/SenseVoiceSmall"
input_file = (
    "https://fun-audio-llm.github.io/audios/emotion/%E4%B8%AD%E7%AB%8B1.wav"
)
model = AutoModel(model=model_dir,
                vad_model="fsmn-vad",
                vad_kwargs={"max_single_segment_time": 30000},
                trust_remote_code=True, device="cuda:0")

PORT = 8080
def modelscope(url):
    res = model.generate(   
        input=(url),
        cache={},
        language="zh", # "zn", "en", "yue", "ja", "ko", "nospeech"
        use_itn=False,
        batch_size_s=0,
    )
    print(res)
    try:
        data = res[0]
        # 移除<|zh|><|NEUTRAL|><|Speech|><|woitn|> 等
        pattern = re.compile(r'<\|.*?\|>')
        text = data['text']   
        text = re.sub(pattern,'', text)
        return text
    except Exception as e:
        print(e.__str__())
        return 'null'


class Handler(http.server.SimpleHTTPRequestHandler):
       def do_GET(self):
        # 解析请求行中的URL
        parsed_path = urlparse(self.path)
        # 解析查询字符串
        query_components = parse_qs(parsed_path.query)
        print(query_components)
        voice_url = query_components.get('url')
        # print(voice_url)
        if voice_url is not None:
            self.send_response(HTTPStatus.OK)
            self.send_header('Content-type', 'text/html; charset=utf-8')
            self.end_headers()
            self.wfile.write(modelscope(voice_url).encode())
        else: 
            self.send_response(HTTPStatus.NOT_FOUND)
            self.end_headers()
            self.wfile.write(b'hello')

if __name__ == '__main__':

    with socketserver.TCPServer(("", PORT), Handler, False) as httpd:
        print("Server started at port", PORT)
        httpd.allow_reuse_address = True
        httpd.server_bind()
        httpd.server_activate()
        httpd.serve_forever()
```
  

