# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Châu Tùng Dương  **MSSV:** 2A202602822  
**Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    117.1
graph       196     34619     6200   0.00594    200.0

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       73   0.00010     3.62
graph       0.94   1.83     5657      158   0.00063     5.11
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00594 | (Flat = $0, Graph tốn thêm $0.00594) |
| Indexing giây | 117.1 | 200.0 | ×1.71 |
| Mỗi câu: USD | $0.00010 | $0.00063 | ×6.30 |
| Mỗi câu: giây | 3.62 | 5.11 | ×1.41 |
| Mỗi câu: in_tok | 696 | 5657 | ×8.13 |

**Chi phí tăng thêm đến từ đâu?**
> 1. Ở giai đoạn **Indexing**: GraphRAG tốn thêm $0.00594 USD và 82.9 giây để thực hiện 20 lượt gọi LLM (`gemini-3.5-flash-lite`) trích xuất các thực thể và quan hệ có cấu trúc (`Case`, `Person`, `Substance`, `Crime`) từ 20 bài báo tin tức. Flat RAG chỉ tạo vector embedding nên không phát sinh chi phí LLM trích xuất.
> 2. Ở giai đoạn **Querying**: Chi phí mỗi câu hỏi của GraphRAG cao hơn 6.3 lần ($0.00063 so với $0.00010) và số lượng input token tăng hơn 8.1 lần (5,657 so với 696) vì prompt của GraphRAG được bổ sung toàn bộ các dữ kiện mở rộng từ Knowledge Graph (tóm tắt vụ việc, danh sách các khoản luật và hình phạt tương ứng liên quan đến chất ma túy của vụ án).

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa (Flat rẻ hơn) | Khái niệm tiền chất nằm gọn trong Điều 2 Luật PCMT nên vector search lấy đúng chunk và trả lời trọn vẹn. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa (Flat rẻ hơn) | Danh tính 2 bị cáo tử hình nằm trọn vẹn trong một bài báo tin tức, vector top-k lấy đủ thông tin. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | GraphRAG | Flat RAG chỉ tìm được mức án 36 tháng trong tin tức nhưng thiếu luật, GraphRAG đi qua node cầu nối Crime sang Điều 251 khoản 1. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | GraphRAG | Flat RAG không có thông tin khung phạt; GraphRAG tìm được Điều 255 và khung phạt cơ bản của khoản 1. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | GraphRAG | Câu hỏi đòi hỏi xâu chuỗi Cái Quang Huy → 9.6kg MDMA → Khoản 4 Điều 250 (tử hình); Flat RAG thiếu dữ kiện luật, GraphRAG trả lời chính xác tuyệt đối. |
| Q6 | aggregation | 0.00 / 2 | 1.00 / 2 | GraphRAG | GraphRAG tổng hợp đầy đủ tên cả 3 vụ án nhờ truy vấn ngược từ node Substance "MDMA", Flat RAG chỉ nhớ tên tắt nên recall bằng 0. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (sai khung hình phạt tối đa dù graph có đủ Điều luật)

- **Hiện tượng:** Tại câu Q4 ("Giang hồ 'Hoàng Nato' bị bắt về hành vi gì, và hành vi đó có thể bị phạt tù tối đa bao nhiêu theo Bộ luật Hình sự?"), GraphRAG chỉ trả lời mức phạt tù tối đa là 07 năm tù (theo Khoản 1 Điều 255), trong khi đáp án chuẩn là 20 năm tù hoặc tù chung thân (thuộc Khoản 4 Điều 255).
- **Bằng chứng:** 
  Trích nguyên văn câu trả lời Q4 của GraphRAG trong `ket_qua_benchmark_kg.txt`:
  > *"Về mức hình phạt tù tối đa, dựa theo Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy khoản 1, hành vi tổ chức sử dụng trái phép chất ma túy dưới bất kỳ hình thức nào có mức phạt tù từ 02 năm đến 07 năm (ngữ cảnh không đề cập chi tiết các khoản cao hơn của Điều 255). Do đó, mức phạt tù được nêu trong ngữ cảnh là tối đa đến 07 năm."*

  Truy vấn Cypher kiểm tra cấu trúc Điều 255 trong Neo4j:
