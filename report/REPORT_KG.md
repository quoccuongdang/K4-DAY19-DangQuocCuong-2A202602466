# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Đặng Quốc Cường  **MSSV:** 2A202602466  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```text
Chat model: openrouter:openai/gpt-4o-mini | Embedding: openrouter:openai/text-embedding-3-small | top_k=3 | chunk_size=800 | chunks=176 | KG: 204 nodes / 385 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     73.8
graph       196     91958     4807   0.00939    161.1

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     2.02
graph       0.89   1.83     4689       76   0.00074     2.97
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00939 | ×8.38 |
| Indexing giây | 73.8s | 161.1s | ×2.18 |
| Mỗi câu: USD | $0.00013 | $0.00074 | ×5.69 |
| Mỗi câu: giây | 2.02s | 2.97s | ×1.47 |
| Mỗi câu: in_tok | 694 | 4689 | ×6.76 |

**Chi phí tăng thêm đến từ đâu?**
> Ở bước **Indexing**, chi phí tăng thêm chủ yếu đến từ 20 lần gọi LLM để trích xuất entity/relation từ các bài báo sang JSON (tiêu tốn thêm 35.886 input tokens và 4.807 output tokens để dựng Neo4j).
> Ở bước **Querying**, chi phí tăng gấp ~5.7 lần do đồ thị tri thức trả về ngữ cảnh phong phú (multi-hop facts gồm các Điều/Khoản luật và thông tin vụ việc), làm tăng lượng `in_tok` trong prompt từ 694 lên 4.689 tokens.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi định nghĩa đơn lẻ trong Luật PCMT, vector search tìm trúng chunk nên cả hai pipeline đều trả lời xuất sắc. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Thông tin án tử hình của 2 bị cáo nằm trọn trong 1 bài báo, vector retrieval lấy đủ dữ cảnh giúp cả hai đạt điểm tuyệt đối. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Cần bắc cầu từ vụ án sang Điều 251 BLHS; Flat RAG chỉ lấy được bài báo nên trả lời "Không đủ thông tin", còn Graph đi qua node cầu nối Crime lấy đủ Điều 251 khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat RAG không liên kết được bí danh Hoàng Nato sang Điều 255, trong khi Graph tìm thấy bí danh và duyệt đồ thị lấy khoản 4 Điều 255 (tù chung thân). |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Flat RAG nhầm thành "khoản b)", trong khi GraphRAG cung cấp facts chi tiết về Điều 250 và MDMA giúp LLM đối chiếu 9,6kg > 100g thuộc khoản 4 (án tử hình). |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph | Câu hỏi tổng hợp nhiều vụ việc liên quan đến MDMA; GraphRAG nhờ gom cạnh INVOLVES đã liệt kê đầy đủ các vụ án lớn hơn Flat RAG. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy: vụ án không nối được sang luật

- **Hiện tượng:** Có 1 vụ việc trong KB tin tức được tạo thành node `Case` nhưng không có quan hệ `[:CHARGED_WITH]` nào nối sang `Crime`, khiến vụ án bị cô lập và không thể duyệt sang KB luật.
- **Bằng chứng:** Truy vấn Cypher kiểm tra các Case không có liên kết tội danh:

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name, k.doc_id;
```

```text
k.name: "Vụ tông cảnh sát giao thông ở An Giang"
k.doc_id: "news-100260926112415229"
```

- **Nguyên nhân:** Nằm ở bước **prompt trích xuất và thiết kế danh mục tội danh**. Bài báo `news-100260926112415229.md` phản ánh vụ việc đối tượng vận chuyển ma túy tông xe vào CSGT khi bị chặn bắt. Báo chí tập trung mô tả hành vi chống đối và cơ quan công an đang tạm giữ điều tra. Prompt `NEWS_EXTRACTION_PROMPT` ép buộc LLM chỉ được chọn tội danh từ `DANH SÁCH TỘI DANH` (13 tội ma túy của Chương XX BLHS). Do bài báo chưa nêu tội danh chính thức và hành vi chống CSGT (Điều 330 BLHS) không nằm trong danh sách 13 tội, LLM trả về `charges: []`.
- **Đề xuất sửa:** Cho phép ontology bổ sung quan hệ phỏng đoán `[:SUSPECTED_OF]` dựa trên tang vật ma túy thu giữ, hoặc mở rộng danh mục tội danh cho phép các tội liên quan gián tiếp (như Điều 330 Chống người thi hành công vụ). Đánh đổi: Có thể làm tăng số lượng cạnh suy diễn không chắc chắn.

---

### Lỗi E3: Trùng thực thể: một thứ ngoài đời thành nhiều node

- **Hiện tượng:** Cùng một chất ma túy ngoài đời thực nhưng bị phân tách thành 2 node `Substance` khác nhau trong đồ thị do khác biệt chữ hoa/thường (`Ketamine` vs `ketamine`, `Methamphetamine` vs `methamphetamine`).
- **Bằng chứng:** Truy vấn danh sách các chất ma túy trong đồ thị:

```cypher
MATCH (s:Substance) RETURN s.name ORDER BY toLower(s.name);
```

```text
"Ketamine"
"ketamine"
"MDMA"
"Methamphetamine"
"methamphetamine"
```

- **Nguyên nhân:** Nằm ở bước **xây dựng đồ thị (KG-2)**. Khi nạp KB luật, hàm `add_law_article` sử dụng danh sách `SUBSTANCES` gốc viết hoa chữ đầu (`"Ketamine"`, `"Methamphetamine"`). Khi nạp KB tin tức, LLM trích xuất dạng chữ thường (`"ketamine"`). Neo4j constraint `REQUIRE n.name IS UNIQUE` phân biệt chữ hoa/thường (case-sensitive) nên `MERGE` coi đây là 2 thực thể riêng biệt.
- **Đề xuất sửa:** Chuẩn hóa `name` của `Substance` bằng `toLower(name)` hoặc hàm ánh xạ canonical substance trước khi thực hiện câu lệnh Cypher `MERGE`. Đánh đổi: Cần thêm 1 bước tiền xử lý chuẩn hóa chuỗi, chi phí tính toán không đáng kể.

---

### Lỗi E4: Phép đo sai: recall và judge mâu thuẫn nhau

- **Hiện tượng:** Tại câu Q6 (`aggregation`), GraphRAG có chỉ số `recall = 0.33` (rất thấp) nhưng lại đạt điểm đánh giá chất lượng `judge = 1`, phản ánh sự mâu thuẫn giữa đo lường từ khóa tự động và ngữ nghĩa thực tế.
- **Bằng chứng:** Trích từ file `ket_qua_benchmark_kg.txt`:

```text
--- Q6 [aggregation] graph recall=0.33 judge=1 3.17s
Các vụ việc liên quan đến ma túy MDMA trong tin tức bao gồm:

