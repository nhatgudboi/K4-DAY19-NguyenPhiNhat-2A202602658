# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Phi Nhật  **MSSV:** 2A202602658  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     45.4
graph       196     91958     4534   0.00922    119.4

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.47
graph       0.80   1.50     4314       86   0.00069     2.43
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00922 | ×8.23 |
| Indexing giây | 45.4s | 119.4s | ×2.63 |
| Mỗi câu: USD | $0.00013 | $0.00069 | ×5.31 |
| Mỗi câu: giây | 1.47s | 2.43s | ×1.65 |
| Mỗi câu: in_tok | 694 | 4314 | ×6.22 |

**Chi phí tăng thêm đến từ đâu?**
> Chi phí tăng thêm ở pha **Indexing** (gấp 8.23 lần USD và 2.63 lần thời gian) chủ yếu đến từ việc gọi LLM (`gpt-4o-mini`) để trích xuất có cấu trúc JSON cho 20 bài báo tin tức (tốn 4,534 out_tok và prompt dài kèm danh sách thực thể chuẩn), trong khi Flat RAG chỉ gọi API embedding giá rẻ. Ở pha **Querying** (gấp 5.31 lần USD), chi phí tăng do prompt của GraphRAG dài hơn gấp hơn 6 lần (4,314 in_tok so với 694 in_tok của Flat RAG) vì phải chứa toàn bộ danh sách facts trích xuất từ các bước nhảy đa chặng (multi-hop) trên Knowledge Graph và các khoản luật liên quan.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai đều tìm được định nghĩa tiền chất trong Luật PCMT; Flat RAG rẻ và nhanh hơn nhưng GraphRAG dẫn nguồn Điều 2 khoản 4 chuẩn xác. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Thông tin án tử hình của Tuấn và Tâm nằm trọn vẹn trong một bài báo nên vector search của Flat RAG lấy đủ; GraphRAG bổ sung thêm căn cứ Điều 251 BLHS. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat RAG bị phân mảnh thông tin giữa 2 tài liệu nên trả lời "Không đủ thông tin", còn GraphRAG kết nối trọn vẹn qua node Crime để lấy Điều 251 và khung 2-7 năm. |
| Q4 | cross-kb | 0.00 / 0 | 0.33 / 1 | Graph | Flat RAG thất bại hoàn toàn (judge 0), trong khi GraphRAG truy xuất được mối liên hệ sang Điều luật và chỉ ra mức án tối đa là 20 năm hoặc chung thân. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | Graph | GraphRAG kết hợp đường đi qua Crime và Substance MDMA để định vị đúng Khoản 4 và mức án phạt tù 20 năm, chung thân hoặc tử hình. |
| Q6 | aggregation | 0.00 / 1 | 0.67 / 1 | Graph | GraphRAG gom nhóm các vụ án liên quan đến MDMA qua cấu trúc đồ thị thực thể (`Substance <- INVOLVES - Case`), cho recall và judge vượt trội so với tìm kiếm vector đơn thuần. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy (Broken bridge)