```cypher
MATCH (a:Article)-[:HAS_CLAUSE]->(cl:Clause)
WHERE a.id CONTAINS '255'
RETURN cl.number AS number, cl.penalty AS penalty ORDER BY number;
```

```
number | penalty
1      | "phạt tù từ 02 năm đến 07 năm"
2      | "phạt tù từ 07 năm đến 15 năm"
3      | "phạt tù từ 15 năm đến 20 năm"
4      | "phạt tù 20 năm hoặc tù chung thân"
```

- **Nguyên nhân:** Lỗi nằm ở quy tắc lọc khoản luật trong hàm `Neo4jGraph.context` (KG-3). Quy tắc hiện tại là: `WHERE (cl.number = 1 OR EXISTS { (k)-[:INVOLVES]->(:Substance)<-[:MENTIONS]-(cl) })`. Vì vụ án Hoàng Nato bị khởi tố về hành vi tổ chức sử dụng thuần túy (không gắn với khối lượng chất cụ thể nào quy định ở các khoản sau), câu truy vấn chỉ lấy duy nhất Khoản 1 và bỏ qua các khoản tăng nặng. Do đó LLM không có dữ kiện về Khoản 4 để suy ra mức án tối đa là "chung thân".
- **Đề xuất sửa:** Mở rộng Cypher trong `context()`: nếu câu hỏi chứa các cụm từ truy vấn trần hình phạt ("tối đa", "cao nhất", "kịch khung"), truy vấn thêm Khoản có số thứ tự lớn nhất hoặc có penalty cao nhất của Điều luật tương ứng:
  `MATCH (a)-[:HAS_CLAUSE]->(cl) WITH a, cl ORDER BY cl.number DESC WITH a, head(collect(cl)) AS max_cl RETURN max_cl`.
  *Đánh đổi:* Thêm khoảng 30–50 input tokens vào prompt, nhưng giải quyết triệt để các câu hỏi về khung hình phạt kịch khung.

---

### Lỗi E4: Phép đo sai (Recall và Judge mâu thuẫn nhau)

- **Hiện tượng:** Tại câu Q6 đối với Flat RAG, `recall = 0.00` nhưng `judge = 2` (điểm tuyệt đối). Đây là sự mâu thuẫn trực tiếp giữa phép đo từ khóa bắt buộc và phép đo bằng mô hình ngôn ngữ.
- **Bằng chứng:** 
  Trích file `ket_qua_benchmark_kg.txt`:
  > `--- Q6 [aggregation] flat recall=0.00 judge=2 3.67s`
  > `Dựa trên ngữ cảnh, cả 3 vụ việc đều có liên quan đến ma túy MDMA:`
  > `1. Vụ việc thứ nhất: Lực lượng chức năng phát hiện thùng hàng chứa các viên nén màu xanh là MDMA (khối lượng gần 4,3kg) liên quan đến Đạt và Huy.`
  > `2. Vụ việc thứ hai: Thành bị bắt quả tang khi mang 5 viên nén màu trắng đi bán, kết quả giám định là ma túy MDMA.`
  > `3. Vụ việc thứ ba: Kết quả giám định xác định các viên nén hình tam giác màu hồng - xám trong kiện hàng là MDMA (khối lượng hơn 5,3kg) liên quan đến Đức và Huy.`

  Đối chiếu với `must_include` trong `data/benchmark_kg.json`:
```json
"must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
```