1. Vụ vận chuyển ma túy từ Đức về Việt Nam: Tổng khối lượng hơn 9,6kg MDMA.
2. Vụ góp tiền mua ma túy tại Hà Nội: Có liên quan đến 5 viên MDMA.
3. Vụ việc của Thành: Bị bắt quả tang khi mang 5 viên MDMA đi bán.
4. Vụ việc của Đông: Tại Viện Pháp y tâm thần Trung ương, thu giữ 0,686g ma túy MDMA.

Tất cả các vụ việc này đều có liên quan đến ma túy MDMA.
```

- **Nguyên nhân:** Hàm `keyword_recall` trong `bench_kg.py` yêu cầu bắt buộc xuất hiện chính xác 3 chuỗi con: `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`. Câu trả lời của GraphRAG thực chất đã liệt kê trọn vẹn cả 3 vụ việc (vụ từ Đức của Huy, vụ Thành mua bán, vụ Đông ở Viện Pháp y tâm thần), nhưng LLM tóm tắt theo tên vụ (*"Vụ vận chuyển ma túy từ Đức về Việt Nam"*) và viết tắt tên (*"Thành"* thay vì *"Lê Minh Thành"*). Do so khớp chuỗi tĩnh quá cứng nhắc, `keyword_recall` chỉ tính điểm 1/3 (0.33), trong khi LLM-as-judge đọc hiểu toàn diện ngữ nghĩa nên vẫn cho điểm đạt.
- **Đề xuất sửa:** Mở rộng danh sách `must_include` cho phép regex hoặc tập hợp từ khóa đồng nghĩa (ví dụ: `["Cái Quang Huy|Huy|từ Đức", "Lê Minh Thành|Thành", "Pháp y tâm thần"]`), hoặc ưu tiên dùng điểm số ngữ nghĩa của LLM-as-judge làm thước đo chính.

---

## 4. Kết luận (5 điểm)

1. **Khi nào nên dùng Knowledge Graph (GraphRAG):**
   * Khi bài toán đòi hỏi **liên kết tri thức đa nguồn** (như đối chiếu giữa hồ sơ vụ án thực tế và điều luật chuyên ngành) hoặc **suy luận bắc cầu (multi-hop reasoning)**.
   * Minh chứng: Trên các câu hỏi `cross-kb` (Q3, Q4, Q5), Flat RAG thất bại nặng nề (recall chỉ 0.00 – 0.60, điểm judge 0 – 1) do các đoạn văn bản luật và tin tức nằm phân tán ở các tài liệu khác nhau. Ngược lại, GraphRAG đạt độ chính xác gần như tuyệt đối (recall 1.00, judge 2/2) nhờ duyệt theo quan hệ thực thể.
2. **Khi nào Flat RAG là đủ:**
   * Khi câu hỏi là dạng **truy xuất đơn điểm (single-hop)** hoặc thông tin câu trả lời nằm trọn vẹn trong một ngữ cảnh tài liệu cục bộ (như Q1 tra cứu định nghĩa Luật hay Q2 tra cứu bản án trong 1 bài báo).
   * Minh chứng: Ở Q1 và Q2, Flat RAG đạt điểm số tối đa (recall 1.00, judge 2) ngang bằng GraphRAG nhưng chi phí truy vấn rẻ hơn **5.7 lần** ($0.00013 vs $0.00074) và tốc độ nhanh hơn **1.5 lần** (2.02s vs 2.97s), đồng thời tiết kiệm 100% chi phí xây dựng đồ thị ban đầu.

---

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.06s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00078. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ vận chuyển ma túy từ Đức về Việt Nam, truy tố Điều 250 BLHS).

## Vấn đề gặp phải (không tính điểm)

* Ban đầu file `.env` bị xung đột giữa cấu hình `OPENROUTER_API_KEY` và biến `LLM_PROVIDER=openai` dẫn đến lỗi `RuntimeError: LLM_PROVIDER=openai nhưng chưa có OPENAI_API_KEY`. Sau khi đồng bộ cấu hình sang `openrouter` thì pipeline hoạt động trơn tru.
* Terminal PowerShell trên Windows mặc định mã hóa CP1252 làm phát sinh lỗi `UnicodeEncodeError` khi in các chuỗi tiếng Việt có dấu; đã giải quyết triệt để bằng việc thiết lập môi trường `$env:PYTHONIOENCODING="utf-8"`.
