
## 1. 파이어베이스 연동 및 데이터 가져오기
```python
import firebase_admin
from firebase_admin import credentials
from firebase_admin import db, storage

cred=credentials.Certificate('firebase.json') 
firebase_admin.initialize_app(cred,{
    'databaseURL' : 'https://test01-fa68f.firebaseio.com/',  # 자기의 url 
    'storageBucket' : 'test01-fa68f.appspot.com'             # 자기의 url
})

ref = db.reference('image_ai/first')
print(ref.get())
# ref2 = db.reference('image_ai_state')
# ref2.update({'select':'aNb'}) 
```


## 2. 스토리지에서 파일 다운받기
```python
import firebase_admin
from firebase_admin import credentials
from firebase_admin import db, storage

remote_path_file = 'a.jpg'      # 스토리지 경로 및 파일이름
local_path_file = 'a.jpg'       # 내 디렉토리의 경로 및 파일이름

cred=credentials.Certificate('firebase.json')
firebase_admin.initialize_app(cred,{
    'databaseURL' : 'https://test01-fa68f.firebaseio.com/', 
    'storageBucket' : 'test01-fa68f.appspot.com'
})

def download_image(remote_path, local_path):
    bucket = storage.bucket()
    get_blob = bucket.blob(remote_path)
    get_blob.download_to_filename(local_path)
    print('다운로드 완료')
    return True


download_image(remote_path_file, local_path_file)
```

## 3. gpt를 사용해서 이미지 분석
```python
from openai import OpenAI
from dotenv import load_dotenv
import os
import base64

load_dotenv(verbose=True)
key = os.getenv('OPENAI_API_KEY')

img_file = 'a.jpg'
base64_image = None

client = OpenAI(api_key=key)

def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode('utf-8')
    
def request():
    response = client.chat.completions.create(
        model="gpt-4o", 
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": "당신은 이미지 분석 전문가입니다. 단어로 간력히 답변하세요. 이미지를 분석해서 물건의 이름을 bullet point로 작성해주세요. 쓸데 없는 말은 하지말아주세요. 한국어로 답변해주세요."},
                    {"type": "image_url", "image_url": {"url":  f"data:image/jpeg;base64,{base64_image}"}},
                ],
            }
        ],
        max_tokens=500
    )

    # print(response.choices[0])
    text = response.choices[0].message.content

    return text


base64_image = encode_image(img_file)

answer = request()
print(answer)
print(type(answer))
```


## 4. gpt를 사용해서 이미지 분석
```python
from openai import OpenAI
from dotenv import load_dotenv
import os
import base64
import json


load_dotenv(verbose=True)
key = os.getenv('OPENAI_API_KEY')

img_file = 'a.jpg'
base64_image = None

client = OpenAI(api_key=key)

def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode('utf-8')
    
def request():
    response = client.chat.completions.create(
        model="gpt-4o", 
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": "당신은 이미지 분석 전문가입니다. 단어로 간력히 답변하세요. 이미지를 분석해서 물건의 이름을 bullet point로 작성해주세요. 쓸데 없는 말은 하지말아주세요. 한국어로 답변해주세요."},
                    {"type": "image_url", "image_url": {"url":  f"data:image/jpeg;base64,{base64_image}"}},
                ],
            }
        ],
        max_tokens=500
    )

    # print(response.choices[0])
    text = response.choices[0].message.content

    return text


def get_json_data(question):
    
    completion = client.chat.completions.create(
        model="gpt-3.5-turbo-1106",
        messages=[
            {
                "role": "system",
                # 답변 형식을 JSON 으로 받기 위해 프롬프트에 JSON 형식을 지정
                "content": "You are a helpful assistant designed to output JSON. You must answer in Korean.",
            },
            {
                "role": "user",
                "content": question + "에 있는 물건의 이름을 key, 갯수를 value로 만들어 주세요.",
            },
        ],
        response_format={"type": "json_object"},  # 답변 형식을 JSON 으로 지정
        temperature=0.5,
        max_tokens=300,
    )

    for res in completion.choices:
        print(res.message.content)

    json_obj = json.loads(res.message.content)
    # print(json_obj)
    # print(type(json_obj))

    return json_obj
```

## 5. listen
```python
import firebase_admin
from firebase_admin import credentials
from firebase_admin import db, storage

cred=credentials.Certificate('firebase.json')
firebase_admin.initialize_app(cred,{
    'databaseURL' : 'https://test01-fa68f.firebaseio.com/', 
    'storageBucket' : 'test01-fa68f.appspot.com'
})

def listener(event):
    print(event)

ref = db.reference('image_ai_state/select')
ref.listen(listener)
```

