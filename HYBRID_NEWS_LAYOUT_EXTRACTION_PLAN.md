# Giải pháp hybrid (thuật toán + AI) cho trích xuất nội dung báo in từ PDF dàn trang

## 1) Mục tiêu hệ thống
- Trích xuất đầy đủ từng bài báo từ PDF dàn trang báo khổ 42 x 58.5 cm (vector + ảnh), không thiếu/không thừa/không sửa nội dung.
- Nhận diện đúng cấu trúc bài: **Tiêu đề → Tác giả → Chú thích ảnh → Lead (Sapo) → Nội dung**.
- Đọc phần Nội dung đúng thứ tự cột từ trái sang phải trong phạm vi từng bài.
- Loại bỏ Header/Footer và các thành phần rác.
- Ghép chính xác bài nhiều kỳ/tiếp trang thông qua chỉ dấu “XEM TRANG …” và “Tiếp theo trang …”.
- SLA tốc độ: xử lý 8 trang ≤ 60 giây.

## 2) Kiến trúc tổng thể (Hybrid)

### 2.1. Tại sao hybrid?
- **Thuật toán hình học + typography** xử lý nhanh, ổn định với PDF vector (đọc tọa độ chính xác), giúp đạt tốc độ.
- **AI vision/NLP** xử lý các trường hợp khó: layout bất quy tắc, nhiễu, mơ hồ ranh giới, nhận diện semantic block.
- Kết hợp giúp tối ưu cả **độ chính xác** lẫn **chi phí/latency**.

### 2.2. Pipeline 3 tầng
1. **Tầng Parse định vị (deterministic, nhanh):**
   - Parse PDF object model: text spans, font size/style, bbox, vector lines/rectangles, image boxes.
   - Tạo graph bố cục trang (blocks, separators, adjacency).
2. **Tầng Nhận diện ngữ nghĩa (AI + rule):**
   - Classify block thành: title/byline/caption/lead/body/noise.
   - Segment article region + assign block vào article cluster.
3. **Tầng Hợp nhất & QA (deterministic + AI verify):**
   - Sắp thứ tự đọc, ghép bài tiếp trang, loại trùng/lỗi.
   - Chấm điểm chất lượng, cờ kiểm duyệt khi confidence thấp.

## 3) Mô hình dữ liệu chuẩn (Canonical Document Model)

### 3.1. Page primitives
- `TextSpan`: text, bbox, font_name, font_size, weight, color, page_id, reading_dir.
- `VectorLine`: orientation, thickness, length, bbox.
- `ImageBox`: bbox, dpi_est, caption_candidates.
- `Block`: tập span liền mạch (paragraph/heading/caption).

### 3.2. Article entity
- `article_id`
- `title`, `author`, `lead`, `captions[]`, `body_columns[]`
- `page_refs[]`, `continuation_links{from,to}`
- `confidence_scores{segmentation,classification,ordering,continuation}`
- `audit_flags[]`

## 4) Thuật toán chi tiết theo bước

### Bước A — PDF Ingestion & Chuẩn hóa tọa độ
- Dùng parser PDF vector (khuyến nghị: MuPDF/PyMuPDF hoặc PDFium bindings) lấy toàn bộ text glyph-level + object geometry.
- Chuẩn hóa hệ tọa độ về đơn vị point, origin thống nhất (top-left).
- Tiền xử lý:
  - Gom glyph → word → line → paragraph bằng khoảng cách baseline + khoảng trắng động theo font-size.
  - Nhận diện line mảnh dọc/ngang để suy ranh bài (separator detection).

### Bước B — Phân vùng trang (Layout zoning)
- Dựng occupancy map (text/image/vector density).
- Dùng line separators + whitespace cuts để chia trang thành vùng ứng viên bài (rect/quadrilateral gần chữ nhật).
- Xây graph kề cận vùng: nút = block, cạnh = khoảng cách + không bị separator cắt.
- Chạy clustering (DBSCAN/connected components có ràng buộc separator) để tạo **article clusters** sơ bộ.

### Bước C — Nhận diện thành phần bài (Block role labeling)
**Rule-first, AI-second:**
1. Rule features:
   - Tiêu đề: font lớn nhất cụm, đậm, thường ngắn dòng, nằm đầu vùng hoặc nổi bật trung tâm.
   - Tác giả: mẫu regex (`^[-–]?\s*(Theo|Bài|Tác giả|[A-ZÀ-Ỵ].+)` tùy tòa soạn), cỡ chữ nhỏ hơn title.
   - Lead/Sapo: thường italic/bold nhẹ, nằm dưới title, 1–3 đoạn ngắn.
   - Chú thích ảnh: gần image box nhất, cỡ chữ nhỏ, thường có tiền tố “Ảnh:”.
   - Body: phần văn bản còn lại sau khi trừ các role trên.
