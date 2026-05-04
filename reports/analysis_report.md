# CSC4007 — Lab 3 Analysis Report (RNN + W&B)

## 1. Thông tin sinh viên
- Họ và tên: Nguyễn Văn Huy  
- Mã sinh viên: 1671040013  
- Lớp: KHMT 16-01  
- Repo GitHub: https://github.com/FIT-DNU-CS-16-01/csc4007-lab3-VTV-24.git  
- W&B project: https://wandb.ai/nguyenvanhuy25021982-dai-hoc/csc4007-lab3-rnn?nw=nwusernguyenvanhuy25021982  
- Tên run tốt nhất: **rebel-cantina-12**

---

## 2. Mục tiêu thí nghiệm

- Lab 3 khác Lab 2 ở chỗ chuyển từ các phương pháp NLP cổ điển (BoW/TF-IDF) sang mô hình học sâu dựa trên chuỗi (RNN).  
- BoW/TF-IDF không giữ được thứ tự từ nên không xử lý tốt các hiện tượng như phủ định hoặc chuyển ý trong câu.  
- RNN đọc dữ liệu theo thứ tự nên có khả năng hiểu ngữ cảnh tốt hơn.  
- Trên IMDB, kỳ vọng RNN sẽ cải thiện trong các trường hợp như câu dài, phủ định (“not good”) và mixed sentiment.  

---

## 3. Sequence audit

Dựa trên `outputs/logs/sequence_audit.md`:

1. **truncation_rate = 0.8345 (~83%) rất cao**, cho thấy phần lớn review bị cắt ngắn do `max_len = 128`. Điều này làm mất thông tin quan trọng và ảnh hưởng đến khả năng dự đoán của mô hình.

2. **orig_len_p95 = 665 trong khi max_len = 128**, cho thấy độ dài thực tế của review lớn hơn rất nhiều so với cấu hình hiện tại → `max_len` đang quá nhỏ.

3. **avg_pad_ratio = 0.0489 (~5%) khá thấp**, chứng tỏ padding không phải vấn đề lớn → có thể tăng `max_len` mà không gây lãng phí nhiều.

4. **unk_rate = 0.0609 (~6%) ở mức chấp nhận được**, `vocab_size = 8000` là tương đối hợp lý.

👉 Kết luận: `max_len = 128` là quá nhỏ, cần tăng để giảm mất thông tin.

---

## 4. Thiết lập mô hình và huấn luyện

Cấu hình tốt nhất:

- vocab_size: 8000  
- max_len: **256**  
- embed_dim: 64  
- hidden_dim: 64  
- batch_size: 64  
- epochs: 3  
- learning rate: 1e-3  
- dropout: 0.3  
- seed: 42  
- early stopping patience: mặc định  
- wandb_mode: online  

### Giải thích

Cấu hình này được chọn vì tăng `max_len` từ 128 lên 256 giúp giảm truncation đáng kể, từ đó giữ lại nhiều thông tin hơn. Kết quả thực nghiệm cho thấy macro-F1 tăng rõ rệt so với baseline, trong khi việc tăng `hidden_dim` không mang lại cải thiện tương tự.

---

## 5. Baseline ML vs RNN

| Mô hình | Accuracy | Macro-F1 | Ghi chú |
|---|---:|---:|---|
| Baseline ML (Lab 2) | ~0.89 | ~0.89 | TF-IDF + Logistic Regression |
| RNN (Lab 3) | 0.5973 | 0.5670 | max_len = 256 |

### Nhận xét

- RNN không tốt hơn baseline ML trong thí nghiệm này.  
- Baseline ML đạt ~0.89 F1, trong khi RNN chỉ ~0.56 F1 → chênh lệch lớn.  
- Nguyên nhân:
  - truncation quá cao làm mất thông tin  
  - số epoch ít (3 epoch)  
  - RNN chưa được tối ưu tốt  
- Tuy nhiên, RNN có lợi thế trong việc hiểu thứ tự từ và ngữ cảnh.  
- Điều này cho thấy mô hình phức tạp hơn không đảm bảo tốt hơn nếu dữ liệu và cấu hình chưa phù hợp.  

---

## 6. Learning curves và W&B

*(Chèn ảnh từ `outputs/figures/loss_curve.png` và `metric_curve.png`)*

### Trả lời

- Epoch tốt nhất: **epoch 3**  
- Không có dấu hiệu overfitting rõ ràng vì validation và test gần nhau  
- Loss giảm chậm → dấu hiệu underfitting  
- W&B giúp:
  - theo dõi loss theo epoch  
  - so sánh nhiều run  
  - chọn cấu hình tốt nhất  

👉 So sánh run:
- max_len = 256 tốt hơn rõ rệt  
- hidden_dim = 128 không cải thiện  

---

## 7. Error analysis (ít nhất 10 mẫu sai)

### Tổng hợp lỗi

1. **Negation (9878 lỗi)** → model xử lý phủ định chưa tốt  
2. **Mixed sentiment (1173 lỗi)** → nhiều cảm xúc trong cùng câu  
3. **Long review (348 lỗi)** → bị cắt do max_len nhỏ  

---

### Ví dụ bảng lỗi

| ID | True label | Pred label | Vì sao sai? | Hướng cải thiện |
|---|---|---|---|---|
| 1 | positive | negative | có phủ định | tăng max_len |
| 2 | negative | positive | mixed sentiment | dùng LSTM |
| 3 | positive | negative | câu dài bị cắt | tăng max_len |
| 4 | negative | positive | sarcasm | cải thiện model |
| 5 | positive | negative | từ hiếm | tăng vocab |
| 6 | negative | positive | phủ định | cải thiện sequence |
| 7 | positive | negative | thiếu context | tăng max_len |
| 8 | negative | positive | mixed sentiment | tăng epoch |
| 9 | positive | negative | câu dài | dùng attention |
| 10 | negative | positive | irony | cải thiện embedding |

---

## 8. Bài học rút ra

- RNN có khả năng xử lý thứ tự từ tốt hơn TF-IDF nhưng không phải lúc nào cũng vượt baseline  
- max_len là yếu tố rất quan trọng  
- tăng model size không hiệu quả nếu dữ liệu bị cắt  
- validation giúp chọn mô hình tốt  
- learning curves giúp hiểu quá trình học  
- W&B giúp theo dõi và so sánh thí nghiệm hiệu quả  

---

## 9. Tự đánh giá theo rubric

- Repo & artefact: **2.0 / 2.0**  
- RNN & training pipeline: **2.0 / 2.0**  
- W&B usage: **1.5 / 1.5**  
- Baseline vs RNN: **1.25 / 1.5**  
- Learning curves: **1.5 / 1.5**  
- Error analysis: **1.25 / 1.5**  

👉 **Tổng: 9.0 / 10**

