# 🚀 Cảm Biến Ảo Trên Xe (Virtual Vehicle Sensor)
*Đọc bản [Tiếng Anh 🇬🇧](README.md)*

### Hệ thống Học máy cho Bài toán Hồi quy Độ dốc & Phân loại Khối lượng Xe

**Tác giả:** Phạm Trâm Anh
**Cố vấn:** Phạm Long Vân – Data Manager  
**Địa điểm:** TP. Hồ Chí Minh, Việt Nam – 2026 

---

# 🌍 Bối Cảnh Thực Tế

## 🚘 Cảm biến đo độ dốc & Ước tính trọng tải xe

Việc biết được trọng lượng của xe và độ dốc của mặt đường mang lại rất nhiều lợi ích cho các hệ thống hỗ trợ lái xe. Ví dụ, thông tin này có thể được sử dụng để tối ưu hóa quản lý năng lượng hoặc lập kế hoạch lộ trình dựa trên các biển báo hạn chế trọng tải.

Tuy nhiên, việc lắp đặt thêm các cảm biến tĩnh chỉ để đo trọng lượng và độ dốc trực tiếp là rất tốn kém. Dù vậy, xe cộ hiện đại đã theo dõi sẵn các thuộc tính như công suất động cơ và tốc độ - những yếu tố phụ thuộc rất mạnh vào các đại lượng chưa biết này. Chúng có thể được sử dụng làm đầu vào cho một "cảm biến" phần mềm, với chi phí rẻ hơn rất nhiều so với giải pháp phần cứng.

**Thách thức cốt lõi**: *Bạn có thể dự đoán chính xác trọng lượng và độ dốc chỉ bằng các tín hiệu đã có sẵn trên xe không?*

---

## 📊 Mô Tả Dữ Liệu & Cấu Trúc Hệ Thống

Để giải quyết thách thức này, bộ dữ liệu đóng vai trò là nền tảng cốt lõi cho hệ thống cảm biến phần mềm. Nó bao gồm 11 tín hiệu xe riêng biệt và **không có trình tự thời gian** (thứ tự đã bị xáo trộn), nghĩa là hệ thống dự đoán phải hoạt động ổn định trên từng **khung hình (frame) độc lập**.

### 1. Cấu Trúc Hệ Thống
Kiến trúc của chúng tôi hoạt động như một **Cảm biến Ảo** động cơ kép, sử dụng phép đo vô tuyến (telemetry) tiêu chuẩn làm đầu vào và đưa trực tiếp vào:
- **Nhánh Hồi quy (Regression Engine)**: Dự báo độ dốc liên tục của đường.
- **Nhánh Phân loại (Classification Engine)**: Xác định ranh giới tải trọng khối lượng rời rạc.

### 2. Các Chỉ Số Đầu Vào (9 Biến Dự Đoán)
Đây là các biến tín hiệu tiêu chuẩn trên xe được các mô hình máy học sử dụng trực tiếp:
- `Epm_nEng_100ms`: Tốc độ trung bình (rpm) của một xi-lanh động cơ
- `VehV_v_100ms`: Tốc độ phương tiện (km/h)
- `ActMod_trqInr_100ms`: Mô-men xoắn bên trong động cơ (hiện tại tính ngược, Nm)
- `RngMod_trqCrSmin_100ms`: Mô-men xoắn tối thiểu tại trục khuỷu (Nm)
- `CoVeh_trqAcs_100ms`: Nhu cầu mô-men xoắn của các bộ phận phụ kiện (Nm)
- `Clth_st_100ms`: Trạng thái bộ ly hợp đã được xử lý chống dội (debounced) (-)
- `CoEng_st_100ms`: Trạng thái hoạt động của động cơ (định mức enum: 0 đến 5)
- `Com_rTSC1VRVCURtdrTq_100ms`: Giới hạn mô-men xoắn định trước (%)
- `Com_rTSC1VRRDTrqReq_100ms`: Nhu cầu mô-men xoắn được yêu cầu bởi bộ hãm retarder (%)

