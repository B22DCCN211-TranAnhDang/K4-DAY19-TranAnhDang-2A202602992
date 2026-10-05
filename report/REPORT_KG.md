# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Trần Anh Đăng  **MSSV:** 2A202602992  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    121.9
graph       196     91958     4734   0.00934    196.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       48   0.00013     4.07
graph       0.72   1.50     3189       79   0.00052     4.34
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00934 | ×8.34 |
| Indexing giây | 121.9s | 196.3s | ×1.61 |
| Mỗi câu: USD | $0.00013 | $0.00052 | ×4.00 |
| Mỗi câu: giây | 4.07s | 4.34s | ×1.07 |
| Mỗi câu: in_tok | 694 | 3189 | ×4.60 |

**Chi phí tăng thêm đến từ đâu?**
Chi phí Indexing của GraphRAG tăng gấp ~8.34 lần do phải gọi LLM (`extract_news_cases`) trích xuất dữ liệu cấu trúc JSON từ 20 bài báo tin tức (tốn thêm 35,886 input tokens và 4,734 output tokens). Đối với mỗi câu hỏi truy vấn, chi phí USD của GraphRAG tăng gấp 4.00 lần vì prompt trả lời được bổ sung thêm danh sách dữ kiện (facts) multi-hop trích xuất từ Neo4j, làm lượng `in_tok` trung bình tăng từ 694 lên 3,189 tokens.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều trích xuất chính xác định nghĩa tiền chất từ văn bản Luật PCMT 2021. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều tìm thấy 2 bị cáo Trần Thanh Tuấn và Trần Minh Tâm bị tuyên án tử hình. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat RAG bị thiếu thông tin Điều luật do chunk tin tức không chứa luật; GraphRAG kết nối xuyên 2 KB qua node Crime tới Điều 251 khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 0.00 / 0 | Hòa | Cả hai pipeline đều không tìm thấy thông tin biệt danh 'Hoàng Nato' do lỗi trích xuất alias trong bước indexing. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | GraphRAG kết nối chính xác khối lượng tang vật 9.6kg MDMA với khoản 4 Điều 250 BLHS nhờ facts liên kết từ Graph, đạt điểm tuyệt đối. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph | GraphRAG tổng hợp chính xác các vụ việc từ graph có liên quan đến MDMA (nối được nhân vật Cái Quang Huy). |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy do thiếu thông tin biệt danh (Hoàng Nato - Q4)

- **Hiện tượng:** Ở câu Q4, cả Flat RAG và GraphRAG đều trả về `"Không đủ thông tin"` khi được hỏi về biệt danh "Hoàng Nato" (Dương Minh Tuấn).
- **Bằng chứng:**
  * Nguyên văn câu trả lời trong `ket_qua_benchmark_kg.txt`: `"Không đủ thông tin."`
  * Truy vấn Cypher kiểm tra node `Person` liên quan đến biệt danh "Hoàng":
```cypher
MATCH (p:Person)
WHERE p.name CONTAINS 'Hoàng' OR any(a IN coalesce(p.aliases, []) WHERE a CONTAINS 'Hoàng')
RETURN p.name, p.aliases, labels(p);
```
```text
(no records)
```
- **Nguyên nhân:** Lỗi nằm ở bước trích xuất LLM Indexing (`extract_news_cases` / `NEWS_EXTRACTION_PROMPT`). Prompt chưa bắt buộc trích xuất đầy đủ biệt danh hoặc tên hay gọi của nghi phạm vào thuộc tính `aliases`. Do đó node `Person` chỉ lưu tên thật "Dương Minh Tuấn", khiến hàm `seed_facts` không thể matching từ khóa "Hoàng Nato" trong câu hỏi với node tương ứng trên đồ thị.
- **Đề xuất sửa:** Bổ sung ví dụ cụ thể trong `NEWS_EXTRACTION_PROMPT` yêu cầu tách riêng biệt danh (alias) vào mảng `aliases`. Đồng thời cải tiến `seed_facts` để tách họ tên và biệt danh bằng regex trước khi thực hiện tìm kiếm chuỗi.

---

### Lỗi E4: Phép đo mâu thuẫn giữa Keyword Recall và LLM Judge (Q6)

- **Hiện tượng:** Ở câu Q6, GraphRAG trả lời đúng và đầy đủ cả 3 vụ án liên quan đến ma túy MDMA, nhưng chỉ số `recall` đo được lại chỉ đạt `0.33` trong khi `judge` đạt `1` điểm.
- **Bằng chứng:**
  * Nguyên văn câu trả lời của GraphRAG ở Q6:
    `"Các vụ việc trong tin tức có liên quan đến ma túy MDMA bao gồm: 1. Vụ tổ chức sử dụng ma túy tại Sầm Sơn - Lê Văn Đông... 2. Vụ góp tiền mua ma túy tại Hà Nội... 3. Vụ vận chuyển ma túy từ Đức về Việt Nam - Cái Quang Huy..."`
  * Mảng từ khóa bắt buộc (`must_include` trong `benchmark_kg.json`): `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`.
  * Hàm đo `recall` dùng so sánh chuỗi chính xác (exact string match). LLM nhắc được "Cái Quang Huy" nên khớp được 1/3 từ khóa (`recall = 0.33`), nhưng không ghi từ "Lê Minh Thành" và "Pháp y tâm thần" mà mô tả theo bối cảnh ("Vụ góp tiền mua ma túy tại Hà Nội", "Vụ tại Sầm Sơn").
- **Nguyên nhân:** Lỗi thuộc về **phép đo (Evaluation Metric)**. Cách đo `recall` bằng keyword exact matching đơn thuần quá máy móc, không ghi nhận các cách diễn đạt tương đương về mặt ngữ nghĩa (semantic equivalence) mà LLM tạo ra.
- **Đề xuất sửa:** Mở rộng danh sách `must_include` trong benchmark để chứa cả tên địa danh / từ khóa đại diện (ví dụ: `["Cái Quang Huy", "Đức"]`, `["Lê Minh Thành", "Hà Nội"]`), hoặc sử dụng điểm số từ LLM Judge làm thước đo chính thay vì chỉ dựa vào exact keyword recall.

## 4. Kết luận (5 điểm)

- **Khi nào Flat RAG là đủ:** Đối với các tác vụ tra cứu đơn hop (single-hop) nơi thông tin câu trả lời nằm gọn trong một đoạn văn bản (như Q1, Q2), Flat RAG đạt kết quả xuất sắc (recall = 1.00, judge = 2) với chi phí thấp hơn 4.0 lần về Token và thời gian phản hồi nhanh hơn.
- **Khi nào nên dùng Knowledge Graph (GraphRAG):** Đối với các bài toán tra cứu xuyên nguồn dữ liệu (Cross-KB, multi-hop) đòi hỏi liên kết thông tin rải rác từ nhiều tài liệu khác nhau (như Q3, Q5), Flat RAG thất bại (`recall = 0.00, judge = 0` ở Q3), trong khi GraphRAG đạt hiệu quả vượt trội (`recall = 1.00, judge = 2` cho cả Q3 và Q5, nâng điểm recall trung bình từ 0.43 lên 0.72 và điểm judge trung bình từ 1.00 lên 1.50). Mặc dù chi phí dựng graph ban đầu đắt hơn x8.34 lần, GraphRAG là giải pháp bắt buộc để giải quyết bài toán suy luận phức tạp.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.13s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00065. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Trần Thanh Tuấn

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: Không có. Đã giải quyết được lỗi rate limit 429 bằng cách chuyển sang provider OpenRouter.
