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
| Q4 | cross-kb | 0.00 / 0 | 0.33 / 1 | Graph (yếu) | Flat RAG thất bại hoàn toàn. GraphRAG kéo được mối liên hệ sang KB luật nên recall khác 0, **nhưng trả lời sai tội danh**: đáp án chuẩn là *tổ chức sử dụng* (Điều 255) còn hệ thống dẫn tới *tàng trữ* (Điều 249) — xem lỗi E5. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | Hòa (đều sai) | Cả hai pipeline đều nói **Điều 251** trong khi đáp án chuẩn là **Điều 250 khoản 4**; GraphRAG chỉ nhỉnh hơn ở chi tiết khối lượng — xem lỗi E5. |
| Q6 | aggregation | 0.00 / 1 | 0.67 / 1 | Graph | GraphRAG gom nhóm các vụ án liên quan đến MDMA qua cấu trúc đồ thị thực thể (`Substance <- INVOLVES - Case`), cho recall và judge vượt trội so với tìm kiếm vector đơn thuần. |

> **Quy luật:** GraphRAG thắng rõ ở nhóm **cross-kb / aggregation** (Q3, Q4, Q6) vì đó là lúc thông tin bắt buộc phải nằm ở 2 tài liệu khác nhau mà không chunk nào chứa trọn. Ở nhóm **single-hop** (Q1, Q2) hai bên hòa nhau, nên ưu thế của KG không hiện ra. Nhưng lưu ý: cả 3 câu thắng đều là câu mà KG **có dữ kiện đúng**; hai câu mà KG *cầu nối được nhưng ánh xạ sai* (Q4, Q5) lại cho thấy độ hơn về recall chưa đi kèm độ hơn về tính đúng — xem lỗi E5.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy (Broken bridge)

- **Hiện tượng:** Có vụ án trong KB tin tức tồn tại trong graph nhưng hoàn toàn bị cô lập, không có quan hệ `[:CHARGED_WITH]` nào nối tới node `Crime` của KB luật.
- **Bằng chứng 1 — Cypher trên Neo4j Browser**, truy vấn các Case không có liên kết tội danh:

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

- **Bằng chứng 2 — output thô của LLM** cho đúng bài báo đó (chạy lại `NEWS_EXTRACTION_PROMPT` với `news-100260926112415229`, in nguyên văn trước khi `link_entity` lọc):

```json
{"cases": [{
  "name": "Vụ tông cảnh sát giao thông ở An Giang",
  "summary": "Nguyễn Minh Nhân đã tông vào Thiếu tá Trần Ngọc Nam trong khi chạy xe không có giấy phép lái xe và sử dụng rượu, ma túy.",
  "date": "2023-09-10",
  "location": "An Giang",
  "charges": ["chống người thi hành công vụ"],
  "substances": [{"name": "ma túy", "amount": ""}],
  "people": [{ "name": "Nguyễn Minh Nhân", "aliases": [], "role": "bị can",
               "charge": "chống người thi hành công vụ", "sentence": "" }]
}]}
```

Đối chiếu hai vế cho thấy đúng chuỗi nhân quả:

| Bước | Kết quả quan sát được |
| --- | --- |
| LLM trích xuất | `charges = ["chống người thi hành công vụ"]` — LLM **có** trích tội danh, không phải trả về mảng rỗng |
| `link_entity` (KG-1) | Không tìm thấy trong `known_crimes` ⇒ trả `None` ⇒ `charges` thành `[]` |
| `build_graph` (KG-2) | `FOREACH (crime IN $charges \| ...)` lặp 0 lần ⇒ **không tạo cạnh `CHARGED_WITH`** |

- **Nguyên nhân:** Lỗi **không nằm ở bước trích xuất LLM** như trực giác hay nghĩ đầu tiên, mà nằm ở **thiết kế ontology: phạm vi KB luật hẹp hơn phạm vi thực tế của KB tin tức**. Cả hai KB đều được thu thập theo chủ đề "ma túy", nhưng KB luật chỉ gồm 13 Điều của Chương XX BLHS + 5 Điều của Luật PCMT, trong khi bài báo này nói về tội *"chống người thi hành công vụ"* (Điều 330 BLHS) — hoàn toàn nằm ngoài KB luật. `link_entity` chỉ liên kết vào danh sách tội danh đã có trong graph, nên tội danh hợp lệ ngoài phạm vi đó bị **âm thầm loại bỏ**. Cùng cơ chế đó, cột `substances` ghi `"ma túy"` (tên gọi chung) thay vì một chất cụ thể, nên cầu nối dự phòng qua `Substance` cũng không khớp `MENTIONS` của bất kỳ khoản luật nào.
  - Hậu quả thiết kế: node `Case` này rơi vào "mồ côi" — có mặt trong KB tin tức nhưng về mặt pháp lý bất khả dụng, và **không có tín hiệu cảnh báo nào** vì `link_entity` trả `None` là hành vi hợp lệ.