**Bản Xem Trước Dữ Liệu (df.head())**
| Epm_nEng_100ms | VehV_v_100ms | ActMod_trqInr_100ms | RngMod_trqCrSmin_100ms | CoVeh_trqAcs_100ms | Clth_st_100ms | CoEng_st_100ms | Com_rTSC1VRVCURtdrTq_100ms | Com_rTSC1VRRDTrqReq_100ms | RoadSlope_100ms | Vehicle_Mass |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1000.5 | 45.2 | 120.4 | -15.0 | 9.999747 | 0 | 3 | 0 | 0 | 0.5 | 49 |
| 850.5 | 24.5 | 50.1 | -15.0 | 9.999747 | 0 | 3 | 0 | 0 | 1.2 | 38 |
| 1200.0 | 60.0 | 450.2 | -15.0 | 9.999747 | 0 | 3 | 0 | 0 | -0.2 | 49 |

### 3. Đầu Ra Mục Tiêu Của Bài Toán
- **Khối lượng Xe (Phân loại)**: Bị giới hạn và rời rạc hóa thành 2 nhãn chính xác: `38 t` (tấn) hoặc `49 t` (tấn).
- **Độ dốc Đường (Hồi quy)**: Biểu diễn dưới dạng phần trăm độ dốc thực tế tính theo góc đo ADASIS, sử dụng hàm trích công thức `tan(slope) = RoadSlope_100ms / 100`.

### 4. Luồng Ứng Dụng Thực Tế
Một khi cảm biến ảo dự đoán chính xác **Độ dốc Đường** và **Khối lượng hệ thống**, những thông tin đo đạc này sẽ giúp tăng cường một cách mạnh mẽ vào Hệ thống Hỗ trợ Lái xe Tiên tiến (ADAS) và các hạm đội định tuyến bảo quản nhiên liệu:
- **Quản lý Năng lượng Thông minh & Tiết kiệm nhiên liệu**: Động thái tinh chỉnh mô-men xoắn tự động, tính toán hành vi đạp ga tối ưu nhất hoặc tối ưu tỷ số truyền số thích hợp dựa trên tải trọng xe và độ nghiêng mặt tiền rẽ dốc.
- **Lập kế hoạch Lộ Trình Động**: Bơm số liệu thực vào hệ thống định vị để tái lập bản đồ tuyến nhanh, tránh né các con dốc bất thường hay các giới hạn thiết kế mặt dây văng cầu nhẹ (bỏ tải 49 tấn).
- **Giải Pháp Kinh Tế Triệt Bỏ Thiết Bị Rườm Rà**: Xóa rèm bài toàn bảo trì, cài cắm độ cao hay thiết lập lại số đo vật lý vốn cực tốn chi phí của giới điện tử cảm biến thô.

---

## 📌 Tóm Tắt Quy Trình Pipeline Hệ Thống

Hệ thống cung ứng giải pháp Data nhúng này đảm đương phần thiết kế Pipeline phân tích để biến tín hiệu vô nghĩa thành nhãn dữ liệu chuẩn xác. Dòng công việc diễn ra chuyên biệt như sau:

- Tiến hành **Khám phá Dữ liệu (EDA)** toàn diện để giải mã các tương quan cơ khí xe hạng nặng.  
- Xây dựng một quy trình **Chuẩn bị Dữ liệu & Xây dựng Đặc trưng (Feature Engineering)** tinh gọn để đảm bảo luồng vào cân bằng.  
- Áp dụng các kỹ thuật **Phòng vệ Dữ liệu Ngoại lai (Outlier Defense)** mạnh mẽ để triệt tiêu nhiễu môi trường động cơ qua các hàm co giãn toán học.   
- Khảo nghiệm và **Mô hình Hóa (Học máy phân loại & Hồi quy chiều sâu)** để chốt sổ một cấu trúc cảm biến hoàn chỉnh nhất, sẵn sàng vươn tầm kiểm duyệt gắt gao.

---

# 1️⃣ Phân Tích & Khám Phá Dữ Liệu (EDA)

Quy trình EDA đóng vai trò cực điểm quyết yếu khi tiếp xúc bộ dữ liệu điện toán xe - nơi thang đo kỹ thuật biến đổi gắt gao và hiện tượng điểm mù cảm biến thường trực.

## 🔹 Đánh Giá Dữ Liệu Trống (Missing Values Assessment)
- Xác thực và chứng nhận toàn khối dữ liệu an toàn **không tồn tại ô trống Missing Values**. Sự nhất quán này mang đến sức mạnh phân tích thuần khiết, chặn đứng nguy cơ áp dập thiên kiến đắp bù dữ liệu vô căn cứ (imputation).

```python
# Verify payload continuity across all vehicle frames
missing_stats = df.isnull().sum()
print(f"Total missing values detected: {missing_stats.sum()}")
```
```text
Total missing values detected: 0
```

