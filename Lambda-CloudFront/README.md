## **📌 프로젝트 개요**


웹사이트에서 대용량 이미지를 불러올 경우 로딩 속도가 느려져 사용자 경험이 저하됩니다. 이를 해결하기 위해 **AWS Lambda를 이용한 자동 이미지 리사이징**과 **CloudFront를 사용하여 캐싱 최적화**를 적용하였습니다.

### **🔎 적용 이유**



✔ **Lambda 활용 (이미지 리사이징 자동화)**

- 원본 이미지 크기가 커서 로딩 속도가 저하됨
- 사용자가 불편을 느끼지 않도록 자동으로 최적화된 이미지 제공

✔ **CloudFront 활용 (빠른 이미지 제공)**

- 자주 사용되는 이미지를 캐싱하여 로딩 속도 향상
- S3 버킷 URL을 직접 노출하지 않음 → 보안성 강화

## **✨ 최종 결과**



🎯 **AWS Lambda + S3 + CloudFront를 활용하여 완전 자동화된 이미지 최적화 시스템을 구축하였습니다!**

✅ 이미지 로딩 속도 30% 개선 (200ms → 140ms)

✅ 원본 이미지를 업로드하면 자동으로 최적화하여 제공

✅ 캐싱을 활용하여 빠른 서비스 제공

## **📂 시스템 구성**



### **1️⃣ S3 버킷 구성**

💾 **원본 이미지 저장:** `dapanda-file-origin` (비공개)

📁 **리사이징된 이미지 저장:** `dapanda-file-resizing`

![Image](https://github.com/user-attachments/assets/dfae2073-0d00-4338-84d5-e045788276a0)

### **2️⃣ Lambda를 활용한 이미지 리사이징**

**📌 동작 방식**

1. 사용자가 `dapanda-file-origin` 버킷에 이미지를 업로드
2. S3 이벤트 트리거 → **Lambda 함수 실행**
3. Lambda가 이미지를 리사이징하여 `dapanda-file-resizing` 버킷에 저장
4. CloudFront를 통해 빠르게 제공

![Image](https://github.com/user-attachments/assets/c4d5a216-636d-48f4-a74d-c389de64ec30)

- 람다 함수 생성

![Image](https://github.com/user-attachments/assets/fd98e4d5-34ed-4499-b9cd-9364088e9756)

🔑 Lambda 권한(Role) 설정

```c
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::dapanda-file-origin/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::dapanda-file-resizing/*"
        }
    ]
}
```

🔑 Lambda 신뢰 관계 설정

```c
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

![Image](https://github.com/user-attachments/assets/3a1bdef4-b179-4798-b83d-82d883360fa1)

## **🖥️ Lambda 코드 구현**



Lambda가 이미지를 자동으로 리사이징하여 최적화된 파일을 S3에 저장하는 코드입니다.

```c
import boto3
import os
from PIL import Image

# S3 클라이언트 설정
s3_client = boto3.client('s3')

def resize_image(image_path, resized_path, width, height):
    with Image.open(image_path) as image:
        image = image.resize((width, height), Image.LANCZOS)
        image.save(resized_path, image.format, quality=95)
        
def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        download_path = f"/tmp/{os.path.basename(key)}"
        upload_path = f"/tmp/resized-{os.path.basename(key)}"
        
        os.makedirs(os.path.dirname(download_path), exist_ok=True)
        
        s3_client.download_file(bucket, key, download_path)
        resize_image(download_path, upload_path, 588, 626)
        
        s3_client.upload_file(upload_path, 'dapanda-file-resizing', key, ExtraArgs={'ContentType': 'image/jpeg'})
        
        if os.path.exists(download_path):
            os.remove(download_path)
        if os.path.exists(upload_path):
            os.remove(upload_path)

```

**✨ 주요 기능**

✅ 이미지 다운로드 & 리사이징 후 저장

✅ LANCZOS 필터 적용 → 고품질 리사이징

✅ S3에서 가져온 이미지를 /tmp 폴더에 임시 저장 후 삭제

Layer에 pillow 추가

![Image](https://github.com/user-attachments/assets/54242637-5b63-4f95-bd58-009cb1259c7c)

## **🛠️ CloudFront 설정**

**📌 CloudFront를 사용하여 S3에서 직접 이미지 제공하는 것이 아닌, 캐싱을 활용하여 빠르게 로드할 수 있도록 설정하였습니다.**

### **🎯 CloudFront 역할**

✔ **자주 요청되는 이미지 캐싱 → 성능 최적화**

✔ **S3 버킷 직접 접근 차단 → 보안 강화**

### **🔧 CloudFront 배포 설정**

- **Origin:** `dapanda-file-resizing` S3 버킷
- **Behavior 설정:**
    - `GET`, `HEAD` 요청 허용
    - 캐시 정책 최적화
    - CORS 설정 추가하여 웹사이트에서 정상적으로 로드 가능하도록 설정

![Image](https://github.com/user-attachments/assets/dabdaa08-6e24-4c0c-888f-2d9bb528dc4e)

![Untitled](attachment:12dacf16-2c95-47a5-bf4e-b9965694dd4b:Untitled.png)

![Untitled](attachment:dc6ed979-501a-4a24-b6df-216a35b44e2b:Untitled.png)
