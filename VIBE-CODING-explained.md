# Vibe Coding — Bản giải thích chi tiết (kèm dịch từng prompt tiếng Anh)

> Đây là bản sao của [VIBE-CODING.md](VIBE-CODING.md), giữ nguyên nội dung gốc,
> nhưng mỗi đoạn prompt tiếng Anh đều được dịch + giải thích chi tiết ngay bên dưới
> (đánh dấu bằng khối "📖 Giải thích"). Đọc file này nếu bạn thấy các ví dụ prompt
> tiếng Anh trong bản gốc khó hiểu.

## Vibe coding là gì?

**Vibe coding** (Andrej Karpathy, 02/2025) — bạn để LLM viết phần lớn code,
còn bạn đảm nhận vai *architect* (kiến trúc sư) và *reviewer* (người review):
mô tả intent (ý định/yêu cầu) → review diff (xem lại phần code thay đổi)
→ accept (chấp nhận) hoặc reject (từ chối). Bạn không gõ từng dòng `for`
loop; bạn ép spec (đặc tả) rõ ràng và đảm bảo không có bug ngầm trong diff
trả về.

Vibe coding **không phải** là "copy-paste từ ChatGPT". Vibe coding là một
*workflow* (quy trình làm việc):

```
   intent (spec)          → ý định / yêu cầu, viết dưới dạng đặc tả
      ↓
   prompt LLM              → gửi đặc tả đó cho AI dưới dạng prompt
      ↓
   review diff (không skip!)  → đọc kỹ phần code AI vừa thay đổi
      ↓
   run + verify             → chạy thử + kiểm chứng kết quả
      ↓
   commit hoặc rollback     → lưu lại nếu đúng, hoàn tác nếu sai
```

Bỏ qua bất kỳ bước nào → vibe coding biến thành "gambling with code"
(cờ bạc với code — tức là chấp nhận rủi ro mù quáng).

---

## 2 phong cách kỷ luật: SDD và TDD

### Spec-Driven Development (SDD) — Phát triển hướng đặc tả

> Viết **spec** (đặc tả) trước, code sau. Spec là hợp đồng (contract) giữa
> bạn và LLM — giống như bạn đưa cho AI một bản yêu cầu kỹ thuật rõ ràng
> thay vì nói chung chung.

**Spec đầy đủ** thường gồm:
- *Inputs (đầu vào):* tên + kiểu dữ liệu + ràng buộc của mỗi tham số
- *Output (đầu ra):* hình dạng dữ liệu + kiểu + các bất biến (invariants — điều luôn đúng)
- *Behavior (hành vi):* trường hợp biên (edge case), lỗi, side-effects (tác dụng phụ)
- *Constraints (ràng buộc):* giới hạn thời gian phản hồi (latency budget), giới hạn bộ nhớ, thư viện cấm dùng

Ví dụ prompt gốc:
```
Function: search_hybrid(query: str, top_k: int = 10, rrf_k: int = 60)
Inputs:
  - query: non-empty string, max 500 chars
  - top_k: 1..100
  - rrf_k: 1..200
Output:
  - list[SearchHit] sorted by RRF score desc, length == min(top_k, |union of inputs|)
Behavior:
  - Empty corpus → []
  - rank_r is 1-based (first result = rank 1)
  - Score formula: sum_r 1/(rrf_k + rank_r(d))
Constraints:
  - P99 latency < 50ms server-side
  - No external API calls (in-memory only)
```

📖 **Giải thích chi tiết:**
- `Function: search_hybrid(query: str, top_k: int = 10, rrf_k: int = 60)`
  — Khai báo hàm cần viết: tên hàm `search_hybrid`, nhận 3 tham số:
  `query` kiểu chuỗi (str), `top_k` kiểu số nguyên mặc định 10, `rrf_k`
  kiểu số nguyên mặc định 60.
- **Inputs:**
  - `query: non-empty string, max 500 chars` → chuỗi tìm kiếm không được
    rỗng, tối đa 500 ký tự.
  - `top_k: 1..100` → số kết quả trả về, phải nằm trong khoảng 1 đến 100.
  - `rrf_k: 1..200` → hằng số dùng trong công thức RRF (Reciprocal Rank
    Fusion — kỹ thuật gộp kết quả từ nhiều bộ tìm kiếm), giá trị từ 1–200.
