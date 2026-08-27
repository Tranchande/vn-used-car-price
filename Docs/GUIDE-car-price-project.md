# GUIDE — Dự đoán giá xe ô tô cũ Việt Nam

Repo name gợi ý: `vn-used-car-price`

Mục tiêu cuối: một API nhận thông số xe → trả về giá dự đoán, có Docker chạy được bằng 1 lệnh, README có bảng kết quả.

Đọc từng phase, làm xong tick vào checkbox. Đừng nhảy cóc.

---

## PHASE 0 — Setup (30 phút)

- [ ] Tạo repo trên GitHub, tick "Add .gitignore" → chọn Python
- [ ] Clone về máy, mở bằng VS Code
- [ ] Tạo virtual environment:

```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate
```

- [ ] Cài package cần thiết:

```bash
pip install requests beautifulsoup4 pandas numpy scikit-learn lightgbm matplotlib seaborn jupyter fastapi uvicorn joblib
pip freeze > requirements.txt
```

- [ ] Tạo cấu trúc thư mục:

```
vn-used-car-price/
├── README.md
├── requirements.txt
├── Dockerfile
├── .gitignore
├── data/
│   ├── raw/            # HTML thô + csv thô
│   └── processed/      # dữ liệu đã làm sạch
├── notebooks/
├── src/
│   ├── scraper.py
│   ├── preprocess.py
│   └── train.py
├── app/
│   └── main.py
└── models/
```

- [ ] Thêm vào `.gitignore`:

```
venv/
data/raw/
*.pkl
__pycache__/
.ipynb_checkpoints/
```

> **Lưu ý:** `data/raw/` không push lên GitHub (nặng + có thể vi phạm ToS của trang web). Nhưng `data/processed/` thì NÊN push một file mẫu vài trăm dòng để người khác chạy thử được.

- [ ] Commit đầu tiên: `git commit -m "chore: project structure"`

---

## PHASE 1 — Thu thập dữ liệu (ngày 1–3)

### Nguyên tắc

1. **Tải HTML thô xuống đĩa trước, parse sau.** Nếu logic parse sai, bạn sửa code và chạy lại trên file đã tải, không phải crawl lại từ đầu.
2. **`time.sleep(1)` giữa mỗi request.** Crawl quá nhanh sẽ bị chặn IP.
3. **Mục tiêu 3.000–5.000 tin là đủ.** Đừng tham.

### Bước 1.1 — Tải trang danh sách

Vào bonbanh.com hoặc xe.chotot.com, mở DevTools (F12) → tab Network, xem URL phân trang có dạng gì. Thường là `?page=2`, `/page-2`, hoặc tương tự.

```python
# src/scraper.py
import requests, time, os
from pathlib import Path

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
}

def download_page(url: str, save_path: Path) -> bool:
    if save_path.exists():          # đã tải rồi thì bỏ qua
        return True
    try:
        r = requests.get(url, headers=HEADERS, timeout=15)
        r.raise_for_status()
        save_path.parent.mkdir(parents=True, exist_ok=True)
        save_path.write_text(r.text, encoding="utf-8")
        time.sleep(1)
        return True
    except Exception as e:
        print(f"[FAIL] {url} — {e}")
        return False
```

- [ ] Tải khoảng 100–200 trang danh sách, lưu vào `data/raw/pages/`
- [ ] Kiểm tra: mở thử 1 file HTML bằng trình duyệt, xem có đúng nội dung không (đôi khi bị chặn và trả về trang captcha)

### Bước 1.2 — Parse ra CSV

Dùng DevTools → Inspect một tin đăng, xem class/id của các trường: tên xe, giá, năm, số km, hộp số, nhiên liệu, địa điểm.

```python
from bs4 import BeautifulSoup
import pandas as pd

def parse_listing(html: str) -> list[dict]:
    soup = BeautifulSoup(html, "html.parser")
    rows = []
    for item in soup.select("SELECTOR_CUA_1_TIN"):   # bạn tự tìm selector
        rows.append({
            "title":    _text(item, "SELECTOR_TITLE"),
            "price_raw": _text(item, "SELECTOR_PRICE"),
            "year_raw":  _text(item, "SELECTOR_YEAR"),
            "km_raw":    _text(item, "SELECTOR_KM"),
            "location":  _text(item, "SELECTOR_LOC"),
        })
    return rows

def _text(node, selector):
    el = node.select_one(selector)
    return el.get_text(strip=True) if el else None
```

- [ ] Lưu tất cả ra `data/raw/listings_raw.csv`
- [ ] In `df.shape` và `df.head(20)` — kiểm tra bằng mắt xem có bị lệch cột không
- [ ] Commit: `feat: scraper + raw dataset`