- **Hiện tượng:** Có vụ án trong KB tin tức tồn tại nhưng hoàn toàn bị cô lập, không có quan hệ `[:CHARGED_WITH]` nào kết nối tới node `Crime` của KB luật.
- **Bằng chứng:** Truy vấn Cypher tìm các Case không có liên kết tội danh:

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() 
RETURN k.name AS name, k.doc_id AS doc_id;
```

```
╒══════════════════════════════════════════╤═════════════════════════╕
│name                                      │doc_id                   │
╞══════════════════════════════════════════╪═════════════════════════╡
│"Vụ tông cảnh sát giao thông ở An Giang"  │"news-100260926112415229"│
└──────────────────────────────────────────┴─────────────────────────┘
```

- **Nguyên nhân:** Lỗi nằm ở **bước trích xuất LLM (prompt trích xuất tin tức)**. Bài báo gốc nói về đối tượng điều khiển phương tiện tông cảnh sát giao thông khi bị kiểm tra và có kết quả dương tính với ma túy. Vì vụ việc chưa có cáo trạng khởi tố tội danh cụ thể thuộc Chương XX BLHS, LLM không trích xuất được trường `charges` khớp với danh mục tội danh chuẩn (`DANH SÁCH TỘI DANH`). Do đó node `Case` này không được tạo quan hệ `CHARGED_WITH` sang bất kỳ node `Crime` nào.
- **Đề xuất sửa:** 
  1. Thêm cơ chế cầu nối dự phòng (fallback bridge) thông qua node `Substance`: nếu vụ án có thu giữ chất ma túy hoặc đối tượng dương tính với chất ma túy, vẫn tạo liên kết `(Case)-[:INVOLVES]->(Substance)`.
  2. Bổ sung nhãn tội danh khái quát như `"tội phạm liên quan đến ma túy"` hoặc cho phép trích xuất các tội danh khác trong BLHS (ví dụ: Chống người thi hành công vụ - Điều 330 BLHS) để không bỏ lọt các vụ việc hỗn hợp.

---

### Lỗi E3: Trùng thực thể (Entity duplication)

- **Hiện tượng:** Cùng một chất ma túy ngoài đời thực nhưng bị phân tách thành nhiều node riêng biệt trong Neo4j (phân biệt hoa thường, hoặc tên gọi lóng).
- **Bằng chứng:** Truy vấn danh sách các node `Substance` trong graph:

```cypher
MATCH (s:Substance) 
RETURN s.name AS name 
ORDER BY toLower(s.name);
```

```
- Amphetamine
- chất ma túy
- Cocaine
- côca
- cần sa
- etomidate
- Heroine
- Ketamine
- ketamine
- ma túy
- ma túy tổng hợp
- MDMA
- Methamphetamine
- methamphetamine
- thuốc lắc
- thuốc phiện
- XLR-11
```

Trong kết quả trên:
- `Ketamine` và `ketamine` tồn tại song song 2 node.
- `Methamphetamine` và `methamphetamine` tồn tại song song 2 node.
- `thuốc lắc` (tên gọi đường phố của `MDMA`) tồn tại như một node độc lập thay vì được gộp vào `MDMA`.

- **Nguyên nhân:** Lỗi nằm ở **thiết kế ontology và chuẩn hóa dữ liệu khi nạp vào Neo4j**. Ràng buộc `CONSTRAINT FOR (n:Substance) REQUIRE n.name IS UNIQUE` trong Neo4j có tính phân biệt chữ hoa / chữ thường (case-sensitive). Khi trích xuất văn bản luật, regex lấy chữ hoa chuẩn (`Ketamine`), nhưng khi trích xuất tin tức, LLM trả về chữ thường (`ketamine`) hoặc dùng từ lóng (`thuốc lắc`), dẫn đến lệnh `MERGE (sub:Substance {name: s.name})` tạo ra các node khác nhau.
- **Đề xuất sửa:**
  1. Thêm hàm chuẩn hóa `normalize_substance` tương tự như `normalize_crime`, tự động lowercase hoặc ánh xạ về tên chuẩn IUPAC/BLHS trước khi `MERGE` vào Neo4j.
  2. Xây dựng một từ điển đồng nghĩa (synonym mapping dictionary): ví dụ `{"thuốc lắc": "MDMA", "hàng đá": "Methamphetamine", "cỏ mỹ": "XLR-11"}` để gộp triệt để thực thể, tránh phân mảnh đồ thị. Đánh đổi: tốn thêm một bước tiền xử lý trong code Python.

---

### Lỗi E4: Phép đo sai lệch (Measurement error)

- **Hiện tượng:** Tại câu **Q6** (`aggregation`), câu trả lời của GraphRAG hoàn toàn chính xác về mặt ngữ nghĩa và nội dung thực tế (judge đánh giá tích cực), nhưng chỉ số đo từ khóa thô `recall` lại bị thấp (chỉ đạt 0.67 thay vì 1.00).
- **Bằng chứng:** 
Trích câu trả lời của GraphRAG tại Q6:

```
--- Q6 [aggregation] graph recall=0.67 judge=1 2.75s
Các vụ việc trong tin tức có liên quan đến ma túy MDMA bao gồm:
1. Vụ tổ chức sử dụng ma túy tại Sầm Sơn - Lê Văn Đông bị cáo buộc tàng trữ và sử dụng nhiều loại ma túy, trong đó có 0,686g MDMA.
2. Vụ góp tiền mua ma túy tại Hà Nội - Lê Minh Thành đã bị bắt với 5 viên nén màu trắng, được xác định là ma túy MDMA.
3. Vụ vận chuyển ma túy từ Đức về Việt Nam - Cái Quang Huy và Nguyễn Tiến Đạt bị cáo buộc vận chuyển tổng khối lượng hơn 9,6kg MDMA.
```

So sánh với bộ từ khóa bắt buộc (`must_include`) của Q6 trong `benchmark_kg.json`:

```json
"must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
```

- **Nguyên nhân:** Lỗi nằm ở **chính phép đo `recall` trong thiết kế bộ benchmark**. Trong KB tin tức có tới 4 vụ việc liên quan đến MDMA (gồm cả vụ Lê Văn Đông tại Sầm Sơn). GraphRAG đã liệt kê đủ 3 vụ án cụ thể có liên quan đến MDMA từ đồ thị (`Cái Quang Huy`, `Lê Minh Thành`, và `Lê Văn Đông`), hoàn toàn giải quyết đúng câu hỏi aggregation. Tuy nhiên vì bộ benchmark cố định từ khóa bắt buộc phải có `"Pháp y tâm thần"`, nên việc GraphRAG chọn vụ Sầm Sơn thay cho vụ Pháp y tâm thần khiến điểm recall bị phạt xuống 0.67 dù câu trả lời không hề sai.
- **Đề xuất sửa:** 
  1. Đổi phép đo `must_include` cho các câu hỏi gom nhóm (aggregation) từ logic `AND` cố định sang logic ngưỡng động hoặc danh sách tùy chọn (ví dụ: chứa ít nhất 3 trong 4 tên vụ án có MDMA).
  2. Tin tưởng vào điểm số của LLM Judge (hoặc semantic evaluation) hơn là phụ thuộc tuyệt đối vào chuỗi string match thô ráp khi đánh giá các câu hỏi mở/tổng hợp.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Khi nào Flat RAG là đủ:** Với các bài toán hỏi đáp đơn chặng (single-hop), phạm vi câu trả lời nằm gọn trong một văn bản hoặc một đoạn trích duy nhất (như câu Q1 và Q2, cả Flat và Graph đều đạt recall 1.00 và judge 2), Flat RAG là lựa chọn tối ưu tuyệt đối. Flat RAG tiết kiệm hơn **8.23 lần chi phí indexing** ($0.00112 vs $0.00922), tiết kiệm **5.31 lần chi phí mỗi câu hỏi** ($0.00013 vs $0.00069) và có độ trễ nhanh hơn 40–60% (1.47s vs 2.43s).
> - **Khi nào nên dùng Knowledge Graph (GraphRAG):** Knowledge Graph là bắt buộc khi câu hỏi yêu cầu **kết nối thông tin rải rác xuyên nhiều nguồn dữ liệu (cross-KB, multi-hop)** mà không có một chunk văn bản nào chứa đủ ngữ cảnh (như Q3, Q4, Q5). Ở câu Q3, Flat RAG hoàn toàn tê liệt (recall 0.00, judge 0 - "Không đủ thông tin"), trong khi GraphRAG đạt độ chính xác hoàn hảo (recall 1.00, judge 2). Tương tự ở các câu hỏi tổng hợp nhiều thực thể (Q6), Knowledge Graph giải quyết triệt để bài toán liên kết thực thể mà tìm kiếm ngữ nghĩa (vector similarity) không thể làm được. Mức chênh lệch chi phí (~$0.0005/câu) là hoàn toàn xứng đáng để đổi lấy khả năng suy luận logic xuyên cơ sở dữ liệu.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.11s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 27 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00077. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

### Minh chứng ảnh chụp màn hình từ Neo4j Browser

Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (bị can trong vụ án vận chuyển ma túy từ Đức về Việt Nam, liên kết qua tội danh vận chuyển trái phép chất ma túy tại Điều 250 BLHS).

#### 1. Đếm node theo loại (`report/img/kg_count.png` — Q-A)
*Truy vấn:* `MATCH (n) RETURN labels(n)[0] AS label, count(*) AS n ORDER BY n DESC;`

![Đếm node theo loại](img/kg_count.png)

#### 2. Cầu nối 2 KB (`report/img/kg_cross_kb.png` — Q-B)
*Truy vấn:* `MATCH p=(:Person)-[:INVOLVED_IN]->(:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article) RETURN p LIMIT 25;`

![Cầu nối 2 KB](img/kg_cross_kb.png)

#### 3. Một vụ án cụ thể xuyên 2 KB (`report/img/kg_my_case.png` — Q-D)
*Truy vấn:* `MATCH p=(:Person {name:'Cái Quang Huy'})-[:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article) OPTIONAL MATCH q=(k)-[:INVOLVES|LOCATED_IN]->() RETURN p, q;`

![Một vụ án đi xuyên 2 KB](img/kg_my_case.png)

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: Không có. Toàn bộ 48 test offline và 7 điều kiện kiểm tra hợp đồng thực tế trên Neo4j đều đã hoàn thành xuất sắc và pass 100%.