- **Output:**
  - `list[SearchHit] sorted by RRF score desc` → trả về danh sách đối
    tượng `SearchHit`, sắp xếp theo điểm RRF giảm dần.
  - `length == min(top_k, |union of inputs|)` → độ dài danh sách bằng
    giá trị nhỏ hơn giữa `top_k` và tổng số kết quả duy nhất (hợp của các
    nguồn tìm kiếm).
- **Behavior:**
  - `Empty corpus → []` → nếu kho dữ liệu (corpus) rỗng, trả về danh sách
    rỗng, không lỗi.
  - `rank_r is 1-based (first result = rank 1)` → thứ hạng bắt đầu từ 1
    (kết quả đầu tiên có rank = 1), không phải từ 0. Đây là chi tiết cực
    kỳ dễ nhầm — AI hay mặc định bắt đầu từ 0.
  - `Score formula: sum_r 1/(rrf_k + rank_r(d))` → công thức tính điểm:
    tổng của 1/(rrf_k + thứ hạng) trên tất cả các nguồn xếp hạng chứa tài
    liệu `d`.
- **Constraints:**
  - `P99 latency < 50ms server-side` → 99% số request phải phản hồi dưới
    50 mili-giây, đo phía server.
  - `No external API calls (in-memory only)` → không được gọi API bên
    ngoài, chỉ xử lý trong bộ nhớ (không gọi mạng, không gọi DB ngoài).

