# Chuyên Đề 29 — Xử Lý Ảnh, Âm Thanh, Video

## 29.1 Pillow — thao tác ảnh cơ bản

```bash
pip install Pillow
```

```python
from PIL import Image, ImageFilter, ImageDraw, ImageFont, ImageOps

img = Image.open("anh.jpg")
print(img.size, img.mode, img.format)

img.resize((800, 600)).save("resize.jpg", quality=85)
img.rotate(90, expand=True).save("rotate.jpg")
img.convert("L").save("gray.jpg")

img.filter(ImageFilter.GaussianBlur(5)).save("blur.jpg")
img.filter(ImageFilter.SHARPEN).save("sharp.jpg")

ImageOps.equalize(img).save("equalized.jpg")
ImageOps.autocontrast(img).save("contrast.jpg")
```

### Vẽ

```python
draw = ImageDraw.Draw(img)
draw.rectangle([10, 10, 200, 100], outline="red", width=3)
draw.text((20, 20), "Xin chào", fill="white")
img.save("drawn.jpg")
```

### Crop và paste

```python
vung = img.crop((100, 100, 400, 400))
img.paste(vung, (0, 0))
```

### EXIF orientation

```python
from PIL import ImageOps

img = ImageOps.exif_transpose(Image.open("anh.jpg"))
```

Xử lý ảnh chụp từ điện thoại bị xoay.

### Thumbnail

```python
img = Image.open("anh.jpg")
img.thumbnail((200, 200))   # giữ tỉ lệ, tại chỗ
img.save("thumb.jpg")
```

## 29.2 Watermark

```python
from PIL import Image, ImageDraw, ImageFont

def watermark(path: str, chu: str, out: str) -> None:
    img = Image.open(path).convert("RGBA")
    lop = Image.new("RGBA", img.size, (0, 0, 0, 0))
    draw = ImageDraw.Draw(lop)
    font = ImageFont.truetype("arial.ttf", 40)
    rong, cao = draw.textbbox((0, 0), chu, font=font)[2:]
    x = img.width - rong - 20
    y = img.height - cao - 20
    draw.text((x, y), chu, font=font, fill=(255, 255, 255, 128))
    Image.alpha_composite(img, lop).convert("RGB").save(out)
```

## 29.3 OpenCV — xử lý ảnh nâng cao

```bash
pip install opencv-python
```

```python
import cv2
import numpy as np

img = cv2.imread("anh.jpg")       # BGR
print(img.shape)                   # (cao, rộng, 3)

cv2.imwrite("out.jpg", img)

gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
blur = cv2.GaussianBlur(gray, (5, 5), 0)
edges = cv2.Canny(blur, 100, 200)

cv2.imwrite("edges.jpg", edges)
```

### Resize và crop

```python
resize = cv2.resize(img, (800, 600), interpolation=cv2.INTER_AREA)
crop = img[100:400, 200:500]
```

### Vẽ

```python
cv2.rectangle(img, (10, 10), (200, 100), (0, 255, 0), 2)
cv2.putText(img, "Hello", (20, 50), cv2.FONT_HERSHEY_SIMPLEX, 1, (255, 255, 255), 2)
```

### Threshold

```python
_, binary = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY)
adaptive = cv2.adaptiveThreshold(gray, 255, cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
                                 cv2.THRESH_BINARY, 11, 2)
```

### Contour — tìm vật thể

```python
contours, _ = cv2.findContours(binary, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
for c in contours:
    if cv2.contourArea(c) > 500:
        x, y, w, h = cv2.boundingRect(c)
        cv2.rectangle(img, (x, y), (x + w, y + h), (0, 255, 0), 2)
```

### Video

