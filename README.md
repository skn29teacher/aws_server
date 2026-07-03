# CI/CD 우선형 OpenAI `gpt-5-nano` AWS 챗봇 구축 가이드
*(GitHub Actions + Nginx + Gunicorn + Django + S3 + RDS + OpenAI)*

이 가이드는 AWS 자원이 아무것도 없는 상태에서 시작하여, **"AWS 클라우드 자원 생성 ➡️ EC2 기본 뼈대 구성 ➡️ GitHub Actions CI/CD 자동화 구축 ➡️ 로컬 중심 연동 개발(RDS, S3, OpenAI)"** 순서로 프로젝트를 완수해 나가는 완전 무결한 올인원(All-in-One) 가이드라인입니다.

이 가이드를 충실히 따라가면 서버에 직접 접속하지 않고 로컬 PC에서 안전하게 개발하여 실서비스에 자동 배포하는 고성능 AI 애플리케이션의 개발 사이클을 완벽하게 체득할 수 있습니다.

---

## 1. [초기 환경 구축] AWS 클라우드 자원 생성 가이드

가상 인프라 구축을 위해 AWS 웹 콘솔에 로그인한 뒤, 아래 순서대로 인프라 자원을 먼저 생성합니다.

### ① IAM 역할 (Role) 생성 (S3 접근 권한 부여)
액세스 키를 코드에 하드코딩하지 않고 EC2 서버 자체가 S3 버킷에 안전하게 접근할 수 있도록 IAM 역할을 부여합니다.

1. AWS 콘솔에서 **IAM** 검색 ➡️ 왼쪽 메뉴 **역할(Roles)** 클릭 ➡️ **역할 만들기** 클릭
2. 신뢰할 수 있는 엔터티 유형: **AWS 서비스** 선택
3. 서비스 또는 사례: **EC2** 선택 ➡️ 하단의 EC2 라디오 버튼 체크 후 **다음** 클릭
4. 권한 정책 추가: 검색창에 **`AmazonS3FullAccess`** 검색 후 체크 ➡️ **다음** 클릭
5. 역할 이름: **`skn29-ec2-s3-role`** 입력 ➡️ **역할 생성** 클릭

---

### ② 보안 그룹 (Security Group) 생성
네트워크 방화벽 역할을 하는 보안 그룹을 웹서버용과 DB용으로 각각 생성합니다.

#### 1. 웹서버 보안 그룹 (`skn29-django-sg`)
1. AWS 콘솔에서 **VPC** 또는 **EC2** 검색 ➡️ 왼쪽 메뉴 **보안 그룹** ➡️ **보안 그룹 생성** 클릭
2. 보안 그룹 이름: **`skn29-django-sg`**
3. 설명: `Django Web Server Security Group`
4. **인바운드 규칙(Inbound Rules) 추가**:
   * **규칙 1**: 유형 `SSH` (22 포트) / 소스 `내 IP` (보안상 내 컴퓨터에서만 접속 허용)
   * **규칙 2**: 유형 `HTTP` (80 포트) / 소스 `Anywhere-IPv4` (`0.0.0.0/0`)
   * **규칙 3**: 유형 `HTTPS` (443 포트) / 소스 `Anywhere-IPv4` (`0.0.0.0/0`)
5. **보안 그룹 생성** 클릭

#### 2. 데이터베이스 보안 그룹 (`skn29-rds-sg`)
1. 동일하게 **보안 그룹 생성** 클릭
2. 보안 그룹 이름: **`skn29-rds-sg`**
3. 설명: `PostgreSQL RDS Security Group`
4. **인바운드 규칙 추가**:
   * 유형: **`PostgreSQL`** (5432 포트 자동 지정)
   * 소스: 사용자 지정 선택 후, 우측 검색 칸을 눌러 위에서 만든 **`skn29-django-sg`** 선택 (EC2 웹서버에서만 DB에 들어올 수 있도록 차단 설정)
5. **보안 그룹 생성** 클릭

---

### ③ EC2 인스턴스 생성 및 탄력적 IP 연결