**Kẹt ở đâu thì quay lại hỏi mình, đặc biệt nếu trang dùng JavaScript render (lúc đó BeautifulSoup không thấy dữ liệu, phải đổi cách).**

---

## PHASE 2 — Làm sạch & EDA (ngày 4–6)

Đây là phần **quan trọng nhất** của cả project. Đừng làm qua loa để chạy sang phần model.

Tạo `notebooks/01_eda.ipynb`.

### Bước 2.1 — Chuẩn hóa giá

Giá trên tin đăng có đủ kiểu: `"1 tỷ 250 triệu"`, `"850 triệu"`, `"1.25 tỷ"`, `"Thương lượng"`, `"Liên hệ"`.

- [ ] Viết hàm `parse_price(s) -> float | None`, trả về đơn vị **triệu đồng**
- [ ] Bỏ hết dòng không parse được giá (Thương lượng, Liên hệ) — ghi lại **bao nhiêu dòng bị bỏ**, con số này sẽ đưa vào README
- [ ] Bỏ outlier vô lý: giá < 50 triệu hoặc > 20.000 triệu

### Bước 2.2 — Chuẩn hóa các cột khác

- [ ] `year`: ép về int, bỏ dòng ngoài khoảng 1990–2026
- [ ] `km`: bỏ dấu chấm/phẩy, ép về int. Xe đời 2023 mà 500.000 km là tin rác → bỏ
- [ ] Tạo cột `age = 2026 - year` (thường mạnh hơn `year` khi làm feature)
- [ ] Tách `brand` và `model` từ `title`. Chuẩn hóa viết thường, bỏ dấu cách thừa
- [ ] Gom brand hiếm (< 20 tin) vào nhóm `"other"` — tránh tạo quá nhiều cột khi one-hot

### Bước 2.3 — EDA (vẽ ít nhất 5 biểu đồ)

- [ ] Histogram của `price` → gần như chắc chắn lệch phải
- [ ] Histogram của `log(price)` → so sánh, quyết định dùng target nào
- [ ] Boxplot `price` theo `brand` (top 10 brand)
- [ ] Scatter `age` vs `price`
- [ ] Scatter `km` vs `price`
- [ ] Heatmap tương quan giữa các biến số

**Với mỗi biểu đồ, viết 1–2 câu nhận xét ngay dưới cell.** Notebook chỉ có hình mà không có chữ thì vô giá trị.

- [ ] Lưu `data/processed/clean.csv`
- [ ] Commit: `feat: data cleaning + EDA`

> **Quyết định cần ghi lại:** bạn bỏ bao nhiêu % dữ liệu và vì sao. Từ 5.000 tin xuống còn 3.200 tin sạch là chuyện bình thường — nhưng phải giải thích được.

---

## PHASE 3 — Mô hình (ngày 7–9)

Tạo `notebooks/02_modeling.ipynb`.

### Bước 3.1 — Chia dữ liệu TRƯỚC KHI làm bất cứ gì

```python
from sklearn.model_selection import train_test_split

X = df.drop(columns=["price"])
y = np.log1p(df["price"])        # nếu quyết định dùng log target

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)
```

> ⚠️ **Data leakage — lỗi chết người.** Mọi thứ "học" từ dữ liệu (scaler, encoder, giá trị điền missing) chỉ được `fit` trên `X_train`, rồi `transform` lên `X_test`. Fit trên toàn bộ dữ liệu là sai. Dùng `Pipeline` của sklearn để tránh tự bắn vào chân mình.

### Bước 3.2 — Baseline (bắt buộc)

```python
# Baseline: dự đoán bằng giá trung vị theo brand
```

- [ ] Tính MAE và MAPE của baseline. **Đây là con số mọi model sau phải vượt qua.**

Không có baseline thì "MAE 95 triệu" chẳng nói lên điều gì.

### Bước 3.3 — Các model

Chạy lần lượt, ghi kết quả vào một bảng:

- [ ] Linear Regression
- [ ] Random Forest
- [ ] LightGBM (hoặc XGBoost)

Với mỗi model:
- [ ] Dùng `cross_val_score` với `cv=5`, không chỉ đánh giá một lần
- [ ] Ghi lại MAE, RMSE, MAPE, R²
- [ ] Nếu dùng log target, nhớ `np.expm1()` trước khi tính MAE để số liệu có nghĩa (đơn vị triệu đồng)

### Bước 3.4 — Đào sâu model tốt nhất

- [ ] Tune hyperparameter bằng `RandomizedSearchCV` (nhanh hơn GridSearch)
- [ ] Vẽ **feature importance** — xem cái gì quyết định giá xe. Đây là hình đẹp nhất để đưa vào README
- [ ] Vẽ **residual plot**: dự đoán vs thực tế. Xem model sai nhiều ở phân khúc nào (thường là xe siêu sang, ít dữ liệu)
- [ ] Lưu model: `joblib.dump(pipeline, "models/model.pkl")`

