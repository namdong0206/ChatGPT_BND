# Ứng dụng dịch tài liệu Anh → Việt

Ứng dụng web Flask cho phép:

- Tải lên file `.txt`, `.docx`, hoặc `.pdf`.
- Trích xuất nội dung gốc tiếng Anh.
- Dịch sang tiếng Việt và trau chuốt văn phong tự nhiên bằng OpenAI API.
- Hiển thị kết quả trong trình biên tập để chỉnh sửa thủ công.
- Xuất nội dung cuối dưới dạng `.txt` hoặc `.docx`.

## Cài đặt

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Chạy ứng dụng

```bash
export OPENAI_API_KEY="<your_api_key>"
# tùy chọn
export OPENAI_MODEL="gpt-4.1-mini"
python app.py
```

Mở trình duyệt tại: `http://localhost:8000`

## Lưu ý

- Ứng dụng yêu cầu `OPENAI_API_KEY` để thực hiện dịch và biên tập.
- Dung lượng file upload tối đa mặc định: 25 MB.