#### 1. EC2 인스턴스 생성
1. AWS 콘솔에서 **EC2** 검색 ➡️ **인스턴스 시작** 클릭
2. 이름: **`skn29-django-server`**
3. AMI(운영체제): **Ubuntu Server 26.04 LTS** (또는 최신 LTS 버전)
4. 인스턴스 유형: **`t3.micro`** (프리티어 대상)
5. 키 페어: 새 키 페어 생성 클릭 ➡️ 이름 **`skn29-key`**, 형식 **`.pem`** ➡️ 키 페어 생성 및 다운로드 (다운로드한 `.pem` 파일은 절대 잃어버리지 않도록 안전한 폴더에 보관)
6. 네트워크 설정 ➡️ **기존 보안 그룹 선택** 체크 ➡️ **`skn29-django-sg`** 보안 그룹 체크
7. 스토리지 구성: 기본 8GB를 프리티어 최대 사양인 **`20GB gp3`**로 변경
8. 인스턴스 시작 클릭

#### 2. 탄력적 IP (Elastic IP) 할당 및 연결 (서버 고정 IP 확보)
*서버를 껐다 켜도 IP 주소가 바뀌지 않도록 고정 IP를 매핑합니다.*
1. EC2 콘솔 왼쪽 메뉴 ➡️ **탄력적 IP** ➡️ **탄력적 IP 주소 할당** 클릭 ➡️ 하단 **할당** 클릭
2. 생성된 탄력적 IP 선택 ➡️ 우측 상단 **작업** ➡️ **탄력적 IP 주소 연결** 클릭
3. 리소스 유형: 인스턴스 ➡️ 인스턴스 검색 칸에서 `skn29-django-server` 선택 ➡️ 하단 **연결** 클릭

---

### ④ RDS (PostgreSQL) 데이터베이스 생성
1. AWS 콘솔에서 **RDS** 검색 ➡️ 왼쪽 메뉴 **데이터베이스** ➡️ **데이터베이스 생성** 클릭
2. 생성 방식: **표준 생성** ➡️ 엔진 옵션: **PostgreSQL** 선택
3. 템플릿: **프리 티어** 선택
4. 설정:
   * DB 인스턴스 식별자: **`skn29-django-db`**
   * 마스터 사용자 이름: **`postgres`**
   * 마스터 암호: 본인이 사용할 **보안성 높은 암호 입력 및 별도 기록**
5. 인스턴스 구성: **`db.t3.micro`** 또는 **`db.t4g.micro`**
6. 스토리지: **`gp3`**, 할당된 스토리지 **`20GB`** (스토리지 자동 조정 활성화는 비용 절감을 위해 체크 해제 권장)
7. 연결:
   * **퍼블릭 액세스**: **아니요** (보안 표준: 외부에서는 접속 불가하게 막고 VPC 내부의 EC2만 우회 연결하도록 설정)
   * VPC 보안 그룹: **기존 항목 선택** ➡️ 기본 지정된 default 그룹 해제 후, 위에서 만든 **`skn29-rds-sg`** 선택
8. **추가 구성** (맨 아래 아코디언 메뉴 클릭하여 펼치기):
   * **초기 데이터베이스 이름**: **`chatbotdb`** 입력 (이 값을 입력해 두어야 장고가 바로 연결하여 테이블을 생성할 수 있는 최초의 공간이 마련됩니다)
9. 맨 아래 **데이터베이스 생성** 클릭 (구축 완료 및 상태가 '사용 가능'으로 변할 때까지 약 5~10분 소요됩니다)

---

### ⑤ S3 버킷 생성 및 버킷 정책 설정
웹 화면을 그리는 정적 리소스(CSS/JS)를 브라우저에 뿌려주기 위해 S3 버킷을 열고 퍼블릭 읽기 권한을 부여합니다.

1. AWS 콘솔에서 **S3** 검색 ➡️ **버킷 만들기** 클릭
2. 버킷 이름: **`skn29-django-static-<본인 고유 식별값>`** 입력 (S3 이름은 전 세계 유일해야 하므로 본인 영문 이름 이니셜 등을 뒤에 붙입니다)
3. 객체 소유권: **ACL 비활성화됨(권장)** 선택
4. **이 버킷의 퍼블릭 액세스 차단 설정**:
   * **`모든 퍼블릭 액세스 차단` 체크 해제**
   * 하단에 나타나는 '현재 설정으로 인해... 알고 있습니다' 체크박스 체크 (사용자의 브라우저가 정적 디자인 파일들을 무리 없이 다운로드해 갈 수 있도록 퍼블릭 읽기 권한의 통로를 열어두는 과정입니다)