- [ ] Commit: `feat: model training + evaluation`

---

## PHASE 4 — API & Docker (ngày 10–12)

### Bước 4.1 — FastAPI

```python
# app/main.py
from fastapi import FastAPI
from pydantic import BaseModel
import joblib, pandas as pd, numpy as np

app = FastAPI(title="VN Used Car Price Predictor")
model = joblib.load("models/model.pkl")

class CarInput(BaseModel):
    brand: str
    model_name: str
    year: int
    km: int
    transmission: str
    fuel: str

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/predict")
def predict(car: CarInput):
    df = pd.DataFrame([car.model_dump()])
    df["age"] = 2026 - df["year"]
    pred_log = model.predict(df)[0]
    price = float(np.expm1(pred_log))
    return {
        "predicted_price_million_vnd": round(price, 1),
        "range": [round(price * 0.88, 1), round(price * 1.12, 1)]
    }
```

- [ ] Chạy `uvicorn app.main:app --reload`
- [ ] Mở `http://127.0.0.1:8000/docs` → test thử vài xe bạn biết giá thật ngoài đời
- [ ] Kiểm tra kết quả có hợp lý không. Nếu model đoán Vios 2020 giá 2 tỷ thì có gì đó sai

### Bước 4.2 — Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- [ ] Build: `docker build -t car-price .`
- [ ] Run: `docker run -p 8000:8000 car-price`
- [ ] Xác nhận `/docs` chạy được từ container
- [ ] Commit: `feat: FastAPI service + Docker`

---

## PHASE 5 — README (ngày 13–14)

**Dành hẳn nửa ngày cho phần này.** README quyết định 80% ấn tượng.

Khung bắt buộc:

```markdown
# Vietnamese Used Car Price Prediction

Dự đoán giá xe ô tô cũ tại Việt Nam từ 3.200 tin đăng thu thập được.
MAPE 11.4% — tốt hơn baseline (giá trung vị theo hãng) 34%.

[ẢNH: feature importance hoặc residual plot]

## Problem
[2–3 câu: tại sao bài toán này khó — giá xe cũ phụ thuộc nhiều yếu tố phi tuyến]

## Data
- Nguồn: [trang web], thu thập [tháng/năm]
- 5.000 tin thô → 3.200 tin sau khi làm sạch (bỏ 36%: giá "thương lượng", km bất thường, năm thiếu)
- Features: brand, model, age, km, transmission, fuel, location

## Approach
[Mô tả pipeline: scrape → clean → feature engineering → model]
Target dùng log(price) do phân phối lệch phải.

## Results

| Model             | MAE (triệu) | MAPE  | R²   |
|-------------------|-------------|-------|------|
| Baseline (median) | 187         | 31.2% | 0.41 |
| Linear Regression | 124         | 19.8% | 0.68 |
| Random Forest     | 89          | 13.1% | 0.83 |
| **LightGBM**      | **76**      | **11.4%** | **0.87** |

[1 đoạn nhận xét: cái gì quan trọng nhất theo feature importance]

## Limitations
- Dữ liệu chỉ từ 1 trang, thiên về khu vực TP.HCM
- Không có thông tin tình trạng xe (tai nạn, số đời chủ) — yếu tố ảnh hưởng lớn tới giá thực tế
- Model kém chính xác ở phân khúc xe > 3 tỷ do ít mẫu

## Run locally
[lệnh docker]

## Structure
[cây thư mục]
```

- [ ] Viết đủ các mục trên
- [ ] Chèn ít nhất 2 hình (lưu trong `assets/`, nhúng bằng markdown)
- [ ] Thêm topics cho repo trên GitHub: `machine-learning`, `python`, `fastapi`, `web-scraping`
- [ ] Viết description ngắn cho repo (ô bên phải trang repo)
- [ ] Pin repo lên profile

---

## Checklist cuối — trước khi coi là xong

- [ ] Clone repo về thư mục mới, làm theo README, chạy được không?
- [ ] Không có file `.pkl` nặng hay data thô bị push nhầm (`git count-objects -vH`)
- [ ] Không có API key / thông tin cá nhân trong code
- [ ] Notebook đã chạy hết từ trên xuống, không có cell lỗi
- [ ] Commit history rải đều, message rõ ràng (không phải 1 commit "update" duy nhất)
- [ ] README không còn chữ nào trong ngoặc vuông

---

## Ghi chú

Nếu kẹt ở bước nào — đặc biệt là Phase 1 (site dùng JS render) hoặc Phase 2 (quyết định bỏ dữ liệu nào) — quay lại hỏi. Hai chỗ đó làm sai thì hỏng cả project phía sau.