- **Đề xuất sửa:** 
  1. *Ghi nhận thay vì âm thầm loại bỏ:* thay `charges = [c for c in ... if c]` bằng cách giữ lại tội danh gốc kèm cờ trạng thái, ví dụ tạo `(k)-[:CHARGED_WITH_OUT_OF_SCOPE]->(:UnmappedCrime {name: $raw})` hoặc ghi property `k.unmapped_charges = ["chống người thi hành công vụ"]`. Khi đó câu hỏi loại "các vụ án trong tin tức ngoài phạm vi điều luật đã nạp" sẽ trả lời được, và tỉ lệ gãy cầu nối trở thành chỉ số đo được.
  2. *Cầu nối dự phòng qua `Substance`:* bài này vẫn có `INVOLVES -> "ma túy"`. Nếu bổ sung node `Substance` cấp **khái niệm chung** ("ma túy", "chất ma túy") mà mọi `Clause` đều `MENTIONS` gián tiếp, thì vẫn trả lời được câu hỏi cấp tổng quát, dù không trả lời được khung phạt cụ thể.
  3. *Giảm ràng buộc danh mục trong prompt:* cho phép LLM tự do ghi tội danh ngoài danh sách kèm chỉ báo `"in_scope": false`, thay vì bắt buộc chọn nguyên văn trong `DANH SÁCH TỘI DANH` — hiện tại chỉ thị "BẮT BUỘC chọn đúng nguyên văn" làm LLM buộc phải bỏ trống hoặc chọn sai để làm vừa lòng ràng buộc.

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

---

### Lỗi E5: LLM lệch với graph — trả lời sai điều luật (Evidence–LLM drift)

- **Hiện tượng:** Ở 2 câu `cross-kb`, GraphRAG **có** dựng được đường đi qua node cầu nối `Crime` sang KB luật, nhưng câu trả lời vẫn chỉ tới **sai điều luật**. Cả hai pipeline đều đọc đúng văn bản luật nhưng ánh xạ sai tội danh sang điều luật.
- **Bằng chứng 1 — Q5**, trích nguyên văn từ `ket_qua_benchmark_kg.txt`:

```
--- Q5 [cross-kb-multi-hop] graph recall=0.80 judge=1 2.24s
Cái Quang Huy bị truy tố về tội vận chuyển trái phép chất ma túy với loại ma túy là MDMA. Với khối lượng MDMA trong vụ này là hơn 9,6kg, khoản áp dụng tương ứng là khoản 4 của Điều 251 Bộ luật Hình sự (BLHS). Khung hình phạt theo khoản này là bị phạt tù 20 năm, tù chung thân hoặc tử hình.
```

Đối chiếu với đáp án chuẩn trong `data/benchmark_kg.json` (mục `gold`):
> *"Cái Quang Huy bị truy tố về tội **vận chuyển** trái phép chất ma túy (**Điều 250 BLHS**) với hơn 9,6kg MDMA và khoản 406g Ketamine; MDMA từ 100 gam trở lên thuộc **khoản 4 Điều 250**: tù 20 năm, tù chung thân hoặc tử hình."*

→ Hệ thống nói **Điều 251**, chuẩn là **Điều 250**. Đây không phải lỗi hình thức: hai điều luật có đoạn khoản 4 gần như giống hệt nhau, nên câu trả lời *nghe* rất thuyết phục nhưng **sai điều**. Kiểm tra lại văn bản gốc trong `data/drug_law/blhs-dieu-250.md`:

```
4. Phạm tội thuộc một trong các trường hợp sau đây, thì bị phạt tù 20 năm, tù chung thân hoặc tử hình:
b) Heroine, Cocaine, Methamphetamine, Amphetamine, MDMA hoặc XLR-11 có khối lượng 100 gam trở lên;
```