5. 맨 아래 **버킷 만들기** 클릭
6. 생성된 버킷 이름을 클릭해 들어간 뒤, 상단 **권한 (Permissions)** 탭 클릭
7. 스크롤을 내려 **버킷 정책 (Bucket policy)** 우측의 **편집(Edit)** 클릭 ➡️ 아래 JSON 복사/붙여넣기:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "PublicReadGetObject",
               "Effect": "Allow",
               "Principal": "*",
               "Action": "s3:GetObject",
               "Resource": "arn:aws:s3:::<자신의_버킷_이름>/*"
           }
       ]
   }
   ```
   * *주의: `<자신의_버킷_이름>` 부분을 실제 본인이 생성한 버킷 명칭으로 수정해야 합니다.*
8. **변경 사항 저장** 클릭

---

## 2. 개발 프로세스 설계 및 데이터 흐름

모든 클라우드 자원의 셋팅이 끝났습니다. 이제 이 자원들을 유기적으로 결합하여 코딩하기 위해 구축할 개발 자동화 아키텍처와 요청 처리 순서는 다음과 같습니다.

### 🔄 개발 작업 흐름 (Workflow)
```
[로컬 PC에서 수정] ──(git push)──> [GitHub Repository] ──(자동 배포)──> [GitHub Actions] 
                                                                               │ (SSH 원격 제어)
                                                                               ▼
                                                                        [AWS EC2 서버]
                                                                  (배포 스크립트 실행 및 반영)
```

### 🖥️ 시스템 아키텍처 및 데이터 흐름
```
[사용자 브라우저]
       ↓ (HTTP 요청)
    [Nginx] (리버스 프록시 / 정적 파일 서빙)
       ↓ (유닉스 소켓: gunicorn.sock)
   [Gunicorn] (WSGI 미들웨어 서버)
       ↓ (장고 앱 구동)
   [Django] 
      ├── [RDS (PostgreSQL)]  ← (데이터베이스 연동)
      ├── [S3 (Bucket)]       ← (정적/미디어 파일 위임)
      └── [OpenAI API]        ← (gpt-5-nano 실시간 추론)
```

---

## 3. [서버 최초 1회] EC2 기본 뼈대 초기 설정

자동 배포 스크립트가 EC2에서 정상적으로 명령을 수행하려면, 최초 1회 서버 디렉토리 구조와 Gunicorn/Nginx의 기본 뼈대가 만들어져 있어야 합니다.

### ① 서버 패키지 설치 및 디렉토리 구성
탄력적 IP(EIP) 주소를 복사한 뒤, 다운로드해 둔 `.pem` 키를 사용해 터미널 또는 MobaXterm으로 EC2 서버에 SSH 접속을 진행합니다.
```bash
# 서버 패키지 업데이트 및 빌드 도구 설치
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-venv python3-dev git build-essential libpq-dev nginx

# 프로젝트 디렉토리 생성 및 가상환경 활성화
mkdir ~/myproject && cd ~/myproject
python3 -m venv venv
source venv/bin/activate

# 필수 패키지 임시 설치 및 최초 프로젝트 생성
pip install django gunicorn
django-admin startproject config .
python manage.py migrate

# settings.py의 ALLOWED_HOSTS에 EC2 퍼블릭 IP 등록 (최초 1회 편집)
nano config/settings.py
# ALLOWED_HOSTS = ['<EC2 탄력적 IP>', 'localhost', '127.0.0.1']
# STATIC_ROOT = BASE_DIR / 'staticfiles'
```

### ② Gunicorn 서비스 생성 (`/etc/systemd/system/gunicorn.service`)
```bash
sudo nano /etc/systemd/system/gunicorn.service
```
아래 내용을 입력하고 저장합니다.
```ini
[Unit]
Description=gunicorn daemon for Django Chatbot
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/myproject
ExecStart=/home/ubuntu/myproject/venv/bin/gunicorn \
          --workers 3 \
          --bind unix:/home/ubuntu/myproject/gunicorn.sock \
          --timeout 120 \
          config.wsgi:application

