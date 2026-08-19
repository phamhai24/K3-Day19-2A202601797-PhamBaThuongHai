# Suy Ngẫm & Kế Hoạch Dự Án (Reflection & Action Plan)

### 1. Mapping Bài Giảng vào Code

| Khái niệm bài giảng | Module | Hàm/Code tương ứng | Quan sát thực tế |
|---------------------|--------|---------------------|-----------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Batch 10 chunks/call tiết kiệm API cost nhưng JSON parsing phức tạp hơn |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Guard loại bỏ ~30% triple noise từ LLM extraction |
| **Bulk Cypher UNWIND** | Module 2 | `bulk_insert_nodes_simple()`, `bulk_insert_edges_simple()` | UNWIND 500 rows/batch: 10x nhanh hơn single MERGE |
| **Vector ANN + Lexical Guard** | Module 3 | `resolve_entities()`, `lex_key()` | Cosine threshold 0.88 + lexical guard loại bỏ false merges |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `SUPER_NODE_DEGREE=100` | BFS với cap 50 edges/supernode: latency <4s thay vì >30s |
| **LLM-as-a-Judge (1-5)** | Module 5 | `judge_answer()`, `run_evaluation()` | 3 metrics: comprehensiveness, faithfulness, multi_hop_reasoning |

### 2. Quá Trình Debugging & Bài Học

**Lỗi kỹ thuật phức tạp nhất — Unicode Hell trên Windows:**
- **Vấn đề:** Python 3.11 trên Windows dùng `cp1252` encoding mặc định cho stdout, không encode được emoji và tiếng Việt có dấu → crash bất cứ khi nào print ra terminal.
- **Giải pháp:** `sys.stdout.reconfigure(encoding='utf-8')` ở đầu mỗi script; dùng `[System.IO.File]::ReadAllText(..., UTF8)` trong PowerShell; set `$env:PYTHONIOENCODING="utf-8"` trước khi chạy script.
- **Bài học:** Luôn test encoding compatibility trước khi viết bất kỳ print statement nào có non-ASCII chars.

**Lỗi 2 — APOC không available trên AuraDB Free:**
- **Vấn đề:** `apoc.merge.relationship()` cho dynamic relation types được dùng trong notebook gốc nhưng AuraDB Free tier không hỗ trợ.
- **Giải pháp:** Group triples by relation type → dùng Python f-string để generate static Cypher: `MERGE (src)-[r:{rel_type} {source_chunk_id: ...}]->(dst)`. Vẫn đảm bảo idempotency.

**Lỗi 3 — Groq API Rate Limit & Model availability:**
- **Vấn đề:** API key có trong .env nhưng account không có access đến model LLaMA, đồng thời dễ gặp Rate Limit khi extraction.
- **Giải pháp:** Liệt kê models qua API, tự động fallback xuống `groq/compound-mini`. Xây dựng logic backoff (chờ 20s và retry) để tránh crash pipeline.

### 3. Kế Hoạch Áp Dụng vào Dự Án Thực Tế (Action Plan)

**Dự án thực tế:** Hệ thống Enterprise Knowledge Management cho công ty phần mềm 200 nhân viên.

**Bài toán có cần GraphRAG không?**
- Câu hỏi multi-hop: "Ai trong team đã làm việc với vendor X và kết quả ra sao?" → **Cần GraphRAG**
- Câu hỏi factoid: "Chính sách nghỉ phép 2024 là gì?" → **Flat RAG đủ**
- **Kết luận:** Hybrid RAG với Router — phân loại câu hỏi, dispatch tương ứng.

**Cấu trúc Node/Relation dự kiến:**
```text
Nodes:
  Person (employee), Project, Vendor, Technology, Document, Meeting, Policy

Relations:
  Person  -WORKED_ON->    Project
  Person  -ATTENDED->     Meeting
  Person  -AUTHORED->     Document
  Project -USES->         Technology
  Project -CONTRACTED->   Vendor
  Meeting -DISCUSSED->    Project
  Policy  -GOVERNS->      Team/Department
```

**Chiến lược Entity Resolution:**
- Manual aliases: nickname → full name (e.g., "Nam" → "Nguyễn Văn Nam")
- Cosine threshold: 0.90 (domain hẹp, ít ambiguity)
- Temporal dedup: Person với cùng tên qua nhiều năm → dùng `employee_id` là canonical key

**Chiến lược Super-node:**
- Meeting nodes (nhiều attendees) → cap at 30 edges mới nhất
- Technology nodes (e.g., "Python") → cap at 20, filter by project relevance score
- Policy nodes → không cap (thường có ít edges)

---

## ✅ TỰ ĐÁNH GIÁ

| Tiêu chí | Điểm tự chấm (1-5) | Ghi chú |
|----------|--------------------|---------| 
| Mức độ hiểu bài giảng GraphRAG | 4/5 | Hiểu rõ pipeline, còn cần thực hành thêm về Community Detection |
| Khả năng kiểm soát AI Coding Agent | 4/5 | Từ chối được 3 đề xuất không phù hợp và giải thích lý do rõ ràng |
| Chất lượng đồ thị tri thức xây dựng | 3/5 | Dataset HackerNoon descriptions ngắn → ít triple; cần full article content |
| Khả năng phân tích và debug hệ thống | 5/5 | Resolve được Unicode, APOC, model availability và rate limit issues |