## 🔹 Phân Tích Tương Quan (Feature Correlation)
- **Cắt Giảm Chiều Dữ Liệu**: Loại bỏ hoàn toàn an toàn 5 cột điểm nút chứa hằng số duy nhất. Đặc trưng kiểu này sở hữu phương sai bằng 0 và không có giá trị phân tích máy học.

```python
# Identify and drop columns with zero variance (single unique value)
useless_cols = [col for col in df.columns if df[col].nunique() == 1]
print(f"Features with single unique value:\\n{useless_cols}")

df.drop(columns=useless_cols, inplace=True)
print(f"Shape after dropping zero-variance features: {df.shape}")
```
```text
Features with single unique value:
['CoVeh_trqAcs_100ms', 'Clth_st_100ms', 'CoEng_st_100ms', 'Com_rTSC1VRVCURtdrTq_100ms', 'Com_rTSC1VRRDTrqReq_100ms']
Shape after dropping zero-variance features: (8496, 6)
```
- **Hệ Thống Tương Quan Sâu**: Chứng minh và phát hiện sự tương quan phủ nhận **(-0.62)** mật thiết giữa Vận tốc (`VehV`) và Moment Nhỏ (`RngMod`). Qua đó ra quyết định tổng hợp 2 dòng chỉ số này vào lại một cột thông số duy nhất là `Combined_VehV_RngMod`.

![Correlation Heatmap](images/plot_5.png)

## 🔹 Xử Lý Dữ Liệu Biên / Ngoại Lai (Outlier Detection)
- Phát hiện mức độ lệch cực đoan (data skewing) đè nặng lên các tham số đo từ xa cơ khí động cơ (Điển hình như giá trị trung vị tại cột `RngMod_trqCrSmin_100ms` dẫm đè lên phân vị `Q1`).
- Thông qua việc sử dụng **RobustScaler** làm ranh giới áp suất đàn hồi, giải pháp này dập tan tính nhảy cảm ứng trước các số liệu dị biệt rủi ro biên. Từ đó tăng kháng cự của toàn thuật toán.

![Outlier Detection](images/plot_2.png)

## 🔹 Phân Phối Nhãn Lớp Nhóm (Class Label Distribution)
- Để duy trì quyền biểu thị chính xác kết quả mô hình vào thực tế chia rẽ tệp Data, kỹ thuật bắt buộc sử dụng **Mạch Phân Tầng (`stratify=y`)** khi tách cụm Batch. Tức là giữ đúng tỷ lệ Khối Lượng gốc giữa các tệp kiểm định.

![Class Label Distribution](images/plot_4.png)

---

# 2️⃣ Quá Trình Nhào Nặn & Biến Khổi Dữ Liệu

Từ các chẩn đoán từ EDA, giải pháp nén quy trình tự động lại qua 5 bước sau khi đi qua chốt AI:

1. **Gạt Bỏ Đặc Trưng Thừa:** Cắn rớt các dòng dữ liệu 0 phương sai để hạ khối lượng thông tin tính toán.
2. **Liến Kết Mạng Lưới:** Chồng nhét hợp lý các hệ đặc trưng mang tính tương quan mạnh đã đề cập.
3. **Phân Tách Lõi Data:** Cấp phép tạo ranh chia huấn luyện & kiểm định tảng thực.
4. **Mã Hóa Định Trạng Phân Lớp Máy:** Chuyển vần mã ký tự định danh nhãn phân loại về mã hóa số thuần nhằm thỏa đáp biên học mất mát số nguyên.
5. **Tiêu Chuẩn Scale Cân Khung Nền:** Chạy đệm qua `RobustScaler` thiết lập khung trượt an toàn.

---

# 3️⃣ Mô Hình Hóa & Nghiệm Thu Cuối

## 🧮 Logic Tính Điểm Chuyên Dụng (Custom Scoring)
Hệ quy chuẩn nghiệm thu mô hình đập bỏ cấu trúc tính phần trăm rỗng tuếch, và vác theo độ tinh tế biên ranh tính nhẩm:
- **Số Khai Trừ Phân Cấp G-mean**: Trả về độ lớn Toán Học Tối Ưu bằng căn bậc 2 tỷ lệ `Recall` các tệp để chống sai lệch hạng lớp $\sqrt{Recall_0 \times Recall_1}$.
- **Điểm Thêm Bớt Hồi Quy Lõi**: Giữ nguyên nguyên lý trượt ngầm theo các vòng bao cấp điểm tùy sai số bù trừ chênh lệch (sliding absolute penalty errors).