→ **Vì sao viết chi tiết vậy?** Vì nếu spec mơ hồ (ví dụ chỉ nói "viết
hàm tìm kiếm hybrid"), AI sẽ tự đoán rank bắt đầu từ 0 hay 1, tự đoán công
thức RRF — dễ sai mà không báo lỗi (silent bug), rất khó debug sau này.

LLM viết code khớp spec. Bạn review diff để verify (xác minh) từng dòng
implement đúng spec. Spec mơ hồ → code mơ hồ → debug 1 giờ.

### Test-Driven Development (TDD) cho LLM era — Phát triển hướng kiểm thử

> Viết **test** (bài kiểm thử) trước, code sau. Test là spec dạng máy
> chấm — tức là thay vì mô tả bằng lời, bạn mô tả bằng code kiểm tra tự
> động.

```
Vòng 1: "Write a pytest test for search_hybrid that asserts:
         - hybrid('exact match query') beats keyword on Precision@5
         - hybrid('paraphrase query') beats vector on Precision@5
         - hybrid('') raises ValueError
         Don't write the implementation yet — only the test."
```

📖 **Giải thích chi tiết (dịch nguyên văn prompt):**
"Viết một bài test bằng pytest cho hàm `search_hybrid`, kiểm tra rằng:
- Khi tìm với câu `'exact match query'` (câu khớp chính xác), kết quả của
  chế độ hybrid phải tốt hơn chế độ keyword (tìm theo từ khóa), đo bằng
  chỉ số Precision@5 (độ chính xác trong 5 kết quả đầu).
- Khi tìm với câu `'paraphrase query'` (câu diễn đạt lại ý, không trùng
  từ khóa), kết quả hybrid phải tốt hơn chế độ vector (tìm theo ngữ
  nghĩa/embedding), cũng đo bằng Precision@5.
- Khi gọi với chuỗi rỗng `''`, hàm phải raise (ném ra) lỗi `ValueError`.
- **Đừng viết phần implementation (code xử lý logic) vội — chỉ viết
  test thôi.**"

→ **Vì sao yêu cầu "đừng viết implementation"?** Để tách bạch: bước 1 chỉ
tạo ra "đề bài" (test), bước 2 mới "làm bài" (code). Nếu để AI viết cả
hai cùng lúc, nó có thể viết code trước rồi bịa test khớp với code đó
(test vô nghĩa vì luôn pass).

Test pass-by-construction (test chắc chắn fail lúc mới viết, vì logic
đằng sau nó chưa tồn tại) — bạn run, test fail vì chưa có code. Sau đó:

```
Vòng 2: "Now implement search_hybrid such that the test passes."
```

📖 **Giải thích:** "Bây giờ hãy viết code cho hàm `search_hybrid` sao
cho bài test (đã viết ở vòng 1) chạy pass (thành công)."

Vibe code phần implement, nhưng **test là không đổi** (bạn không sửa
test theo code — nếu code sai thì sửa code, không phải sửa test cho dễ
pass). Nếu test sai (ví dụ assert sai logic), bạn phát hiện ngay từ vòng
1, không phải sau khi deploy.

TDD đặc biệt mạnh với vibe coding vì LLM hay hallucinate (bịa/ảo giác)
edge cases (trường hợp biên) — tests làm bộ chống hallucination.

---

## Khi nào vibe code, khi nào tự nghĩ?

| Vibe code thoải mái | Tự nghĩ kỹ trước khi prompt |
|---|---|
| API route boilerplate (khung code lặp lại cho route API, ví dụ FastAPI, Express, …) | Lựa chọn algorithm (thuật toán) / data structure (cấu trúc dữ liệu) cốt lõi |
| Pydantic / Zod / Typescript schemas (khai báo cấu trúc dữ liệu) | Concurrency model — mô hình xử lý đồng thời (lock: khóa tuần tự vs lock-free: không khóa vs CAS: Compare-And-Swap, một kỹ thuật đồng bộ hóa không khóa) |
| Test scaffolding (khung sườn test: pytest fixtures — dữ liệu mẫu dùng lại, mocks — giả lập đối tượng) | Failure semantics — ngữ nghĩa khi lỗi xảy ra (retry: thử lại, idempotency: gọi nhiều lần vẫn ra kết quả như gọi 1 lần) |
| Config files (file cấu hình: YAML, JSON, env) | Schema migration (di trú cấu trúc dữ liệu) / backward compat (tương thích ngược) |
| README skeleton (khung sườn README), docstrings (chú thích mô tả hàm) | Security boundary — ranh giới bảo mật (auth: xác thực, sandboxing: cô lập môi trường chạy) |
| Synthetic data generators / fixtures (bộ sinh dữ liệu giả để test) | Performance budget tradeoffs (đánh đổi giữa các giới hạn hiệu năng) |
| Error handling cho I/O (xử lý lỗi vào/ra dữ liệu — khung try/except lặp lại) | Cache invalidation strategy (chiến lược làm mất hiệu lực cache) |
| Refactor "đổi tên field X → Y" trên cả repo | Architecture (kiến trúc tổng thể: vector vs graph, monolith — 1 khối vs micro — chia nhỏ dịch vụ) |

**Quy tắc đơn giản:** nếu bug sẽ là *silent regression* (hệ thống chạy
nhưng kém hơn, không lỗi rõ) thay vì *loud failure* (lỗi rõ ràng: có
exception, test fail), đó là **think-hard zone** (vùng cần tự nghĩ kỹ).
Đừng để LLM tự quyết.

---

## 5 prompt patterns universal — 5 khuôn mẫu prompt dùng chung

### 1. Specs in, code out — Đưa vào đặc tả, nhận ra code

> Càng narrow (càng thu hẹp/cụ thể) → cleaner diff (thay đổi code gọn
> hơn), ít iterate (ít phải lặp lại/sửa qua sửa lại) hơn.

```
[VAGUE — DON'T]
"Write a search API"

[NARROW — DO]
"FastAPI GET /search?q=str&mode=Literal['kw','sem','hybrid']
returning SearchResponse(query: str, mode: str, latency_ms: float, hits: list[Hit]).
Use Searcher class from app/search.py — call .search(q, mode=mode, top_k=10).
Measure latency_ms with time.perf_counter() server-side, exclude network."
```

📖 **Giải thích chi tiết:**
- `[VAGUE — DON'T]` (Mơ hồ — ĐỪNG LÀM VẬY): `"Write a search API"` =
  "Viết một API tìm kiếm" — quá chung chung, AI không biết dùng framework
  gì, endpoint nào, trả về gì.
- `[NARROW — DO]` (Cụ thể — NÊN LÀM VẬY), dịch từng phần:
  - `"FastAPI GET /search?q=str&mode=Literal['kw','sem','hybrid']"` →
    "Dùng framework FastAPI, tạo endpoint GET tại đường dẫn `/search`,
    nhận query param `q` kiểu chuỗi và `mode` chỉ được nhận 1 trong 3
    giá trị: `'kw'` (keyword — từ khóa), `'sem'` (semantic — ngữ nghĩa),
    `'hybrid'` (kết hợp cả hai)."
  - `"returning SearchResponse(query: str, mode: str, latency_ms: float, hits: list[Hit])"`
    → "Trả về một object `SearchResponse` gồm 4 trường: `query` (câu
    tìm), `mode` (chế độ đã dùng), `latency_ms` (thời gian xử lý tính
    bằng mili-giây), `hits` (danh sách kết quả kiểu `Hit`)."
  - `"Use Searcher class from app/search.py — call .search(q, mode=mode, top_k=10)"`
    → "Dùng class `Searcher` có sẵn trong file `app/search.py`, gọi
    phương thức `.search(q, mode=mode, top_k=10)` của nó" — tức là chỉ
    rõ luôn code có sẵn nào cần tái sử dụng, tránh AI viết lại từ đầu.
  - `"Measure latency_ms with time.perf_counter() server-side, exclude network"`
    → "Đo `latency_ms` bằng hàm `time.perf_counter()` của Python, đo ở
    phía server, không tính thời gian truyền mạng."