2. AI refinement:
   - Model phân loại block (LightGBM/XGBoost hoặc transformer nhỏ) dùng feature hình học + typography + text cues.
   - Với case khó, gọi VLM/LLM nhỏ ở chế độ constrained JSON để “re-rank role labels”, không cho phép rewrite text.

### Bước D — Thứ tự đọc trong bài (Reading order)
- Chia body theo cột trong **phạm vi article cluster**:
  - Projection profile theo trục X để tìm valley (khoảng trắng giữa cột).
  - Nếu cột lệch cao thấp: vẫn giữ thứ tự cột theo X tăng dần; trong từng cột sắp theo Y tăng dần.
- Đối với layout có title/lead ở giữa:
  - Anchor bằng role labels: title/lead đặt trước body bất kể vị trí Y.
- Output bắt buộc theo schema:
  1) title
  2) author
  3) captions (theo thứ tự ảnh trong bài từ trên xuống)
  4) lead
  5) body (column1→columnN)

### Bước E — Ghép bài tiếp trang (Continuation linker)
- Regex đa mẫu cho chỉ dấu:
  - Forward: `XEM TRANG\s+(\d+)`, `XEM TIẾP TRANG\s+(\d+)`
  - Backward: `Tiếp theo trang\s+(\d+)`, `(Tiếp|Xem lại) trang\s+(\d+)`
- Tạo candidate pairs giữa article phần đầu và phần sau:
  - Ràng buộc số trang khớp chỉ dấu.
  - Similarity tiêu đề n-gram + từ khóa đầu đoạn.
  - Kiểm tra tính duy nhất (one-to-one best match).
- Nếu mơ hồ (>=2 cặp điểm gần nhau), gắn `audit_flag: continuation_ambiguous` để duyệt tay.

### Bước F — Loại bỏ Header/Footer/noise
- Học vùng lặp theo nhiều trang (same y-band, text pattern lặp, font nhỏ cố định).
- Rule:
  - Header thường ở top band cố định + lặp tên báo/số kỳ/ngày.
  - Footer thường chứa số trang/đường kẻ chân trang.
- Loại bỏ trước khi article clustering vòng cuối để tránh trộn bài.

### Bước G — Quality gate chống thiếu/thừa/sai ghép
- Kiểm tra coverage:
  - `sum(text_chars_assigned_to_articles) / sum(text_chars_page_without_noise)` phải > ngưỡng (ví dụ 0.985).
- Kiểm tra overlap:
  - Không block nào thuộc >1 article.
- Kiểm tra role completeness:
  - Mỗi article phải có title và body trừ tin ảnh đặc biệt.
- Kiểm tra thứ tự:
  - Body trong cột phải đơn điệu theo Y.
- Bất kỳ lỗi nào -> đưa vào hàng “needs_review” với snapshot vùng nghi vấn.

## 5) Thiết kế AI cụ thể

### 5.1. Thành phần AI nên dùng
- **Layout detector** (fine-tune nhẹ): Detect block semantics trên ảnh render low-res của trang (optional fallback).
- **Block role classifier**: model nhỏ, inference CPU nhanh.
- **LLM/VLM verifier** (ít gọi): chỉ dùng khi confidence thấp hoặc conflict rules.

### 5.2. Prompting an toàn (không sửa nội dung)
- Prompt bắt buộc:
  - “Chỉ trả về chỉ mục block/role, không paraphrase, không sinh text mới.”
  - Output JSON schema có checksum block_id.
- So khớp hậu kiểm:
  - Text output phải lấy trực tiếp từ PDF spans theo block_id, không lấy text do model sinh.

## 6) Hiệu năng để đạt 8 trang ≤ 1 phút

### 6.1. Chiến lược tối ưu
- Parse PDF song song theo trang (4–8 workers tùy CPU).
- Rule engine chạy trước, AI chỉ chạy trên vùng mơ hồ (<20% blocks).
- Cache font stats/tòa soạn template theo đầu báo.
- Dùng ONNX Runtime cho classifier nhỏ.
- Hạn chế render ảnh độ phân giải cao; chỉ render thumbnail khi cần fallback vision.