```python
cap = cv2.VideoCapture("video.mp4")
fps = cap.get(cv2.CAP_PROP_FPS)
khung_hinh = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
rong = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
cao = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))

while True:
    ok, frame = cap.read()
    if not ok:
        break
    # xử lý frame
    cv2.imshow("video", frame)
    if cv2.waitKey(1) & 0xFF == ord("q"):
        break

cap.release()
cv2.destroyAllWindows()
```

### Ghi video

```python
fourcc = cv2.VideoWriter_fourcc(*"mp4v")
out = cv2.VideoWriter("out.mp4", fourcc, fps, (rong, cao))
# out.write(frame)
out.release()
```

## 29.4 Face detection

```python
import cv2

cascade = cv2.CascadeClassifier(cv2.data.haarcascades + "haarcascade_frontalface_default.xml")

img = cv2.imread("nguoi.jpg")
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

faces = cascade.detectMultiScale(gray, scaleFactor=1.1, minNeighbors=5, minSize=(30, 30))
for x, y, w, h in faces:
    cv2.rectangle(img, (x, y), (x + w, y + h), (0, 255, 0), 2)

cv2.imwrite("faces.jpg", img)
```

Haar cascade nhẹ nhưng kém chính xác. Model DNN hoặc MediaPipe chính xác hơn.

## 29.5 Đọc/ghi metadata

```python
from PIL import Image
from PIL.ExifTags import TAGS

img = Image.open("anh.jpg")
exif = img.getexif()
for tag_id, value in exif.items():
    tag = TAGS.get(tag_id, tag_id)
    print(f"{tag}: {value}")
```

Xóa EXIF trước khi đăng ảnh công khai:

```python
from PIL import Image

img = Image.open("anh.jpg")
du_lieu = list(img.getdata())
img_sach = Image.new(img.mode, img.size)
img_sach.putdata(du_lieu)
img_sach.save("sach.jpg")
```

## 29.6 Audio với `pydub`

```bash
pip install pydub
# cần ffmpeg trên hệ thống
```

```python
from pydub import AudioSegment

audio = AudioSegment.from_mp3("nhac.mp3")
print(audio.duration_seconds, audio.channels, audio.frame_rate)

# Cắt từ 10s đến 30s
doan = audio[10_000:30_000]
doan.export("doan.mp3", format="mp3")

# Nối
tong = audio1 + audio2
tong.export("ghep.mp3", format="mp3")

# Tăng âm lượng
to_hon = audio + 6    # +6 dB
nho_hon = audio - 6
```

### Fade

```python
audio.fade_in(2000).fade_out(2000).export("fade.mp3")
```

## 29.7 Phân tích audio với librosa

```bash
pip install librosa
```

```python
import librosa
import numpy as np

y, sr = librosa.load("nhac.wav", sr=22050)
print(len(y) / sr, "giây")

tempo, beat_frames = librosa.beat.beat_track(y=y, sr=sr)
print(tempo)

# MFCC — đặc trưng cho ML
mfcc = librosa.feature.mfcc(y=y, sr=sr, n_mfcc=13)
print(mfcc.shape)

# Spectrogram
import matplotlib.pyplot as plt
S = librosa.feature.melspectrogram(y=y, sr=sr)
S_db = librosa.power_to_db(S)
plt.figure(figsize=(10, 4))
librosa.display.specshow(S_db, x_axis="time", y_axis="mel", sr=sr)
plt.colorbar()
plt.savefig("spectrogram.png")
```

## 29.8 Video với ffmpeg qua subprocess

```python
import subprocess

def chuyen_doi(src: str, dst: str, codec: str = "libx264", crf: int = 23) -> None:
    subprocess.run([
        "ffmpeg", "-y",
        "-i", src,
        "-c:v", codec, "-crf", str(crf),
        "-c:a", "aac", "-b:a", "128k",
        dst,
    ], check=True)

def cat_video(src: str, dst: str, bat_dau: str, thoi_luong: str) -> None:
    subprocess.run([
        "ffmpeg", "-y", "-ss", bat_dau, "-i", src,
        "-t", thoi_luong, "-c", "copy", dst,
    ], check=True)

def tao_thumbnail(src: str, dst: str, tai: str = "00:00:05") -> None:
    subprocess.run([
        "ffmpeg", "-y", "-ss", tai, "-i", src,
        "-frames:v", "1", "-q:v", "2", dst,
    ], check=True)
```