## 🎯 1. Chiếc Động Cơ Máy Phân Loại Khối Lượng (Vehicle Mass)
Thực đo đánh giá xuyên 4 cột móng cấu trúc hạng nặng (*Logistic Regression, Random Forest, Gradient Boosting, SVM*), thuật toán kinh tế nền **SVM (Khung Cấu Trúc Khối Nền Support Vector Machine - SVC)** vượt biên vô đối nhờ khả năng tối ưu khung G-mean đật sát mốc **0.9970**.

![Classifier Comparison](images/plot_6.png)

## 📈 2. Chiếc Động Cơ Hồi Quy Đo Độ Dốc (Road Slope)
Với tham số dò lưới (grid) khổng lồ đưa vào tìm kiếm biên viền, mọi mô hình quần thể sâu như Gradient Tree đều lệch trọng tâm dung sai chuẩn.
- Nhưng sự quay lại cội nguồn của định luật thuật toán **KNeighborsRegressor (KNN)** đã xé lưới bứt phá với khoảng cách không thể ngờ.
- **Tuyệt Chiêu Tối Ưu Bằng Phân Luồng Đa Nhiệm (Multi-tasking)**: Biến hóa ma trận Hồi quy bằng thao tác đắp thêm cả nhãn đầu ra Dự tính Trọng lượng Xe `Pred_Vehicle_Mass` (Được cung ra từ cỗ máy SVM trên) trở về làm đầu vào. 
- **Kết Trái**: Khung cấu trúc lai tạp *KNN siêu độ với K=1 & lưới gộp=10* kéo sập viền sai số, ôm kỷ lục tổng điểm cao hơn rất nhiều **0.7694**.

![KNN Heatmap](images/plot_7.png)

---

# 4️⃣ Đánh Giá Hệ Thống Chốt Ổ & Các Hiện Vật (Artifacts) Xuất Rời

## Thử Lửa Trực Tiếp Trên Tập Kiểm Tra Bị Mã Hóa Kín (Test Sets)
Song mã mô hình tiêu chuẩn (`SVC` & `KNN Học lai Đa nhiệm MultiTask`) chính thức lao về phía vùng dữ liệu nhiễu đời thực. 
- **Chuẩn điểm Cân xe Phân Loại**: Nhận về tuyệt hảo `0.9953`
- **Chuẩn điểm Đo dốc Hồi Quy**: Nhận về mức khá mạnh `0.7543` 
- **Hiệu Năng Tổng Thế Tổ Hợp Hệ Thống**: Dừng tại vị thế đỉnh cao `1.7496`

## ⚙️ Thiết Yếu Phẩm Triển Khai Thực Tế Đi Kèm Đóng Gói
Hệ sinh thái này đã xả ra các gói nhúng để chờ kết xuất trực tiếp vào bo mạch chủ lập mô hình vô hình gốc trên các hệ thiết bị vi tế của các hãng ô tô hiện nay:
- Ma Trận Phân loại Cân Điểm Trọng Yếu: `svc_vehicle_mass_classifier.joblib`
- Ma Trận Thu Số Đo dốc Biên Gốc: `knn_road_slope_regressor.joblib`
- Xơ Lọc Bảo Hộ Hệ Môi Trường: `robust_scaler.joblib` / `label_encoder.joblib` 

---

# 🏁 Lời Tựu Chung
Luồng khai tác của hệ thiết bị hệ thống phân tách Cảm Biến Ô Tô này không mang mục đích đo xem số đúng hay là số sai; Mà hoàn toàn đúc kết ra một bộ Lõi Truyền Phát Logic End-To-End (End-to-End telemetry). Cú lồng ghép kinh điển của cơ chế Phân tích triệt mầm Nhiễu Hại Ngoại Biên với cỗ máy Hát Xướng Mạng Học Kết Nhóm Đa Nhiệm Song Bào (Dual-model Engine SVC + Augmented KNN), nền móng tri thức này tự tạo nên định vị của mình là một Thể Thức Cảm Biến Ảo Không Gian Độc Lập hoàn chỉnh — Sẵn sàng thay trọn quyền của các mạng lưới đinh cân phần cứng, mạch thiết bị đo độ nghiêng rườm rà tại môi trường xa lộ đời sống công xưởng.
