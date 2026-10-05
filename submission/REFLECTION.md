# Reflection — Lab 19

**Tên:** Dương Quốc Khánh       
**Cohort:** A20-K4      
**Path đã chạy:** lite      

---

## Câu hỏi (≤ 200 chữ)

> Trên golden set 50 queries, mode nào thắng ở loại query nào (`exact` /
> `paraphrase` / `mixed`), và tại sao? Khi nào bạn **không** dùng hybrid
> (i.e. khi nào pure BM25 hoặc pure vector là lựa chọn đúng)?

Precision@10 trên 50 query: hybrid (RRF k=60) 78.6% thắng tổng thể, BM25 77.8%, semantic 73.2%. Theo loại query: `exact` BM25 và hybrid cùng 96.7% (semantic 88.7%); `mixed` hybrid thắng 100% so với 97.0% (BM25) và 98.5% (semantic), vì hai retriever bổ sung cho nhau. Với `paraphrase`, semantic lại thấp nhất (24.0%, BM25 33.3%, hybrid 32.0%). Tôi dùng fastembed bge-small-en-v1.5, model chỉ tiếng Anh nên yếu với paraphrase tiếng Việt; vector không thắng paraphrase trong lab này.

Không dùng hybrid khi: query là mã, ID hoặc từ khóa chính xác và cần độ trễ thấp, hoặc embedding yếu với ngôn ngữ đó, thì BM25 thuần đủ tốt và rẻ hơn. Khi query là câu tự nhiên diễn đạt lại và có model đa ngôn ngữ mạnh như bge-m3, vector thuần hợp lý hơn. Hybrid còn tốn hai retriever cộng bước fusion, nên P99 cao hơn nếu không warm-up.

---

## Điều ngạc nhiên nhất khi làm lab này

P99 của hybrid giảm từ 102.1 ms (cold start) xuống 22.3 ms chỉ nhờ thêm vòng warm-up. Ngoài ra, PIT join ban đầu trả về 2 dòng thay vì 3 mà không báo lỗi, vì timestamp của feature muộn hơn timestamp của entity.

---

## Bonus challenge

- [ ] Đã làm bonus (xem `bonus/`)
- [ ] Pair work với: _<tên đồng đội nếu có>_