### Lấy thông tin

```python
import json
import subprocess

def thong_tin_video(path: str) -> dict:
    out = subprocess.run([
        "ffprobe", "-v", "quiet", "-print_format", "json",
        "-show_format", "-show_streams", path,
    ], capture_output=True, text=True, check=True)
    return json.loads(out.stdout)
```

## 29.9 Kiểm tra ảnh an toàn

Upload ảnh là vector tấn công phổ biến.

```python
from PIL import Image
import magic

ALLOWED = {"image/jpeg", "image/png", "image/webp"}
MAX_BYTES = 5 * 1024 * 1024
MAX_PIXELS = 25_000_000

def kiem_tra_anh(data: bytes) -> str:
    if len(data) > MAX_BYTES:
        raise ValueError("ảnh quá lớn")

    mime = magic.from_buffer(data, mime=True)
    if mime not in ALLOWED:
        raise ValueError(f"loại không cho phép: {mime}")

    Image.MAX_IMAGE_PIXELS = MAX_PIXELS
    try:
        img = Image.open(__import__("io").BytesIO(data))
        img.verify()
    except Exception as e:
        raise ValueError(f"ảnh hỏng: {e}")

    return mime
```

- Giới hạn kích thước byte (chống DoS).
- Giới hạn pixel (chống decompression bomb).
- Kiểm tra magic bytes, không tin đuôi file.
- Lưu file ngoài web root, đổi tên ngẫu nhiên.

## 29.10 Xử lý batch song song

```python
from concurrent.futures import ProcessPoolExecutor
from pathlib import Path
from PIL import Image

def xu_ly(path: Path) -> str:
    img = Image.open(path)
    img.thumbnail((800, 800))
    out = path.with_name(f"{path.stem}_thumb{path.suffix}")
    img.save(out, quality=85)
    return str(out)

paths = list(Path("anh").glob("*.jpg"))
with ProcessPoolExecutor() as ex:
    for ket_qua in ex.map(xu_ly, paths):
        print(ket_qua)
```

Pillow release GIL một phần, nhưng process pool vẫn nhanh hơn cho CPU-bound.

## 29.11 Bài tập

1. Viết script resize toàn bộ ảnh trong thư mục, giữ tỉ lệ.
2. Thêm watermark chéo góc cho ảnh.
3. Dùng OpenCV tìm contour của vật thể màu đỏ trong ảnh.
4. Cắt video 30 giây từ giữa file, không re-encode.
5. Tạo thumbnail từ video tại giây thứ 10.
6. Đọc waveform và vẽ spectrogram cho file WAV.
7. Viết validator upload ảnh với giới hạn byte/pixel/mime.
8. Batch xử lý 1000 ảnh bằng process pool, đo thời gian.

## 29.12 Ghi chú kỹ thuật

- Pillow đọc JPEG/PNG/WebP; OpenCV đọc thêm video.
- OpenCV dùng BGR, Pillow dùng RGB — nhớ chuyển đổi.
- `ffmpeg` mạnh nhất cho video; cài qua package manager.
- Video re-encode tốn CPU; `-c copy` chỉ đổi container, nhanh.
- `PIL.Image.MAX_IMAGE_PIXELS` bảo vệ chống decompression bomb.
- Ảnh lớn nên xử lý theo tile để tránh hết RAM.
- Audio stereo có 2 channel; mono có 1 — kiểm tra trước khi xử lý.
- OpenCV dùng nhiều luồng nội bộ; tắt bằng `cv2.setNumThreads(0)` khi kết hợp process pool.

Stashed.