→ Prompt "NARROW" nói rõ: framework, đường dẫn, tham số, kiểu trả về,
class có sẵn cần dùng, và cách đo latency — AI gần như không còn chỗ để
"đoán mò".

### 2. Validate trước khi generate — Kiểm chứng trước khi để AI viết code

> Với công thức / thuật toán: hỏi AI giải thích, cross-check (đối chiếu
> lại), mới nhờ implement (triển khai).

```
Step 1: "Explain Reciprocal Rank Fusion. Formula? Rank 0-based or 1-based? k=?"
Step 3: "Implement search_hybrid(...) per the formula above. rank is 1-based, k=60."
```

📖 **Giải thích chi tiết:**
- `Step 1: "Explain Reciprocal Rank Fusion. Formula? Rank 0-based or 1-based? k=?"`
  → "Hãy giải thích thuật toán Reciprocal Rank Fusion (RRF — kỹ thuật gộp
  kết quả xếp hạng từ nhiều nguồn tìm kiếm). Công thức là gì? Thứ hạng
  tính từ 0 hay từ 1? Hằng số k bằng bao nhiêu?" — bạn hỏi AI trước để nó
  trình bày công thức, rồi **bạn tự đối chiếu** với tài liệu gốc/paper
  xem AI trả lời đúng không (Step 2, không có code mẫu vì bạn tự làm).
- `Step 3: "Implement search_hybrid(...) per the formula above. rank is 1-based, k=60."`
  → "Bây giờ hãy triển khai hàm `search_hybrid(...)` theo đúng công thức
  ở trên. Thứ hạng tính từ 1, hằng số k = 60." — sau khi đã xác nhận công
  thức đúng, bạn mới ép AI code theo đúng công thức đó, tránh nó tự bịa.

Nhiều AI hallucination (ảo giác/bịa) viết `1/rank` thay vì
`1/(k+rank)`, hoặc rank 0-based — silent regression khó debug (lỗi âm
thầm, chương trình vẫn chạy nhưng ra điểm số sai).

### 3. Tests trước, code sau (TDD)

> Test là spec dạng máy chấm. Viết test trước → code phải pass test.

```
"Write a pytest test that asserts X. Don't write implementation yet."
```

📖 **Giải thích:** "Viết một bài test bằng pytest kiểm tra điều kiện X.
Đừng viết code xử lý logic (implementation) vội." — X ở đây là chỗ trống
bạn tự điền hành vi cụ thể cần kiểm tra.

Sau khi test viết đúng (run pass-by-construction = fail, tức là chạy thử
thì test phải fail vì chưa có code thật), prompt implement (yêu cầu viết
code triển khai).

### 4. Minimal repro → expand — Làm bản tối giản trước, rồi mở rộng dần

> Đừng yêu cầu LLM viết toàn bộ feature trong 1 prompt. Build
> incrementally (xây dựng từng bước nhỏ).

```
Step A: "Write minimal X with 1 feature."
Step B: "Run + verify."
Step C: "Now extend X to handle case Y."
Step D: "Now wrap in benchmark/test loop."
```

📖 **Giải thích từng bước:**
- `Step A: "Write minimal X with 1 feature."` → "Viết phiên bản tối
  giản của X, chỉ với 1 tính năng duy nhất."
- `Step B: "Run + verify."` → "Chạy thử và kiểm chứng kết quả" (bạn tự
  làm bước này, không phải prompt AI).
- `Step C: "Now extend X to handle case Y."` → "Bây giờ mở rộng X để xử
  lý thêm trường hợp Y."
- `Step D: "Now wrap in benchmark/test loop."` → "Bây giờ bọc X trong
  một vòng lặp benchmark/test (đo hiệu năng hoặc kiểm thử tự động)."

LLM không hallucinate khi context đã có working baseline (một phiên bản
chạy được làm nền tảng) — vì mỗi bước AI chỉ cần sửa/thêm trên cái đã
chạy đúng, thay vì tưởng tượng ra toàn bộ hệ thống cùng lúc.

### 5. Plan → code → review loop — Vòng lặp lên kế hoạch → viết code → review

