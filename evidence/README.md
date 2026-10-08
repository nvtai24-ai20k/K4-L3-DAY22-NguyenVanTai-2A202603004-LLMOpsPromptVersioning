# Evidence — Day 22: LangSmith + Prompt Versioning

**Học viên:** Nguyễn Văn Tài — 2A202603004
**Provider:** OpenAI (`gpt-4o-mini`, embeddings `text-embedding-3-small`) · **LangSmith project:** `day22-lab`

| Tệp | Nội dung |
|---|---|
| `01_langsmith_traces.png` | ≥ 50 traces `rag-query` của Bước 1 trên LangSmith |
| `02_prompt_hub.png` | 2 prompt `nguyen-van-tai-rag-prompt-v1` / `-v2` trên Prompt Hub |
| `02_ab_routing_log.txt` | Log A/B routing: push + pull cả 2 prompt từ Hub, nhãn `[prompt-v1]` / `[prompt-v2]` cho 50 câu (V1 = 19, V2 = 31) |
| `03_ragas_scores.png` | Bảng so sánh RAGAS V1 vs V2 |
| `03_ragas_report.json` | Bản sao `data/ragas_report.json` |
| `04_pii_demo_log.txt` | 6 test case PII (email, phone, SSN, thẻ tín dụng, multi-PII, clean) |
| `04_json_demo_log.txt` | 5 test case JSON (hợp lệ, markdown fences, nháy đơn, dấu phẩy thừa, không sửa được) |

---

## Kết quả RAGAS (50 cặp QA × 2 phiên bản prompt)

| Metric | V1 (ngắn gọn, 2-4 câu) | V2 (chuyên gia, có cấu trúc, 3-5 câu) | Chênh lệch |
|---|---|---|---|
| faithfulness | **0.9514** | 0.9273 | V1 +0.024 |
| answer_relevancy | **0.9122** | 0.8902 | V1 +0.022 |
| context_recall | 1.0000 | 1.0000 | hòa |
| context_precision | 0.9483 | 0.9450 | ~ hòa |

Cả 2 phiên bản đều đạt faithfulness ≥ 0.9 (mục tiêu ≥ 0.8).

> Ghi chú: bảng in ra terminal hiển thị `← V2` ở dòng `context_recall` vì code so sánh `s1 > s2`, nên khi hai điểm bằng nhau thì V2 được ghi là thắng. Thực tế hai phiên bản hòa nhau ở chỉ số này.

## Phân tích: vì sao V1 nhỉnh hơn V2

1. **Faithfulness (V1 cao hơn).** Faithfulness = tỉ lệ các *claim* trong câu trả lời được context hỗ trợ. V1 bị giới hạn 2-4 câu nên chủ yếu nêu lại đúng các fact trong context, ít claim hơn và ít chỗ để "thêm thắt". V2 yêu cầu "phân tích, có tổ chức, 3-5 câu", nên model có xu hướng diễn giải, khái quát hoặc nối các ý lại với nhau. Một số câu diễn giải đó không có nguyên văn trong 3 đoạn context, nên RAGAS chấm là không được hỗ trợ.

2. **Answer relevancy (V1 cao hơn).** RAGAS sinh ngược câu hỏi từ câu trả lời rồi so độ tương đồng embedding với câu hỏi gốc. Câu trả lời ngắn, đi thẳng vào ý chính (V1) sinh ra câu hỏi rất sát câu hỏi gốc. Câu trả lời dài hơn, có thêm bối cảnh (V2) làm câu hỏi sinh ra bị "loãng", nên điểm thấp hơn một chút.

3. **Context recall / precision (gần như bằng nhau).** Hai chỉ số này chỉ phụ thuộc vào **retriever** (FAISS, chunk 500/50, k=3) và đáp án chuẩn, không phụ thuộc câu trả lời. Cả V1 và V2 dùng chung một retriever nên context lấy về giống hệt nhau. Recall = 1.0 nghĩa là 3 đoạn context luôn chứa đủ thông tin cho đáp án chuẩn. Chênh lệch 0.003 ở precision là nhiễu của LLM-judge, không phản ánh khác biệt thật giữa 2 prompt.

**Kết luận:** Với knowledge base dạng fact ngắn và câu hỏi định nghĩa như bộ QA này, prompt ngắn gọn (V1) cho câu trả lời trung thực và đúng trọng tâm hơn. V2 phù hợp hơn khi người dùng cần lời giải thích dài, nhưng phải chấp nhận faithfulness giảm nhẹ. Chất lượng retrieval (recall 1.0) là lý do chính khiến cả 2 phiên bản đều đạt faithfulness > 0.9.