## 6. 완료
```python
import firebase_admin
from firebase_admin import credentials
from firebase_admin import db, storage

from openai import OpenAI
from dotenv import load_dotenv

import time
import os
import base64
import json

load_dotenv(verbose=True)
key = os.getenv('OPENAI_API_KEY')


client = OpenAI(api_key=key)


img_file = 'a.jpg'              # 분석할 파일이름
global base64_image

remote_path_file = 'a.jpg'      # storage에 있는 파일이름
local_path_file = 'a.jpg'       # local에 다운로드 될 파일이름

cred=credentials.Certificate('firebase.json')
firebase_admin.initialize_app(cred,{
    'databaseURL' : 'https://test01-fa68f.firebaseio.com/', 
    'storageBucket' : 'test01-fa68f.appspot.com'
})

def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode('utf-8')
    
 
def request():
    global base64_image

    response = client.chat.completions.create(
        model="gpt-4o", 
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": "당신은 이미지 분석 전문가입니다. 단어로 간력히 답변하세요. 이미지를 분석해서 물건의 이름을 bullet point로 작성해주세요. 쓸데 없는 말은 하지말아주세요. 한국어로 답변해주세요."},
                    {"type": "image_url", "image_url": {"url":  f"data:image/jpeg;base64,{base64_image}"}},
                ],
            }
        ],
        max_tokens=500
    )

    # print(response.choices[0])
    text = response.choices[0].message.content

    return text

def get_json_data(question):
    
    completion = client.chat.completions.create(
        model="gpt-3.5-turbo-1106",
        messages=[
            {
                "role": "system",
                # 답변 형식을 JSON 으로 받기 위해 프롬프트에 JSON 형식을 지정
                "content": "You are a helpful assistant designed to output JSON. You must answer in Korean.",
            },
            {
                "role": "user",
                "content": question + "에 있는 물건의 이름을 key, 갯수를 value로 만들어 주세요.",
            },
        ],
        response_format={"type": "json_object"},  # 답변 형식을 JSON 으로 지정
        temperature=0.5,
        max_tokens=300,
    )

    for res in completion.choices:
        print(res.message.content)

    json_obj = json.loads(res.message.content)

    return json_obj

# storage에서 이미지 파일을 다운로드
def download_image(remote_path, local_path):
    bucket = storage.bucket()
    get_blob = bucket.blob(remote_path)
    get_blob.download_to_filename(local_path)
    print('다운로드 완료')
    return True


def data_changed_flag(event):
    print('event:', event)
    print('flag 변경')

    flag = db.reference('image_ai_state/flag')
    flag_data = flag.get()
    # print(flag_data)

    start = flag_data.index('a')+1
    end = flag_data.index('b')

    flag_state= flag_data[start : end]
    print('flag: ', flag_state)

    if flag_state == 'Y':
        ans = download_image(remote_path_file, local_path_file)
        print(ans)
        
        time.sleep(1)
        flag_ref = db.reference('image_ai_state')
        flag_ref.update({'flag':'aNb'})


def data_changed_select(event):
    global base64_image

    print('event:', event)
    print('select 변경') 

    select = db.reference('image_ai_state/select')
    select_data = select.get()
    # print(select_data)

    start = select_data.index('a')+1
    end = select_data.index('b')

    select_state= select_data[start : end]
    print('select: ', select_state)

    if select_state == 'Y':
        start2 = select_data.index('c')+1
        end2 = select_data.index('d')

        select_case_num = select_data[start2 : end2]
        print('case num: ', select_case_num)

        select_case_name = None

        if int(select_case_num)==1:
            select_case_name = 'first'
        elif int(select_case_num)==2:
            select_case_name = 'second'
        elif int(select_case_num)==3:
            select_case_name = 'third'

        select_case_ref = db.reference('image_ai/' + select_case_name)
        print(select_case_name, select_case_ref.get())

        n = len(select_case_ref.get())
        print('개수:', n)


        base64_image = encode_image(img_file)

        answer = request()
        print(answer)
        print(type(answer))

        json_conetnt = get_json_data(answer)
        print('json_conetnt', json_conetnt)

        for i, j in json_conetnt.items():
            n=n+1
            select_case_ref.update({'key'+str(n):i})

        time.sleep(1)
        flag_ref = db.reference('image_ai_state')
        flag_ref.update({'select':'aNb'})
     


ref1 = db.reference('image_ai_state/flag')
ref1.listen(data_changed_flag)

ref2 = db.reference('image_ai_state/select')
ref2.listen(data_changed_select)
```