[Install]
WantedBy=multi-user.target
```
> [!NOTE]
> AI 추론(Inference)은 일반 웹 요청에 비해 응답 지연(Latency)이 발생하므로, Gunicorn 구동 옵션에 `--timeout 120`을 주어 웹서버 연결 끊김(502 Bad Gateway) 에러를 사전에 방지합니다.

### ③ Nginx 리버스 프록시 설정 (`/etc/nginx/sites-available/myproject`)
```bash
sudo nano /etc/nginx/sites-available/myproject
```
아래 설정을 입력하여 포트 80(HTTP) 요청을 Gunicorn 소켓으로 흐르게 프록시 처리합니다.
```nginx
server {
    listen 80;
    server_name <EC2 탄력적 IP>;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /home/ubuntu/myproject/static/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/myproject/gunicorn.sock;
        proxy_read_timeout 120s;
        proxy_connect_timeout 120s;
    }
}
```
설정 반영 및 서비스 활성화:
```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo chmod 755 /home/ubuntu
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
sudo systemctl restart nginx
```

---

## 4. [CI/CD 구축] GitHub Actions 배포 자동화 구현

여기서부터 본격적인 **CI/CD 환경 설정**에 돌입합니다. 이 단계를 마치면 더 이상 EC2 서버에 SSH로 접속해 코드를 만질 필요가 없어집니다.

### ① EC2 프로젝트 Git 초기화 및 최초 Push
서버에 생성된 기본 Django 뼈대 소스코드를 GitHub 저장소에 밀어 넣습니다.
```bash
# EC2 터미널에서 실행
cd ~/myproject
git init
git remote add origin https://github.com/<본인_GitHub_ID>/django-chatbot-preview.git
git branch -M main
# requirements.txt 파일 생성
pip freeze > requirements.txt

# 민감 정보 보호를 위해 .gitignore 작성
echo ".env" >> .gitignore
echo "venv/" >> .gitignore
echo "__pycache__/" >> .gitignore

# 커밋 및 최초 푸시
git add .
git commit -m "Initial skeleton setup"
git push -u origin main
```

### ② GitHub Secrets 등록
GitHub 저장소 페이지 -> **Settings** -> **Secrets and variables** -> **Actions** -> **New repository secret**으로 접속하여 아래 3개의 변수를 등록합니다.

* `EC2_HOST`: EC2 퍼블릭 IP 주소
* `EC2_USER`: `ubuntu`
* `EC2_SSH_KEY`: EC2 접속 시 사용하는 `.pem` 키 내용 전체 복사 (메모장으로 열어 첫 줄부터 끝 줄까지 그대로 복사)

### ③ 로컬 PC로 프로젝트 복제(Clone)
이제 작업공간을 로컬 PC의 IDE(VS Code 등)로 옮깁니다. 로컬 터미널을 열고 코드를 내려받습니다.
```bash
# 로컬 PC 터미널에서 실행
git clone https://github.com/<본인_GitHub_ID>/django-chatbot-preview.git
cd django-chatbot-preview

# 로컬 가상환경 및 패키지 설치
# Windows
python -m venv venv
.\venv\Scripts\activate
pip install -r requirements.txt

# Mac / Linux
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### ④ 배포 워크플로 파일 생성
로컬 PC 프로젝트 루트에 `.github/workflows/deploy.yml` 파일을 생성합니다.
```yaml
name: Deploy Chatbot to EC2

on:
  push:
    branches: [ main ] # main 브랜치로 push 발생 시 배포 구동

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: SSH Connection and Auto Deploy
        uses: appleboy/ssh-action@v1.2.2
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ~/myproject
            source venv/bin/activate
            git pull origin main
            pip install -r requirements.txt
            python manage.py migrate --noinput
            python manage.py collectstatic --noinput
            sudo systemctl restart gunicorn
```

