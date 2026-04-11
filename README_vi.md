# 🚀 Cảm biến ảo cho phương tiện (Virtual Vehicle Sensor)
*Đọc bản tiếng Anh tại [English EN](README.md)*

### Hệ thống Machine Learning cho bài toán hồi quy độ dốc đường & phân loại khối lượng xe

**Tác giả:** Phạm Trâm Anh  
**Mentor:** Phạm Long Vân – Data Manager  
**Địa điểm:** TP. Hồ Chí Minh, Việt Nam – 2026  

---

# 🌍 Bối cảnh bài toán

## 🚘 Đo độ dốc đường & ước lượng khối lượng xe

Việc biết khối lượng xe và độ dốc mặt đường mang lại nhiều lợi ích cho các hệ thống hỗ trợ lái xe. Ví dụ, thông tin này có thể giúp tối ưu quản lý năng lượng hoặc lập kế hoạch lộ trình dựa trên các giới hạn tải trọng.

Tuy nhiên, việc lắp thêm cảm biến để đo trực tiếp các đại lượng này thường rất tốn kém. Trong khi đó, xe đã có sẵn nhiều tín hiệu như công suất động cơ, tốc độ,... có mối liên hệ chặt chẽ với các đại lượng chưa biết. Những tín hiệu này có thể được tận dụng để xây dựng một “cảm biến phần mềm” (software sensor) thay thế.

**Thách thức chính:** *Liệu có thể dự đoán chính xác khối lượng xe và độ dốc đường chỉ từ các tín hiệu có sẵn trên xe hay không?*

---

## 📊 Mô tả dữ liệu & cấu trúc hệ thống

Dataset gồm 11 tín hiệu xe, **không theo chuỗi thời gian** (đã bị xáo trộn thứ tự), vì vậy mô hình phải dự đoán trên từng mẫu độc lập.

### 1. Cấu trúc hệ thống
Hệ thống cảm biến ảo gồm 2 nhánh:
- Nhánh **Hồi quy (Regression)**: dự đoán độ dốc đường liên tục
- Nhánh **Phân loại (Classification)**: xác định khối lượng xe rời rạc

### 2. Các feature đầu vào (9 biến)
- `Epm_nEng_100ms`: tốc độ động cơ trung bình (rpm)  
- `VehV_v_100ms`: tốc độ xe (km/h)  
- `ActMod_trqInr_100ms`: mô-men động cơ nội suy (Nm)  
- `RngMod_trqCrSmin_100ms`: mô-men nhỏ nhất tại trục khuỷu (Nm)  
- `CoVeh_trqAcs_100ms`: mô-men phụ tải (Nm)  
- `Clth_st_100ms`: trạng thái ly hợp  
- `CoEng_st_100ms`: trạng thái động cơ  
- `Com_rTSC1VRVCURtdrTq_100ms`: giới hạn mô-men (%)  
- `Com_rTSC1VRRDTrqReq_100ms`: mô-men yêu cầu (%)  

---

### 3. Output
- **Vehicle Mass (Classification):** 38 tấn hoặc 49 tấn  
- **Road Slope (Regression):** độ dốc mặt đường liên tục (%)

---

### 4. Ứng dụng thực tế
- **Quản lý năng lượng & tiết kiệm nhiên liệu:** điều chỉnh mô-men, ga, hộp số theo tải trọng và độ dốc  
- **Lập kế hoạch lộ trình động:** tránh đường dốc lớn hoặc giới hạn tải trọng  
- **Giảm chi phí phần cứng:** thay thế cảm biến đo tải và đo độ nghiêng truyền thống  

---

# 📌 Tóm tắt hệ thống

Hệ thống gồm các bước:
- Phân tích dữ liệu (EDA)
- Tiền xử lý & feature engineering
- Xử lý outlier bằng scaling
- Huấn luyện mô hình hồi quy & phân loại

---

# 1️⃣ Phân tích dữ liệu (EDA)

## Thiếu dữ liệu
Dataset không có missing values → dữ liệu sạch.

## Feature không có phương sai
Loại bỏ 5 feature chỉ có 1 giá trị duy nhất.

## Tương quan
Phát hiện tương quan âm mạnh (-0.62) giữa tốc độ xe và một tín hiệu mô-men → tạo feature kết hợp.

## Outlier
Dữ liệu có phân bố lệch → sử dụng RobustScaler để giảm ảnh hưởng ngoại lai.

## Phân bố nhãn
Dùng stratify khi chia tập train/test để giữ cân bằng lớp.

---

# 2️⃣ Tiền xử lý & Feature Engineering

- Loại bỏ feature dư thừa
- Tạo feature kết hợp
- Chia tập train/test/dev
- Mã hóa nhãn phân loại
- Chuẩn hóa bằng RobustScaler

---

# 3️⃣ Mô hình & đánh giá

## Metric
- Classification: G-Mean = sqrt(Recall0 × Recall1)
- Regression: thang điểm theo mức sai số

---

## 1. Phân loại (Vehicle Mass)
Thử nghiệm Logistic Regression, Random Forest, Gradient Boosting, SVM

➡️ SVM (SVC) cho kết quả tốt nhất: ~0.9970

---

## 2. Hồi quy (Road Slope)
- Thử nhiều mô hình ensemble
- KNN cho kết quả tốt nhất
- Cải tiến bằng cách thêm feature dự đoán khối lượng xe
- KNN (k=1) đạt ~0.7694

---

# 4️⃣ Kết quả cuối

- Classification: 0.9953  
- Regression: 0.7543  
- Điểm tổng: 1.7496  

---

## Model artifacts
- SVC classifier (vehicle mass)
- KNN regressor (road slope)
- RobustScaler & LabelEncoder

---

# 🏁 Kết luận

Dự án xây dựng một hệ thống cảm biến ảo end-to-end, có thể thay thế cảm biến vật lý đắt tiền bằng mô hình machine learning. Hệ thống kết hợp SVM và KNN cùng kỹ thuật xử lý nhiễu giúp đạt hiệu năng cao và có khả năng triển khai thực tế.
