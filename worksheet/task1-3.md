# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Piranat Wadlom

**Student ID / รหัสนิสิต:** 6732031621

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 488 | 591 | 93.5% | 100% |
| t0-small | 16 | 41 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> The large model gives better accuracy (93.5% vs 72.5%). It costs more, because every review is a paid large-model call (100% vs 0%), and it is slower: the typical wait (p50) is about 470 ms longer (488 ms vs 16 ms).

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 28.9 | 224 | 412 | 19 | 0 | 93.9% |
| t1-after | 9.2 | 389 | 1008 | 0 | 25 | 93.1% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> Looking only at p50, the system seems better (224 ms vs 488 ms), but it is really worse. p50 only shows the middle request, so it says nothing about the 19 requests that failed. The `errors` column tells the real story (19 vs 0), and the run time also got longer (28.9 s).

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> A slow request finishes and frees its place, but a hung request keeps holding its connection in the review service and the gateway, and nobody else can use it. If several requests hang at once, the other reviews have to wait, so the problem spreads beyond the 10%. The run took 28.9 s with hangs, compared with 9.2 s once the timeout was added.

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> Label them later. Keep pending reviews in a queue or store and retry the large model when it is healthy again, instead of deleting them (FR4) or leaving them unlabelled forever (FR2). This is T4 (Asynchronous / batch): reviews are stored first and a worker adds the labels later.

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> At 0.1 s, almost every normal call (about 0.2 s) would be cut off even though nothing is wrong, so nearly all reviews would fall back to pending and the large model's work would be wasted. At 10 s, the timeout protects nothing: the client already gives up at 10 s, so the customer sees an error before the fallback can answer.

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 6 | 557 | 91.5% | 38% |
| t2-0.7 | 0.7 | 382 | 622 | 93.0% | 53% |
| t2-0.8 | 0.8 | 518 | 690 | 93.5% | 64% |
| t2-0.9 | 0.9 | 467 | 564 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> I would deploy threshold 0.6. It has 91.5% accuracy, which meets QR3 (at least 90%), the lowest share of large calls in the table (38%, matching QR4), and p95 of 557 ms, which meets QR1. The quality I give up is accuracy: it is 2 points lower than at threshold 0.8 (93.5%), and only 1.5 points above the 90% limit, so a harder real-world mix of reviews could push it below QR3.

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> At threshold 0.6, 62% of reviews are answered by the small model alone, so the middle request (p50) is in that fast group and takes only about 6 ms. The slowest 5% (p95) are in the group that is also sent to the large model, so they still wait about as long as before (557 ms). The cascade speeds up the reviews that skip the large model, but it does not speed up the ones that need it.

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> A review with "but", such as "The pad thai looked cold but it tasted excellent." The small model sees "cold" and answers negative with confidence 0.68, which is above the 0.6 threshold, so the large model is never asked even though the review is positive. The restaurant owner would get a false negative alert and the restaurant's satisfaction counts would be wrong.

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 6 | 569 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> Reviews whose text matches one seen before benefit, such as "delicious", "ok" or "good", and so do reviews that are reposted or retried. Unique reviews never benefit, such as long free-text ones or ones with different wording or a small typo (the cache matches the exact normalized text). The first time any text appears is always a miss.

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> The cached answers came from the old model, so if they are still served, the better model's accuracy never shows up and the labels in the system end up mixed between two model versions. I would put the model version in the cache key (for example v2 + hash of the text), so old answers never mix with new ones, and let them expire by TTL or clear them later.

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> At thresholds 0.8 and 0.9 the accuracy is the same (93.5%), but large calls rise from 64% to 77%, so the extra cost bought no extra accuracy.