작성이 끝났으면 첫 번째 자동 배포 테스트를 수행합니다.
```bash
git add .
git commit -m "Add GitHub Actions workflow"
git push origin main
```
GitHub 저장소의 **Actions** 탭에서 초록색 체크마크가 뜨고 배포가 끝나는 것을 확인합니다. **이후 모든 과정은 로컬 PC에서만 진행합니다.**

---

## 5. [로컬 작업 1] RDS(PostgreSQL) 연동

이제 로컬 PC에서 장고 설정을 다루며 클라우드 데이터베이스를 연동합니다.

### ① 로컬 패키지 추가
로컬 터미널에서 PostgreSQL 드라이버 및 환경변수 로더를 설치합니다.
```bash
pip install psycopg2-binary python-dotenv
pip freeze > requirements.txt
```

### ② 로컬 `.env` 파일 설정
프로젝트 루트 디렉토리에 로컬 전용 `.env` 파일을 생성하고 AWS RDS 데이터베이스 접속 정보를 기재합니다.
```env
DB_NAME=chatbotdb
DB_USER=postgres
DB_PASSWORD=your_rds_master_password
DB_HOST=your-rds-endpoint.xxxx.ap-northeast-2.rds.amazonaws.com
DB_PORT=5432
```
> [!WARNING]
> `.env` 파일은 절대 Git에 커밋하지 않습니다. 로컬에서 작업한 뒤 배포를 완료하면, **EC2 서버 터미널에도 직접 접속하여 `~/myproject/.env` 파일을 똑같이 수동으로 한 번 작성해 두어야 합니다.**

### ③ `config/settings.py` 수정
```python
import os
from pathlib import Path
from dotenv import load_dotenv

BASE_DIR = Path(__file__).resolve().parent.parent
load_dotenv(os.path.join(BASE_DIR, '.env'))

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}
```

### ④ Git Push 및 자동 배포
```bash
git add .
git commit -m "Configure RDS PostgreSQL connection"
git push origin main
```
배포가 완료되면 Actions가 알아서 EC2 서버 상에서 `python manage.py migrate`를 돌려 RDS 테이블 스키마를 초기화합니다.

---

## 6. [로컬 작업 2] S3 정적/미디어 파일 연동

정적 자산과 업로드 이미지를 AWS S3 스토리지로 이관합니다.

### ① 패키지 추가 설치
```bash
pip install django-storages boto3
pip freeze > requirements.txt
```

### ② `config/settings.py` 스토리지 설정 변경
```python
# settings.py 상단 INSTALLED_APPS에 'storages' 앱을 추가합니다.
INSTALLED_APPS = [
    # ... 기본 앱 ...
    'storages',
]

# S3 버킷 설정 (EC2에 S3 FullAccess IAM 역할이 매핑되어 있으므로 Access Key는 기재 불필요)
AWS_STORAGE_BUCKET_NAME = 'your-s3-bucket-name' # 자신이 실제 생성한 버킷명 기재
AWS_S3_REGION_NAME = 'ap-northeast-2'
AWS_QUERYSTRING_AUTH = False 

STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
    },
    "staticfiles": {
        "BACKEND": "storages.backends.s3boto3.S3StaticStorage",
    },
}
```

### ③ Git Push 및 자동 배포
```bash
git add .
git commit -m "Configure AWS S3 storage backend"
git push origin main
```
자동 배포가 돌면서 `collectstatic` 명령어를 통해 장고의 기본 관리자 페이지 디자인 자산(`admin/`)들이 S3 버킷으로 업로드되는 것을 S3 콘솔에서 확인할 수 있습니다.

---

## 7. [로컬 작업 3] OpenAI `gpt-5-nano` AI 추론 연동

이제 마지막 단계로 AI 추론을 받아 답변하는 핵심 챗봇 백엔드를 연동합니다.

### ① 패키지 설치 및 앱 생성
```bash
pip install openai
pip freeze > requirements.txt

# 로컬에서 chat 앱 생성
python manage.py startapp chat
```
`config/settings.py`의 `INSTALLED_APPS`에 `'chat',`을 추가하여 등록해 줍니다.

