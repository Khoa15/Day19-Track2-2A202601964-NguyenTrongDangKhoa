# Reflection — Lab 19

**Tên:** Nguyen Trong Dang Khoa
**Cohort:** A20-K1
**Path đã chạy:** lite

---

## Câu hỏi (≤ 200 chữ)

> Trên golden set 50 queries, mode nào thắng ở loại query nào (`exact` /
> `paraphrase` / `mixed`), và tại sao? Khi nào bạn **không** dùng hybrid
> (i.e. khi nào pure BM25 hoặc pure vector là lựa chọn đúng)?

**Kết quả benchmark:**
- **exact** (BM25 96.7%): BM25 thắng — query chứa từ kỹ thuật verbatim trong corpus, keyword matching đủ mạnh.
- **paraphrase** (semantic 24%): Semantic thắng nhưng overall thấp — model `bge-small-en` English-trained yếu trên tiếng Việt paraphrase.
- **mixed** (hybrid 100%): Hybrid thắng tuyệt đối — RRF kết hợp cả semantic intent và exact terms.

**Hybrid thắng trung bình** (78.6% > 77.8% keyword > 73.2% semantic) nhờ robust trên mọi loại query.

**Không dùng hybrid khi:**
1. Latency cực kỳ quan trọng và BM25 đã đủ chất lượng (BM25 P99 chỉ ~4ms)
2. Corpus nhỏ, hybrid overhead (RRF fusion) không đáng
3. Tất cả queries đều là exact term matches — BM25 đủ
4. Embedding model không phù hợp ngôn ngữ — hybrid kém hơn pure BM25

---

## Điều ngạc nhiên nhất khi làm lab này

Semantic cache ở ngưỡng 0.75 (con số AWS công bố) và để lọt 36% câu trả lời sai trên corpus này. Đây không phải con số an toàn để copy-paste. Và cross-tenant leak xảy ra hoàn toàn im lặng, không exception, không log đỏ.

---

## Bonus challenge

- [ ] Đã làm bonus (xem `bonus/`)
- [ ] Pair work với: _
