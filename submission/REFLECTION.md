# Reflection — Lab 19

Ten: Nguyen Trong Dang Khoa
Cohort: A20-K1
Path da chay: lite + docker (Qdrant server + Redis + Postgres)

---

Cau hoi (≤ 200 chu)

 tren golden set 50 queries, mode nao thang o loai query nao (exact /
paraphrase / mixed), va tai sao? Khi nao ban KHONG dung hybrid
(i.e. khi nao pure BM25 hoac pure vector la lua chon dung)?

Ket qua benchmark:
- exact (BM25 96.7%): BM25 thang — query chua tu ky thuat verbatim trong corpus, keyword matching du manh.
- paraphrase (semantic 24%): Semantic thang nhung overall thap — model bge-small-en English-trained yeu tren tieng Viet paraphrase.
- mixed (hybrid 100%): Hybrid thang tuyet doi — RRF ket hop ca semantic intent va exact terms.

Hybrid thang trung binh (78.6% > 77.8% keyword > 73.2% semantic) nho robust tren moi loai query.

Khong dung hybrid khi:
1. Latency cuc ky quan trong va BM25 da du chat luong (BM25 P99 chi ~4ms)
2. Corpus nho, hybrid overhead (RRF fusion) khong dang
3. Tat ca queries deu la exact term matches — BM25 du
4. Embedding model khong phu hop ngon ngu — hybrid kem hon pure BM25

---

Dieu ngac nhien nhat khi lam lab nay

Semantic cache o nguong 0.75 (con so AWS cong bo) van de lot 36% cau tra loi sai tren corpus nay — khong phai con so an toan de copy-paste. Va cross-tenant leak xay ra hoan toan im lang, khong exception, khong log do.

---

Bonus challenge

- [ ] Da lam bonus (xem bonus/)
- [ ] Pair work voi: _