### ② `.env` 파일에 OpenAI API Key 추가 (로컬 및 EC2 각각 추가 필요)
* 로컬의 `.env` 및 EC2 서버의 `.env` 파일 하단에 API 키를 기록합니다.
```env
OPENAI_API_KEY=sk-proj-your-openai-api-key-here
```

### ③ AI 추론 뷰 구현 (`chat/views.py`)
OpenAI 최신 API 가이드라인(SDK v1.0.0+ 기준) 및 최신 경량 고효율 모델 `gpt-5-nano`에 맞춰 소스코드를 작성합니다. 시스템 페르소나 지시사항에는 `system` 대신 최신의 **`developer`** 역할을 적용했습니다.

```python
import os
import json
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_http_methods
from openai import OpenAI

# 클라이언트 인스턴스화 (환경변수의 OPENAI_API_KEY를 참조합니다)
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

@csrf_exempt
@require_http_methods(["POST"])
def chat_view(request):
    """
    사용자의 질문을 수신하여 OpenAI gpt-5-nano 모델로 추론을 위임 및 응답하는 API
    """
    try:
        body = json.loads(request.body)
        user_message = body.get("message", "").strip()
        
        if not user_message:
            return JsonResponse({"error": "메시지 필드가 비어있습니다."}, status=400)
            
        # gpt-5-nano 모델 호출 및 추론
        # 최신 가이드를 적용하여 system -> developer 역할로 프롬프트 설정 주입
        response = client.chat.completions.create(
            model="gpt-5-nano",
            messages=[
                {
                    "role": "developer", 
                    "content": "너는 AWS 환경 위에 배포된 똑똑하고 명쾌한 AI 챗봇이야. 항상 정중한 한국어로 간략히 답변해줘."
                },
                {
                    "role": "user", 
                    "content": user_message
                }
            ],
            temperature=0.7,
            max_tokens=500
        )
        
        bot_response = response.choices[0].message.content
        return JsonResponse({
            "status": "success",
            "answer": bot_response
        })
        
    except json.JSONDecodeError:
        return JsonResponse({"error": "유효한 JSON 포맷이 아닙니다."}, status=400)
    except Exception as e:
        return JsonResponse({"error": f"추론 연동 실패: {str(e)}"}, status=500)
```

### ④ 라우팅 설정 (`chat/urls.py` & `config/urls.py`)

* **`chat/urls.py`**
```python
from django.urls import path
from . import views

urlpatterns = [
    path('api/chat/', views.chat_view, name='chat_view'),
]
```

* **`config/urls.py`**
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('chat.urls')),
]
```

### ⑤ Git Push 및 최종 자동 배포
```bash
git add .
git commit -m "Implement gpt-5-nano chatbot API"
git push origin main
```
자동 배포 프로세스가 정상적으로 통과되면 챗봇 API 탑재가 실서버에 반영됩니다.

---

## 8. AI 추론 검증 및 테스트 (cURL)

서버 배포가 모두 완료되었으면 로컬 터미널을 열고 다음 cURL 요청을 실서버의 IP로 쏘아 AI 추론이 원활히 동작하는지 최종 검증합니다.

```bash
curl -X POST http://<EC2_PUBLIC_IP>/api/chat/ \
  -H "Content-Type: application/json" \
  -d '{"message": "안녕! AWS에서 너를 호출하고 있어. 사용 중인 OpenAI 모델이 뭔지 명시해서 답변해줘."}'
```

### 예상 응답 JSON
```json
{
  "status": "success",
  "answer": "안녕하세요! AWS 환경에서 보내주신 메시지를 성공적으로 수신했습니다. 저는 현재 OpenAI의 최신 고효율 모델인 `gpt-5-nano` 모델을 통해 추론을 진행하고 있습니다. 필요하신 사항이 있으시면 언제든지 말씀해 주세요."
}
```

이 검증을 마치면 성공입니다!
로컬에서 개발하고 푸시하기만 하면 즉시 AWS 상에서 AI가 똑똑하게 반응하는 **"CI/CD 기반의 현대적인 AI 애플리케이션 개발 사이클"**의 첫 단추를 제대로 꿴 것입니다. 이제 이 뼈대 위에 RAG와 에이전트 설계 이론들을 결합해 나가겠습니다.