### 6.2. Mục tiêu ngân sách thời gian tham chiếu
- Parse + primitives: ~2–3s/8 trang
- Clustering + ordering: ~5–8s
- AI selective inference: ~10–20s
- QA + assemble JSON: ~3–5s
- Tổng: ~20–40s (headroom cho network/web overhead)

## 7) Kiến trúc web app đề xuất

### 7.1. Thành phần hệ thống
- **Frontend**: upload PDF, progress realtime, viewer overlay (article box + role labels), màn QA.
- **API Gateway**: nhận job, trả job_id.
- **Worker Service**: chạy pipeline hybrid.
- **Model Service**: host classifier/LLM gateway.
- **Storage**:
  - Object store: PDF + artifacts (overlay, debug snapshot)
  - DB: metadata, kết quả JSON, audit logs.
- **Queue**: Redis/Rabbit/Kafka để xử lý async.

### 7.2. Output API
- `POST /extract` -> `job_id`
- `GET /jobs/{id}` -> status, timings, confidence
- `GET /jobs/{id}/result` -> JSON chuẩn article-level
- `GET /jobs/{id}/overlay` -> dữ liệu hiển thị QA

## 8) JSON output mẫu (rút gọn)
```json
{
  "document_id": "2026-04-05-issue-001",
  "pages": 8,
  "articles": [
    {
      "article_id": "p1_a3",
      "title": "...",
      "author": "...",
      "captions": ["..."],
      "lead": "...",
      "body": "...",
      "page_refs": [1, 3],
      "continuation": {
        "forward_hint": "XEM TRANG 3",
        "backward_hint": "Tiếp theo trang 1"
      },
      "confidence": {
        "segmentation": 0.98,
        "roles": 0.96,
        "ordering": 0.99,
        "continuation": 0.93
      },
      "audit_flags": []
    }
  ]
}
```

## 9) Bộ quy tắc kiểm thử/đánh giá
- **Exact-text fidelity**: Levenshtein distance với ground truth gần 0 (không cho rewrite).
- **Article completeness**: recall ký tự theo từng bài ≥ 99.5%.
- **Article purity**: ký tự sai bài ≤ 0.2%.
- **Role accuracy**: title/byline/lead/caption/body F1 theo block-level.
- **Reading-order accuracy**: tỷ lệ cặp câu đúng thứ tự.
- **Continuation linking accuracy**: precision/recall nối phần bài nhiều trang.
- **Latency SLA**: p95 cho 8 trang ≤ 60s.

## 10) Lộ trình triển khai thực tế (khuyến nghị)
1. **P0 (2–3 tuần):** Rule-based 100% + QA coverage gates.
2. **P1 (2 tuần):** Thêm classifier role + selective AI fallback.
3. **P2 (2 tuần):** Continuation linker nâng cao + active learning từ dữ liệu duyệt tay.
4. **P3:** Tối ưu theo từng đầu báo (template/profile-specific tuning).

## 11) Cơ chế vận hành chống sót nội dung
- Mọi block đều có trạng thái: `assigned / noise / unassigned`.
- Không cho phép job hoàn tất nếu còn `unassigned` vượt ngưỡng.
- Dashboard hiển thị heatmap vùng chưa gán + lý do.
- Log đầy đủ quyết định rule và AI (explainability) để truy vết lỗi.

## 12) Stack kỹ thuật tham khảo
- Parse PDF: PyMuPDF/PDFium
- Geometry/graph: Shapely, NetworkX
- ML: scikit-learn/XGBoost/LightGBM, ONNX Runtime
- Backend: FastAPI + Celery/RQ
- Frontend: React + canvas/SVG overlay
- DB/Queue: PostgreSQL + Redis

## 13) Rủi ro & phương án giảm thiểu
- **Layout cực dị biệt**: tăng ngưỡng fallback AI + review queue.
- **OCR cho text trong ảnh**: tách pipeline OCR riêng cho ảnh có text quan trọng.
- **Sai ghép continuation**: bắt buộc tín hiệu kép (regex + semantic similarity + uniqueness).
- **Độ trễ AI**: chỉ gọi AI khi rule confidence thấp.

## 14) Kết luận
Giải pháp hybrid nên lấy **PDF-vector deterministic pipeline** làm lõi để đảm bảo tốc độ và độ chính xác nền; AI đóng vai trò “trọng tài thông minh” cho tình huống khó. Với cơ chế quality gate + audit queue, hệ thống có thể đạt yêu cầu không sót nội dung, đúng thứ tự cột, đúng thành phần bài, và đáp ứng SLA 8 trang dưới 1 phút.