> 3 vòng: AI propose (đề xuất) 3 approaches (phương án) → bạn pick
> (chọn) → AI implement (triển khai) → bạn review.

```
Vòng 1: "Propose 3 approaches to do X. Compare on (cost, complexity, scalability)."
Vòng 2: Bạn pick 1: "Use approach #2 because Z."
Vòng 3: "Implement approach #2 + write test."
Vòng 4: Bạn review diff line-by-line.
```

📖 **Giải thích:**
- `"Propose 3 approaches to do X. Compare on (cost, complexity, scalability)."`
  → "Đề xuất 3 phương án để làm X. So sánh chúng theo 3 tiêu chí: chi
  phí (cost), độ phức tạp (complexity), khả năng mở rộng (scalability)."
- `"Use approach #2 because Z."` → "Dùng phương án số 2, vì lý do Z"
  (bạn tự điền lý do bạn chọn).
- `"Implement approach #2 + write test."` → "Triển khai phương án số 2
  và viết test kèm theo."

Đừng skip (bỏ qua) vòng 1 — bạn sẽ stuck (bị kẹt) trong local optimum
(giải pháp tối ưu cục bộ, không phải tốt nhất) mà LLM nghĩ ra đầu tiên.

---

## CLI tool recommendations — Gợi ý công cụ CLI

Lab này khuyến khích **CLI vibe-coding** (vibe coding qua dòng lệnh) —
git-native (thân thiện với git), terminal-friendly (làm việc tốt trong
terminal), review diff dễ. Ba CLI tool nổi bật 2026:

| Tool | Best at (mạnh nhất ở) | Weak at (yếu ở) |
|---|---|---|
| **Claude Code** (Anthropic) | Multi-file plans (kế hoạch nhiều file), careful edits (sửa cẩn thận), longer reasoning (suy luận sâu hơn), in-terminal TodoWrite + plan mode (chế độ lập kế hoạch trong terminal) | Slower cho 1-line fixes (chậm với sửa lỗi 1 dòng); cần Anthropic API key hoặc subscription (gói thuê bao) |
| **Codex CLI** (OpenAI) | Fast iteration (lặp nhanh), tích hợp chặt với GPT/o1 family, agent mode chạy command thực (chế độ agent thực thi lệnh thật) | Mới hơn, ecosystem (hệ sinh thái) còn đang phát triển; cần OpenAI key |
| **OpenCode** (open-source) | Terminal-first TUI (giao diện terminal), multi-provider — hỗ trợ nhiều nhà cung cấp AI (Anthropic/OpenAI/local Ollama), không bị khóa vào 1 nhà cung cấp (no vendor lock-in) | Cộng đồng nhỏ hơn Claude Code; cần config provider (cấu hình nhà cung cấp AI) lần đầu |

**Why CLI over IDE?** ("Tại sao dùng CLI thay vì IDE?") Trong terminal
bạn dễ:
- Review diff (`git diff`) trước khi accept
- Pipe output qua tools khác (`benchmark.py | grep PASS`) — tức là nối
  kết quả lệnh này làm đầu vào cho lệnh khác
- Reproduce (tái tạo lại) y hệt trên server / CI / pair programming
  session (phiên lập trình cặp đôi)
- Commit + push + revert (hoàn tác) mà không rời context

**Project conventions file** (file quy ước dự án) — commit 1 file ở repo
root để CLI tool tự đọc + respect (tuân theo), giảm prompt boilerplate
(giảm phải lặp lại thông tin nền trong mỗi prompt):
- `CLAUDE.md` — Claude Code
- `AGENTS.md` — Codex CLI, OpenCode (de-facto standard 2025+ — tiêu chuẩn
  ngầm định từ 2025 trở đi)

Đa số CLI tool đọc fallback (dự phòng) tới `AGENTS.md` nếu không có file
riêng, nên 1 file `AGENTS.md` thường đủ cho cả 3 tool. Không cần
duplicate (trùng lặp).

---

## 3 anti-patterns phổ biến — 3 kiểu làm sai thường gặp

### 1. Hỏi AI quyết định kiến trúc

❌ `"Which embedding model should I use?"`
📖 Dịch: "Tôi nên dùng mô hình embedding nào?" — quá chung chung.
→ AI pick default trong training data (chọn đại một cái phổ biến nó từng
thấy khi huấn luyện), không biết corpus (kho dữ liệu) của bạn.