- **Bằng chứng 2 — Q4**, trích nguyên văn từ `ket_qua_benchmark_kg.txt`:

```
--- Q4 [cross-kb] graph recall=0.33 judge=1 2.66s
Giang hồ 'Hoàng Nato' bị bắt về hành vi tàng trữ trái phép chất ma túy. Theo Điều 249 Bộ luật Hình sự, hành vi đó có thể bị phạt tù tối đa lên đến 20 năm hoặc tù chung thân nếu thuộc các trường hợp quy định tại khoản 4 của Điều 249.
```

Đáp án chuẩn: *"Dương Minh Tuấn (Hoàng Nato) bị bắt về hành vi **tổ chức sử dụng** trái phép chất ma túy (**Điều 255 BLHS**); khung cao nhất là tù 20 năm hoặc tù chung thân."*

→ Hệ thống dẫn tới *tàng trữ / Điều 249* thay vì *tổ chức sử dụng / Điều 255*. Hệ quả là `must_include = ["tổ chức sử dụng", "Điều 255", "chung thân"]` chỉ khớp đúng 1/3 ⇒ `recall = 0.33`, `judge = 1`.

- **Nguyên nhân:** Lỗi nằm ở **2 bước, một trong số đó là thiết kế ontology**:

  1. *Bước trích xuất (KG-2):* cả 13 Điều của Chương XX BLHS có khoản 4 với cấu trúc gần giống nhau ("tù 20 năm, tù chung thân hoặc tử hình"). Ontology hiện tại mô hình hóa `Crime` chỉ ở mức **tên tội danh**, mà **không mô hình hóa ngưỡng khối lượng** dù chính những con số này quyết định việc chọn khoản nào. Vì vậy `context()` buộc phải dùng heuristic tiêu chí số như `cl.number = 1 OR cl.number = 4 OR EXISTS { MENTIONS chất }` — với khoản 4 thì **mọi điều luật đều hợp lệ về mặt hình thức**, nên Cypher trả về cả Điều 250 lẫn Điều 251 cùng lúc.
  2. *Bước sinh câu trả lời (KG-4):* khi nhiều điều luật cùng thoả điều kiện lọc và lại có cùng câu chữ khoản 4, LLM không có tín hiệu nào để chọn, và **bám theo thứ tự xuất hiện** trong danh sách facts ⇒ chọn nhầm điều luật. Ở Q4, cùng cơ chế đó biến *"tàng trữ"* thành nhãn dán khi tội danh thật là *"tổ chức sử dụng"* — tội danh đúng **đã có trong graph** (Điều 255 được nạp đầy đủ) nhưng bị các tội danh khác của cùng một vụ án "cạnh tranh" và thắng phiếu.
- **Đề xuất sửa:**
  1. *Đưa ngưỡng khối lượng vào graph, không để LLM tự suy luận:* thêm property `threshold_g` trên `Clause` (Điều 250 khoản 4 điểm b = 100 gam) và parse `amount` của cạnh `INVOLVES` thành số (`r.amount_g`). Khi đó Cypher lọc được bằng **phép so sánh số học** `cl.threshold_g <= k.amount_g` thay vì bằng `cl.number = 4`, và chỉ còn đúng một điều luật thỏa điều kiện.
  2. *Phân biệt tội danh ở mức hành vi, không gộp theo vụ án:* nếu một bài báo nêu nhiều tội danh cho cùng một người, lưu tội danh **trên cạnh `INVOLVED_IN`** (đã có sẵn `r.charge`) thay vì chỉ gắn vào `Case`. Khi đó câu hỏi về *"Hoàng Nato"* sẽ bám đúng tội danh gắn với chính người đó, thay vì phải chọn giữa các tội danh của cả vụ.
  3. *Giảm nhiễu ngữ cảnh:* `max_facts` hiện đang nạp cả khoản 1, 2, 4 của mọi điều luật liên quan. Nên ưu tiên khoản có `penalty` khớp với mức án đã biết từ tin tức (ví dụ `r.sentence` chứa "tử hình" ⇒ ưu tiên khoản 4), thay vì đưa hết vào cho LLM tự lọc.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Khi nào Flat RAG là đủ:** Với các bài toán hỏi đáp đơn chặng (single-hop), phạm vi câu trả lời nằm gọn trong một văn bản hoặc một đoạn trích duy nhất (như câu Q1 và Q2, cả Flat và Graph đều đạt recall 1.00 và judge 2), Flat RAG là lựa chọn tối ưu tuyệt đối. Flat RAG tiết kiệm hơn **8.23 lần chi phí indexing** ($0.00112 vs $0.00922), tiết kiệm **5.31 lần chi phí mỗi câu hỏi** ($0.00013 vs $0.00069) và có độ trễ nhanh hơn 40–60% (1.47s vs 2.43s).
