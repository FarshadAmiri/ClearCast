# راهنمای جامع API - پلتفرم ClearCast
## مستندات کامل سرویس‌های وب

**نسخه API:** v1  
**تاریخ بروزرسانی:** بهمن ۱۴۰۴

---

## 📋 فهرست مطالب

1. [مقدمه](#مقدمه)
2. [احراز هویت](#احراز-هویت)
3. [فرمت‌های پاسخ](#فرمتهای-پاسخ)
4. [کدهای خطا](#کدهای-خطا)
5. [API سرویس حذف نویز](#api-سرویس-حذف-نویز)
6. [API سرویس جداسازی آواز](#api-سرویس-جداسازی-آواز)
7. [API سرویس تقویت صدا](#api-سرویس-تقویت-صدا)
8. [API سرویس استخراج گوینده](#api-سرویس-استخراج-گوینده)
9. [API سرویس دایرایزیشن](#api-سرویس-دایرایزیشن)
10. [نمونه کدهای کامل](#نمونه-کدهای-کامل)

---

## 🎯 مقدمه

### Base URL
```
http://your-domain.com
https://your-domain.com  (Production)
```

### پروتکل‌های پشتیبانی شده
- HTTP/1.1
- HTTPS (توصیه شده برای Production)
- WebSocket (برای استریم Real-time)

### Content Types
- **Request**: `multipart/form-data` (برای آپلود فایل)
- **Response**: `application/json`

---

## 🔐 احراز هویت

### روش ۱: Session-based Authentication (Django)

پس از لاگین، Django یک session cookie ایجاد می‌کند که باید در تمام درخواست‌ها ارسال شود.

**Login API:**
```http
POST /accounts/login/
Content-Type: application/x-www-form-urlencoded

username=your_username&password=your_password
```

**Response:**
```json
{
  "status": "success",
  "user": {
    "id": 1,
    "username": "your_username",
    "email": "user@example.com"
  }
}
```

پس از لاگین موفق، cookie به صورت خودکار Set می‌شود.

### روش ۲: CSRF Token

برای تمام درخواست‌های POST/PUT/DELETE، باید CSRF Token ارسال شود:

**Header:**
```
X-CSRFToken: <your-csrf-token>
```

**دریافت CSRF Token:**

**روش 1: از Cookie**
```javascript
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');
```

**روش 2: از Meta Tag (در صفحات HTML)**
```html
<meta name="csrf-token" content="{{ csrf_token }}">
```

```javascript
const csrftoken = document.querySelector('meta[name="csrf-token"]').content;
```

### مثال کامل درخواست با احراز هویت

```python
import requests

session = requests.Session()

# Login
login_data = {
    'username': 'your_username',
    'password': 'your_password'
}
login_response = session.post('http://your-domain.com/accounts/login/', data=login_data)

# Get CSRF Token
csrf_token = session.cookies.get('csrftoken')

# Make authenticated request
headers = {'X-CSRFToken': csrf_token}
files = {'audio_file': open('sample.mp3', 'rb')}
response = session.post('http://your-domain.com/api/denoise/upload/', files=files, headers=headers)

print(response.json())
```

---

## 📦 فرمت‌های پاسخ

### پاسخ موفق

```json
{
  "status": "success",
  "message": "عملیات با موفقیت انجام شد",
  "data": {
    "file_id": 123,
    "filename": "audio.mp3",
    "processing_status": "completed"
  }
}
```

### پاسخ خطا

```json
{
  "status": "error",
  "message": "توضیح خطا",
  "error_code": "ERROR_CODE",
  "details": {
    "field": "مشکل مربوط به این فیلد"
  }
}
```

---

## ⚠️ کدهای خطا

| کد HTTP | کد خطا | توضیح |
|---------|---------|-------|
| 200 | SUCCESS | عملیات موفق |
| 400 | BAD_REQUEST | درخواست نامعتبر |
| 401 | UNAUTHORIZED | احراز هویت نشده |
| 403 | FORBIDDEN | دسترسی غیرمجاز |
| 404 | NOT_FOUND | منبع پیدا نشد |
| 413 | FILE_TOO_LARGE | فایل بیش از حد بزرگ |
| 415 | UNSUPPORTED_MEDIA | فرمت فایل پشتیبانی نمی‌شود |
| 500 | SERVER_ERROR | خطای سرور |
| 503 | SERVICE_UNAVAILABLE | سرویس در دسترس نیست |

---

## 🎵 API سرویس حذف نویز

### 1. آپلود فایل برای حذف نویز

**Endpoint:** `POST /api/denoise/upload/`

**توضیح:** آپلود فایل صوتی برای حذف نویز با امکان تقویت صدا

**پارامترها:**

| نام | نوع | الزامی | توضیح |
|-----|-----|--------|-------|
| audio_file | File | ✅ | فایل صوتی (MP3, WAV, M4A, FLAC) |
| boost_level | String | ❌ | سطح تقویت: 'none', '2x', '3x', '4x', '5x' |

**حداکثر اندازه فایل:** 200MB (پیش‌فرض)

**Request (cURL):**
```bash
curl -X POST http://your-domain.com/api/denoise/upload/ \
  -H "X-CSRFToken: your-csrf-token" \
  -F "audio_file=@audio.mp3" \
  -F "boost_level=2x" \
  -b "sessionid=your-session-id"
```

**Request (Python requests):**
```python
import requests

url = "http://your-domain.com/api/denoise/upload/"
files = {'audio_file': open('audio.mp3', 'rb')}
data = {'boost_level': '2x'}
headers = {'X-CSRFToken': csrf_token}
cookies = {'sessionid': session_id}

response = requests.post(url, files=files, data=data, headers=headers, cookies=cookies)
result = response.json()
print(result)
```

**Request (JavaScript Fetch):**
```javascript
const formData = new FormData();
formData.append('audio_file', fileInput.files[0]);
formData.append('boost_level', '2x');

fetch('/api/denoise/upload/', {
    method: 'POST',
    body: formData,
    headers: {
        'X-CSRFToken': getCookie('csrftoken')
    },
    credentials: 'same-origin'
})
.then(response => response.json())
.then(data => {
    console.log('File ID:', data.file_id);
    console.log('Status:', data.message);
});
```

**Response موفق (200 OK):**
```json
{
  "status": "success",
  "message": "فایل با موفقیت آپلود شد. پردازش آغاز شد.",
  "file_id": 123,
  "filename": "audio.mp3",
  "boost_level": "2x"
}
```

**Response خطا:**

```json
// خطای عدم ارسال فایل
{
  "status": "error",
  "message": "هیچ فایل صوتی ارسال نشد"
}

// خطای فرمت نامعتبر
{
  "status": "error",
  "message": "فرمت فایل پشتیبانی نمی‌شود. لطفاً MP3, WAV, M4A یا FLAC آپلود کنید."
}

// خطای اندازه فایل
{
  "status": "error",
  "message": "اندازه فایل بیش از حد مجاز است (حداکثر 200MB)"
}
```

---

### 2. دریافت لیست فایل‌های پردازش شده

**Endpoint:** `GET /api/denoise/files/`

**توضیح:** دریافت لیست تمام فایل‌های آپلود شده و وضعیت آن‌ها

**Request (cURL):**
```bash
curl -X GET http://your-domain.com/api/denoise/files/ \
  -b "sessionid=your-session-id"
```

**Request (Python):**
```python
response = requests.get('http://your-domain.com/api/denoise/files/', cookies=cookies)
files_list = response.json()
```

**Request (JavaScript):**
```javascript
fetch('/api/denoise/files/', {
    credentials: 'same-origin'
})
.then(response => response.json())
.then(data => {
    data.files.forEach(file => {
        console.log(`${file.filename}: ${file.status}`);
    });
});
```

**Response (200 OK):**
```json
{
  "files": [
    {
      "id": 123,
      "filename": "audio.mp3",
      "original_file": "/media/uploads/original/audio.mp3",
      "denoised_file": "/media/uploads/denoised/audio_denoised.mp3",
      "status": "completed",
      "boost_level": "2x",
      "uploaded_at": "2024-02-18T10:30:00Z",
      "processed_at": "2024-02-18T10:32:15Z",
      "file_size": 5242880,
      "duration": 180.5
    },
    {
      "id": 124,
      "filename": "podcast.wav",
      "original_file": "/media/uploads/original/podcast.wav",
      "denoised_file": null,
      "status": "processing",
      "boost_level": "none",
      "uploaded_at": "2024-02-18T11:00:00Z",
      "processed_at": null,
      "file_size": 10485760,
      "duration": 300.0
    }
  ],
  "total_count": 2
}
```

**وضعیت‌های ممکن:**
- `pending`: در انتظار پردازش
- `processing`: در حال پردازش
- `completed`: پردازش کامل شده
- `failed`: پردازش با خطا مواجه شده

---

### 3. دریافت اطلاعات یک فایل خاص

**Endpoint:** `GET /api/denoise/files/<file_id>/`

**Request:**
```bash
curl -X GET http://your-domain.com/api/denoise/files/123/ \
  -b "sessionid=your-session-id"
```

**Response:**
```json
{
  "id": 123,
  "filename": "audio.mp3",
  "original_file": "/media/uploads/original/audio.mp3",
  "denoised_file": "/media/uploads/denoised/audio_denoised.mp3",
  "status": "completed",
  "boost_level": "2x",
  "uploaded_at": "2024-02-18T10:30:00Z",
  "processed_at": "2024-02-18T10:32:15Z",
  "file_size": 5242880,
  "duration": 180.5,
  "processing_time": 135.2
}
```

---

### 4. دانلود فایل پردازش شده

**Endpoint:** `GET /api/denoise/download/<file_id>/`

**Request:**
```bash
curl -X GET http://your-domain.com/api/denoise/download/123/ \
  -b "sessionid=your-session-id" \
  -o denoised_audio.mp3
```

**Request (Python):**
```python
response = requests.get(f'http://your-domain.com/api/denoise/download/{file_id}/', 
                       cookies=cookies, 
                       stream=True)

with open('denoised_audio.mp3', 'wb') as f:
    for chunk in response.iter_content(chunk_size=8192):
        f.write(chunk)
```

**Request (JavaScript):**
```javascript
fetch(`/api/denoise/download/${fileId}/`, {
    credentials: 'same-origin'
})
.then(response => response.blob())
.then(blob => {
    const url = window.URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'denoised_audio.mp3';
    a.click();
});
```

---

### 5. حذف فایل

**Endpoint:** `DELETE /api/denoise/files/<file_id>/`

**Request:**
```bash
curl -X DELETE http://your-domain.com/api/denoise/files/123/ \
  -H "X-CSRFToken: your-csrf-token" \
  -b "sessionid=your-session-id"
```

**Response:**
```json
{
  "status": "success",
  "message": "فایل با موفقیت حذف شد"
}
```

---

## 🎤 API سرویس جداسازی آواز

### 1. آپلود فایل برای جداسازی آواز

**Endpoint:** `POST /api/vocal-separation/upload/`

**پارامترها:**

| نام | نوع | الزامی | توضیح |
|-----|-----|--------|-------|
| audio_file | File | ✅ | فایل موزیک/آهنگ |
| output_type | String | ✅ | 'vocals' یا 'instrumental' |

**Request (Python):**
```python
files = {'audio_file': open('song.mp3', 'rb')}
data = {'output_type': 'vocals'}
headers = {'X-CSRFToken': csrf_token}

response = requests.post(
    'http://your-domain.com/api/vocal-separation/upload/',
    files=files,
    data=data,
    headers=headers,
    cookies=cookies
)
```

**Request (cURL):**
```bash
curl -X POST http://your-domain.com/api/vocal-separation/upload/ \
  -H "X-CSRFToken: your-csrf-token" \
  -F "audio_file=@song.mp3" \
  -F "output_type=vocals" \
  -b "sessionid=your-session-id"
```

**Response (200 OK):**
```json
{
  "status": "success",
  "message": "فایل آپلود شد. جداسازی آواز آغاز شد.",
  "file_id": 456,
  "filename": "song.mp3",
  "output_type": "vocals"
}
```

---

### 2. دریافت لیست فایل‌های جداسازی شده

**Endpoint:** `GET /api/vocal-separation/files/`

**Response:**
```json
{
  "files": [
    {
      "id": 456,
      "filename": "song.mp3",
      "original_file": "/media/uploads/original/song.mp3",
      "separated_file": "/media/uploads/vocal_separation/song_vocals.mp3",
      "output_type": "vocals",
      "status": "completed",
      "uploaded_at": "2024-02-18T12:00:00Z",
      "processed_at": "2024-02-18T12:05:30Z",
      "duration": 240.0
    }
  ]
}
```

---

### 3. دانلود فایل جداسازی شده

**Endpoint:** `GET /api/vocal-separation/download/<file_id>/`

**Request:**
```python
response = requests.get(
    f'http://your-domain.com/api/vocal-separation/download/{file_id}/',
    cookies=cookies,
    stream=True
)

with open('vocals_only.mp3', 'wb') as f:
    for chunk in response.iter_content(chunk_size=8192):
        f.write(chunk)
```

---

## 📢 API سرویس تقویت صدا

### 1. آپلود فایل برای تقویت صدا

**Endpoint:** `POST /api/audio-boost/upload/`

**پارامترها:**

| نام | نوع | الزامی | توضیح |
|-----|-----|--------|-------|
| audio_file | File | ✅ | فایل صوتی |
| boost_level | String | ✅ | '2x', '3x', '4x', '5x' |

**Request (Python):**
```python
files = {'audio_file': open('quiet_audio.mp3', 'rb')}
data = {'boost_level': '3x'}
headers = {'X-CSRFToken': csrf_token}

response = requests.post(
    'http://your-domain.com/api/audio-boost/upload/',
    files=files,
    data=data,
    headers=headers,
    cookies=cookies
)
```

**Response:**
```json
{
  "status": "success",
  "message": "فایل آپلود شد. تقویت صدا آغاز شد.",
  "file_id": 789,
  "filename": "quiet_audio.mp3",
  "boost_level": "3x"
}
```

---

## 👤 API سرویس استخراج گوینده

### 1. آپلود فایل برای استخراج گوینده

**Endpoint:** `POST /api/speaker-extraction/upload/`

**پارامترها:**

| نام | نوع | الزامی | توضیح |
|-----|-----|--------|-------|
| audio_file | File | ✅ | فایل صوتی اصلی (چند گوینده) |
| speaker_sample | File | ✅ | نمونه صوتی از گوینده مورد نظر |
| boost_level | String | ❌ | تقویت صدای خروجی: 'none', '2x', '3x', '4x', '5x' |

**Request (Python):**
```python
files = {
    'audio_file': open('meeting.mp3', 'rb'),
    'speaker_sample': open('john_voice.mp3', 'rb')
}
data = {'boost_level': '2x'}
headers = {'X-CSRFToken': csrf_token}

response = requests.post(
    'http://your-domain.com/api/speaker-extraction/upload/',
    files=files,
    data=data,
    headers=headers,
    cookies=cookies
)
```

**Request (JavaScript):**
```javascript
const formData = new FormData();
formData.append('audio_file', mainAudioFile);
formData.append('speaker_sample', speakerSampleFile);
formData.append('boost_level', '2x');

fetch('/api/speaker-extraction/upload/', {
    method: 'POST',
    body: formData,
    headers: {
        'X-CSRFToken': getCookie('csrftoken')
    },
    credentials: 'same-origin'
})
.then(response => response.json())
.then(data => console.log(data));
```

**Response:**
```json
{
  "status": "success",
  "message": "فایل‌ها آپلود شدند. استخراج گوینده آغاز شد.",
  "file_id": 321,
  "filename": "meeting.mp3",
  "speaker_sample_name": "john_voice.mp3"
}
```

---

## 🎙 API سرویس دایرایزیشن

این سرویس از طریق Python API در دسترس است (نه REST API).

### استفاده از Python Module

**نصب:**
```bash
cd Nemo-diarization
pip install -r requirements.txt
```

**Import:**
```python
from main import process_meeting_audio, quick_diarize
from voice_enrollment import create_voice_database
```

---

### 1. ایجاد پایگاه داده گویندگان

```python
from voice_enrollment import create_voice_database

# تعریف نمونه‌های صوتی
speaker_samples = {
    "علی": ["ali_sample1.wav", "ali_sample2.wav"],
    "سارا": ["sara_sample1.wav", "sara_sample2.wav"],
    "حسین": ["hosein_sample1.wav"]
}

# ایجاد پایگاه داده
db_path = create_voice_database(
    speaker_samples,
    output_db_path="speakers_db.json"
)

print(f"پایگاه داده در {db_path} ایجاد شد")
```

---

### 2. دایرایزیشن ساده (بدون شناسایی)

```python
from main import quick_diarize

# دایرایزیشن بدون شناسایی گویندگان
segments = quick_diarize(
    audio_path="meeting.mp3",
    num_speakers=4,  # تعداد گویندگان (اختیاری)
    output_path="diarization_result.json"
)

# نمایش نتایج
for seg in segments:
    print(f"{seg['speaker']}: {seg['start']:.2f}s - {seg['end']:.2f}s")
```

**خروجی:**
```
SPEAKER_0: 0.00s - 23.50s
SPEAKER_1: 23.50s - 45.20s
SPEAKER_0: 45.20s - 67.80s
SPEAKER_2: 67.80s - 89.00s
```

---

### 3. دایرایزیشن کامل با شناسایی و ترنسکریپشن

```python
from main import process_meeting_audio

result = process_meeting_audio(
    meeting_audio_path="podcast.mp3",
    voice_embeddings_database_path="speakers_db.json",
    expected_language="fa",  # فارسی
    output_transcriptions=True,
    transcriptor_model_path="medium",  # مدل Whisper
    output_dir="output/",
    num_speakers=3
)

# نمایش نتایج
print("گویندگان شناسایی شده:", result['identified_speakers'])
print("تعداد سگمنت:", len(result['segments']))

# نمایش چند سگمنت اول
for seg in result['segments'][:3]:
    print(f"\n{seg['speaker']} ({seg['start']:.1f}s - {seg['end']:.1f}s):")
    if 'text' in seg:
        print(f"  {seg['text']}")
```

**خروجی:**
```
گویندگان شناسایی شده: ['علی', 'سارا', 'حسین']
تعداد سگمنت: 45

علی (0.0s - 12.5s):
  سلام، امروز می‌خواهیم در مورد پروژه جدید صحبت کنیم.

سارا (12.5s - 28.3s):
  بله، من فکر می‌کنم باید ابتدا در مورد بودجه بحث کنیم.

علی (28.3s - 42.8s):
  موافقم. پیشنهاد من این است که...
```

---

### 4. تنظیمات پیشرفته

```python
from main import process_meeting_audio

result = process_meeting_audio(
    meeting_audio_path="meeting.wav",
    voice_embeddings_database_path="speakers_db.json",
    expected_language="en",
    output_transcriptions=True,
    transcriptor_model_path="/path/to/whisper/model",
    output_dir="results/",
    num_speakers=None,  # تشخیص خودکار تعداد گویندگان
    window_size=1.5,  # اندازه پنجره (ثانیه)
    hop_size=0.5,  # جهش پنجره (ثانیه)
    output_formats=['json', 'txt', 'srt', 'vtt']  # فرمت‌های خروجی
)

# فایل‌های ایجاد شده:
# - results/meeting_diarization.json
# - results/meeting_diarization.txt
# - results/meeting_diarization.srt
# - results/meeting_diarization.vtt
```

**فرمت خروجی JSON:**
```json
{
  "audio_file": "meeting.wav",
  "duration": 1800.5,
  "num_speakers": 3,
  "identified_speakers": ["علی", "سارا", "حسین"],
  "segments": [
    {
      "id": 1,
      "start": 0.0,
      "end": 12.5,
      "speaker": "علی",
      "identified": true,
      "confidence": 0.95,
      "text": "سلام، امروز می‌خواهیم..."
    }
  ]
}
```

**فرمت خروجی SRT:**
```
1
00:00:00,000 --> 00:00:12,500
[علی] سلام، امروز می‌خواهیم در مورد پروژه جدید صحبت کنیم.

2
00:00:12,500 --> 00:00:28,300
[سارا] بله، من فکر می‌کنم باید ابتدا در مورد بودجه بحث کنیم.
```

---

## 💡 نمونه کدهای کامل

### مثال ۱: Client کامل Python

```python
import requests
from pathlib import Path

class ClearCastClient:
    def __init__(self, base_url, username, password):
        self.base_url = base_url
        self.session = requests.Session()
        self.csrf_token = None
        self.login(username, password)
    
    def login(self, username, password):
        """ورود به سیستم"""
        data = {'username': username, 'password': password}
        response = self.session.post(f'{self.base_url}/accounts/login/', data=data)
        
        if response.status_code == 200:
            self.csrf_token = self.session.cookies.get('csrftoken')
            print("✓ ورود موفق")
        else:
            raise Exception("خطا در ورود")
    
    def denoise_file(self, audio_path, boost_level='none'):
        """حذف نویز از فایل"""
        headers = {'X-CSRFToken': self.csrf_token}
        files = {'audio_file': open(audio_path, 'rb')}
        data = {'boost_level': boost_level}
        
        response = self.session.post(
            f'{self.base_url}/api/denoise/upload/',
            files=files,
            data=data,
            headers=headers
        )
        
        return response.json()
    
    def get_file_status(self, file_id):
        """بررسی وضعیت پردازش"""
        response = self.session.get(f'{self.base_url}/api/denoise/files/{file_id}/')
        return response.json()
    
    def download_denoised(self, file_id, output_path):
        """دانلود فایل پردازش شده"""
        response = self.session.get(
            f'{self.base_url}/api/denoise/download/{file_id}/',
            stream=True
        )
        
        with open(output_path, 'wb') as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
        
        print(f"✓ فایل در {output_path} ذخیره شد")
    
    def separate_vocals(self, audio_path, output_type='vocals'):
        """جداسازی آواز"""
        headers = {'X-CSRFToken': self.csrf_token}
        files = {'audio_file': open(audio_path, 'rb')}
        data = {'output_type': output_type}
        
        response = self.session.post(
            f'{self.base_url}/api/vocal-separation/upload/',
            files=files,
            data=data,
            headers=headers
        )
        
        return response.json()
    
    def extract_speaker(self, audio_path, speaker_sample_path, boost_level='none'):
        """استخراج گوینده"""
        headers = {'X-CSRFToken': self.csrf_token}
        files = {
            'audio_file': open(audio_path, 'rb'),
            'speaker_sample': open(speaker_sample_path, 'rb')
        }
        data = {'boost_level': boost_level}
        
        response = self.session.post(
            f'{self.base_url}/api/speaker-extraction/upload/',
            files=files,
            data=data,
            headers=headers
        )
        
        return response.json()


# استفاده
if __name__ == "__main__":
    # ایجاد client
    client = ClearCastClient(
        base_url="http://localhost:8000",
        username="your_username",
        password="your_password"
    )
    
    # حذف نویز
    result = client.denoise_file("noisy_audio.mp3", boost_level="2x")
    file_id = result['file_id']
    print(f"فایل آپلود شد. ID: {file_id}")
    
    # چک کردن وضعیت
    import time
    while True:
        status = client.get_file_status(file_id)
        print(f"وضعیت: {status['status']}")
        
        if status['status'] == 'completed':
            break
        elif status['status'] == 'failed':
            print("خطا در پردازش")
            break
        
        time.sleep(2)
    
    # دانلود نتیجه
    client.download_denoised(file_id, "denoised_output.mp3")
```

---

### مثال ۲: استفاده در JavaScript/Node.js

```javascript
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

class ClearCastClient {
    constructor(baseURL) {
        this.baseURL = baseURL;
        this.axios = axios.create({
            baseURL: baseURL,
            withCredentials: true
        });
        this.csrfToken = null;
    }
    
    async login(username, password) {
        try {
            const response = await this.axios.post('/accounts/login/', 
                `username=${username}&password=${password}`,
                {
                    headers: {
                        'Content-Type': 'application/x-www-form-urlencoded'
                    }
                }
            );
            
            // Get CSRF token from cookies
            const cookies = response.headers['set-cookie'];
            const csrfCookie = cookies.find(c => c.startsWith('csrftoken='));
            this.csrfToken = csrfCookie.split(';')[0].split('=')[1];
            
            console.log('✓ Login successful');
            return true;
        } catch (error) {
            console.error('Login failed:', error.message);
            return false;
        }
    }
    
    async denoiseFile(audioPath, boostLevel = 'none') {
        const formData = new FormData();
        formData.append('audio_file', fs.createReadStream(audioPath));
        formData.append('boost_level', boostLevel);
        
        try {
            const response = await this.axios.post('/api/denoise/upload/', formData, {
                headers: {
                    ...formData.getHeaders(),
                    'X-CSRFToken': this.csrfToken
                }
            });
            
            return response.data;
        } catch (error) {
            console.error('Upload failed:', error.message);
            throw error;
        }
    }
    
    async getFileStatus(fileId) {
        const response = await this.axios.get(`/api/denoise/files/${fileId}/`);
        return response.data;
    }
    
    async downloadDenoised(fileId, outputPath) {
        const response = await this.axios.get(`/api/denoise/download/${fileId}/`, {
            responseType: 'stream'
        });
        
        const writer = fs.createWriteStream(outputPath);
        response.data.pipe(writer);
        
        return new Promise((resolve, reject) => {
            writer.on('finish', resolve);
            writer.on('error', reject);
        });
    }
}

// Usage
(async () => {
    const client = new ClearCastClient('http://localhost:8000');
    
    // Login
    await client.login('your_username', 'your_password');
    
    // Upload for denoising
    const result = await client.denoiseFile('noisy_audio.mp3', '2x');
    const fileId = result.file_id;
    console.log(`File uploaded. ID: ${fileId}`);
    
    // Poll status
    let status;
    do {
        await new Promise(resolve => setTimeout(resolve, 2000));
        status = await client.getFileStatus(fileId);
        console.log(`Status: ${status.status}`);
    } while (status.status === 'processing' || status.status === 'pending');
    
    // Download result
    if (status.status === 'completed') {
        await client.downloadDenoised(fileId, 'denoised_output.mp3');
        console.log('✓ File downloaded');
    }
})();
```

---

## 📱 نکات مهم و Best Practices

### 1. مدیریت Session
- Session ها پس از ۱۴ روز عدم فعالیت منقضی می‌شوند
- در صورت دریافت خطای ۴۰۱، مجدداً لاگین کنید

### 2. Rate Limiting
- حداکثر ۱۰۰ درخواست در دقیقه
- حداکثر ۱۰ آپلود همزمان

### 3. پردازش Async
- پردازش فایل‌ها Asynchronous است
- از Polling یا WebSocket برای دریافت نتیجه استفاده کنید

### 4. مدیریت خطا
- همیشه response status code را بررسی کنید
- پیام‌های خطا در فیلد `message` قرار دارند

### 5. امنیت
- همیشه از HTTPS در Production استفاده کنید
- CSRF Token را در تمام POST requests ارسال کنید
- Session cookies را ایمن نگه دارید

---

**© 2024-2026 ClearCast Team**
