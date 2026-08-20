# Reflection — Lab 19

**Tên:** _Trần Bá Lợi_
**Cohort:** _A20-K4_
**Path đã chạy:** _lite_

---

## Câu hỏi (≤ 200 chữ)

> Trên golden set 50 queries, mode nào thắng ở loại query nào (`exact` /
> `paraphrase` / `mixed`), và tại sao? Khi nào bạn **không** dùng hybrid
> (i.e. khi nào pure BM25 hoặc pure vector là lựa chọn đúng)?

Trung bình trên 50 query: hybrid (78.6%) > keyword (77.8%) > semantic (73.2%).
Nhưng con số trung bình che mất khác biệt theo loại query:

- **exact** (15 câu): keyword và hybrid đồng hạng (96.7%), semantic thấp hơn
  (88.7%) — khi query trùng gần như y hệt từ khóa trong doc, BM25 match trực
  tiếp là đủ, không cần vector.
- **paraphrase** (15 câu): keyword (33.3%) > hybrid (32.0%) > semantic
  (24.0%) — ngược với lý thuyết "vector thắng câu diễn đạt lại". Nguyên nhân:
  model embedding mặc định (`bge-small-en-v1.5`) là model tiếng Anh, không
  hiểu tốt paraphrase tiếng Việt, nên phần vector kéo điểm hybrid xuống thay
  vì giúp ích.
- **mixed** (20 câu): hybrid thắng rõ nhất (100%) so với semantic (98.5%) và
  keyword (97%) — đây là chỗ RRF phát huy tác dụng thật, khi tín hiệu từ khóa
  và ngữ nghĩa bổ sung cho nhau.

**Không nên dùng hybrid** khi: (1) query chủ yếu là exact-match (mã sản phẩm,
tên riêng, số liệu) — pure BM25 đã đạt trần và rẻ hơn; (2) embedding model
không phù hợp ngôn ngữ/domain của corpus — lúc đó vector là nhiễu, cần đổi
`EMBEDDING_BACKEND` (vd. `multilingual`/`bge-m3`) trước khi thêm hybrid,
không phải thêm hybrid để che lấp model yếu.

---

## Điều ngạc nhiên nhất khi làm lab này

Semantic search lại **thua** keyword search trên đúng loại query mà nó lẽ ra
phải mạnh nhất (paraphrase). Lý do không phải RRF sai mà là giả định ngầm bị
phá: "vector luôn thắng câu diễn đạt lại" chỉ đúng nếu embedding model hiểu
đúng ngôn ngữ của corpus — model mặc định của path lite là tiếng Anh, dùng
cho corpus tiếng Việt.

---

## Bonus challenge

- [ ] Đã làm bonus (xem `bonus/`)
- [ ] Pair work với: _<tên đồng đội nếu có>_