- **Nguyên nhân:** Lỗi nằm ở thiết kế của hàm đo `keyword_recall`:
  Hàm này so khớp chuỗi con tuyệt đối không dấu phân cách từ (`k.lower() in answer.lower()`). Báo chí thường gọi tên nhân vật ngắn gọn ("Huy", "Đạt và Huy", "Thành"). Flat RAG tóm tắt lại đúng theo câu từ của bài báo nên chỉ nêu "Thành", "Đạt và Huy", dẫn đến việc thiếu cả họ đệm ("Cái Quang Huy", "Lê Minh Thành"). Ngoài ra, Flat RAG liệt kê tách 2 kiện hàng của cùng một vụ Nội Bài thành 2 vụ việc và bỏ sót vụ Pháp y tâm thần. LLM Judge linh hoạt nhận thấy Flat RAG chỉ ra được đúng các tình tiết MDMA có thật nên đã cho điểm 2, trong khi `keyword_recall` đánh giá 0.00 vì thiếu chuỗi con cố định.
- **Đề xuất sửa:** 
  1. Trong benchmark, trường `must_include` nên hỗ trợ danh sách các biến thể hoặc regex (ví dụ: `["Cái Quang Huy|Huy", "Lê Minh Thành|Thành"]`).
  2. Bổ sung trích xuất tên chuẩn hóa thực thể ngay trong prompt generation để mô hình luôn dùng danh xưng đầy đủ.
  3. Khi đánh giá hệ thống, không nên phụ thuộc 100% vào keyword recall mà cần kết hợp cả Semantic Judge hoặc đối sánh thực thể (Entity F1).

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2:

> 1. **Nên dùng Knowledge Graph (GraphRAG) khi:**
>    - Bài toán yêu cầu liên kết thông tin phân mảnh qua nhiều nguồn dữ liệu độc lập (Cross-KB, Multi-hop). Minh chứng: Ở các câu Q3 và Q5, Flat RAG hoàn toàn bất lực trong việc tìm Điều luật và khung phạt (recall chỉ đạt 0.33 và 0.40, judge = 1), trong khi GraphRAG đạt recall 1.00 và judge = 2 nhờ đi qua node cầu nối `Crime`.
>    - Bài toán yêu cầu tổng hợp đa thực thể (Aggregation). Minh chứng: Ở câu Q6, GraphRAG đạt recall 1.00 nhờ khả năng duyệt ngược đồ thị từ thực thể `Substance`, trong khi Flat RAG bị sót và recall = 0.00.
>    - Hệ thống đòi hỏi độ tin cậy cao, có thể giải trình nguồn gốc (truy vết rõ ràng theo Điều, khoản, mối liên hệ thực thể).
>
> 2. **Flat RAG là đủ khi:**
>    - Câu hỏi thuộc dạng đơn bước (Single-hop), câu trả lời nằm trọn vẹn trong một đoạn văn bản cục bộ. Minh chứng: Ở câu Q1 và Q2, cả Flat RAG và GraphRAG đều đạt recall 1.00 và judge = 2.
>    - Tuy nhiên, Flat RAG tiết kiệm hơn **6.3 lần chi phí mỗi câu hỏi** ($0.00010 so với $0.00063 USD), tiêu tốn ít hơn **8 lần token đầu vào** (696 so với 5657 tokens), và phản hồi nhanh hơn **30%** (3.62s so với 5.11s).
>
> **Kết luận hòa vốn:** Knowledge Graph hoàn toàn đáng tiền và là thành phần bắt buộc nếu hệ thống phục vụ nghiệp vụ tra cứu/tư vấn pháp lý chuyên sâu. Nhưng đối với các hệ thống hỏi đáp tài liệu thông thường với phần lớn câu hỏi đơn giản, Flat RAG vẫn là sự lựa chọn tối ưu về chi phí và tốc độ.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.26s
```

```
$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 18 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00053. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Lê Văn Đông** (Vụ sai phạm tại Viện Pháp y tâm thần Trung ương và tổ chức sử dụng trái phép chất ma túy tại Sầm Sơn).

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: Không có. Toàn bộ các bước cài đặt code, thiết kế ontology, kết nối cơ sở dữ liệu Neo4j, kiểm thử hợp đồng và chạy benchmark đã hoàn tất thành công 100%.