✅ 
```
"I have a 1M-doc Vietnamese corpus, GPU=A10, latency budget=20ms. List 3
candidate embedding models with (MTEB-vi score, dim, RAM/1M vecs, cost).
Recommend top 1, explain why."
```
📖 Dịch: "Tôi có kho dữ liệu tiếng Việt 1 triệu tài liệu, dùng GPU loại
A10, giới hạn thời gian phản hồi 20ms. Hãy liệt kê 3 mô hình embedding
ứng viên kèm theo (điểm MTEB-vi — bộ benchmark đánh giá embedding tiếng
Việt, số chiều vector `dim`, dung lượng RAM cần cho 1 triệu vector, chi
phí). Đề xuất 1 mô hình tốt nhất, giải thích lý do."

→ Câu này cho AI đủ dữ liệu thật (quy mô, phần cứng, ràng buộc thời gian)
để đưa ra khuyến nghị có căn cứ, thay vì đoán mò.

### 2. Generate-and-trust không test — Sinh code rồi tin luôn, không test

❌ Accept (chấp nhận) AI-written code → commit → push → discover bug in
prod (phát hiện lỗi ở môi trường production/thật).

✅ AI generate (sinh code) → bạn run test → test pass → review diff →
commit. Nếu chưa có test, viết test trước (pattern #3 ở trên).

### 3. "Make it faster" không có số — Yêu cầu tối ưu nhưng không có con số cụ thể

❌ `"Make this latency faster"`
📖 Dịch: "Làm cho độ trễ này nhanh hơn."
→ AI optimize (tối ưu) ngẫu nhiên, có thể slower (chậm hơn) — vì không
biết mục tiêu cụ thể là gì, đang chậm ở đâu.

✅ 
```
"P99 hiện tại = 87ms (measured by `<command>`). Target < 50ms.
Profile shows 60% time in `<function>`. Suggest 3 optimizations with
expected speedup."
```
📖 Dịch: "Chỉ số P99 (độ trễ của 99% request) hiện tại là 87ms (đo bằng
lệnh `<command>` — bạn điền tên lệnh thật). Mục tiêu: dưới 50ms. Kết quả
profiling (đo hiệu năng chi tiết) cho thấy 60% thời gian nằm ở hàm
`<function>` (bạn điền tên hàm thật). Hãy đề xuất 3 cách tối ưu, kèm mức
tăng tốc dự kiến (expected speedup) cho mỗi cách."

→ Có số đo cụ thể (87ms, 50ms, 60%, tên hàm) giúp AI tối ưu đúng chỗ,
thay vì sửa lung tung.

### 4. Bonus — prompt thiếu context (thiếu ngữ cảnh)

❌ `"Fix this bug"` → "Sửa lỗi này đi" — không nói lỗi gì, ở đâu.

✅ Paste (dán) exact error message (thông báo lỗi chính xác) + expected
output (kết quả mong muốn) + minimal repro (đoạn code tối giản để tái
hiện lỗi) + relevant file paths (đường dẫn file liên quan) + last commit
that worked (commit gần nhất còn chạy đúng). Mơ hồ in → mơ hồ out (đầu
vào mơ hồ thì đầu ra cũng mơ hồ).

---

## Workflow điển hình cho 1 task

```
1. Đọc / viết spec (5 phút)         → bạn nghĩ
2. Plan: think-hard zone? (1 phút)  → bạn nghĩ
3. Prompt với spec rõ                → AI sinh
4. Review diff line-by-line          → bạn xác minh
5. Run test / benchmark              → máy verify
6. Commit hoặc rollback              → bạn quyết định
```

Không skip step 4 và 5. Đó là chỗ vibe coding fail-soft (thất bại "mềm",
tức là sai nhưng vẫn phát hiện được) thay vì fail-loud (thất bại "ồn",
tức là sai và biết ngay). Thật ra ý ở đây là: **nếu skip bước 4–5, bạn sẽ
không phát hiện lỗi âm thầm** — nên đừng skip.

---

## Đọc thêm

- Andrej Karpathy — "Vibe coding" tweet (02/2025)
- Simon Willison — "Vibe coding is here, and it's pretty cool" (02/2025)
- Anthropic — "Effective coding with Claude" docs
- Geoffrey Litt — "Malleable software" essay (2024) — context (bối cảnh)
  cho tại sao vibe coding work (hiệu quả)

Tốt — giờ áp dụng những patterns này khi làm Lab 19. Đọc TODO comment
trong từng notebook, viết spec rõ trong đầu, prompt AI, review, run,
commit.