> - **Khi nào nên dùng Knowledge Graph (GraphRAG):** Knowledge Graph là bắt buộc khi câu hỏi yêu cầu **kết nối thông tin rải rác xuyên nhiều nguồn dữ liệu (cross-KB, multi-hop)** mà không có một chunk văn bản nào chứa đủ ngữ cảnh. Ở Q3, Flat RAG hoàn toàn tê liệt (recall 0.00, judge 0 — trả lời "Không đủ thông tin"), trong khi GraphRAG đạt độ chính xác hoàn hảo (recall 1.00, judge 2): đây là bằng chứng rõ nhất cho thấy KG thắng *vì cấu trúc*, không phải vì model to hơn. Ở Q6, KG cũng vượt trội (recall 0.67 vs 0.00).
>
> - **Cảnh báo quan trọng — recall cao KHÔNG đồng nghĩa trả lời đúng:** Q4 và Q5 cho thấy điểm yếu còn lại của hướng tiếp cận này. Cả hai pipeline đều **đọc đúng văn bản luật** nhưng vẫn chỉ sai điều (Q5: Điều 251 thay vì Điều 250; Q4: Điều 249 thay vì Điều 255) vì ontology chỉ mô hình hóa tội danh mà chưa mô hình hóa **ngưỡng khối lượng**, và vì `context()` nạp khoản 1 + 2 + 4 của mọi điều luật liên quan rồi để LLM tự chọn. Nói cách khác, KG giải quyết được bài toán *truy xứa* nhưng chưa giải quyết bài toán *suy luận định lượng*. Muốn sửa thì phải đưa ngưỡng khối lượng vào graph và lọc bằng phép so sánh số trong Cypher (xem lỗi E5), chứ không thêm prompt.
>
> **Điểm hòa vốn:** chênh lệch chi phí là khoảng **$0.00056/câu hỏi** ($0.00069 − $0.00013) và **0.96 giây/câu**. Ở quy mô 100 câu hỏi, GraphRAG tốn thêm $0.056 so với Flat RAG — vẫn rất nhỏ. Vì vậy **ranh giới quyết định không nằm ở chi phí mà nằm ở tỉ lệ câu hỏi cross-kb**: nếu trên 50% câu hỏi cần nối 2 KB thì KG đáng tiền, vì mỗi câu đó Flat RAG trả lời "Không đủ thông tin" (recall 0.00) — một câu hỏi sai cũng tốn tiền gọi LLM. Nếu phần lớn câu hỏi là single-hop như Q1–Q2 thì Flat RAG thắng rõ về cả chi phí lẫn độ trễ mà không thiếu gì.

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

Còn lại 4 hạn chế đã phân tích có bằng chứng ở mục 3 và mục 8 của `ONTOLOGY.md`:
- **E1** — 1/20 vụ án rơi vào "mồ côi" vì tội danh nằm ngoài phạm vi 18 điều luật đã nạp, và `link_entity` loại bỏ âm thầm không cảnh báo.
- **E5** — Q4/Q5 trả lời sai điều luật vì thiếu mô hình hóa ngưỡng khối lượng trong ontology.
- **E3** — `Substance` bị phân mảnh theo hoa/thường và tên lóng (`Ketamine`/`ketamine`, `thuốc lắc` ≠ `MDMA`).
- **E4** — `recall` chỉ so khớp chuỗi, nên phạt oan câu trả lời đúng ở Q6.

Ngoài ra, benchmark chỉ gồm 6 câu nên chưa đủ cơ sở để kết luận thống kê về độ trễ; và quy mô KB (18 điều luật, 20 bài báo) còn nhỏ nên chưa thấy rõ chi phí KG có cải thiện hay không khi graph lớn lên.
