# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** ธนกร ไชยยุทธ

**Student ID / รหัสนิสิต:** 6631321321

This is **individual work**: each student completes and submits their own copy, in their own fork.

_งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง_

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls |
| -------- | ------ | ------ | -------- | ----------- |
| t0-large | 506    | 602    | 93.5%    | 100%        |
| t0-small | 33     | 74     | 72.5%    | 0%          |

**Q0.** What does the large model buy, and what does it cost?

_large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร_

> large model ให้ accuracy 93.5% ซึ่งผ่าน QR3 (>= 90%) ขณะที่ small model ได้แค่ 72.5% แต่ต้องแลกด้วยความช้า (p50 ประมาณ 506 ms เทียบกับ 33 ms) ทุก call มีค่าใช้จ่าย และรับได้พร้อมกันแค่ 4 calls (QR5 สูงสุด 20 calls ต่อวินาที)

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run       | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
| --------- | ------------ | ------ | ------ | ------ | ------- | -------- |
| t1-before |              | 259    | 488    | 18     | 0       | 92.9%    |
| t1-after  |              | 417    | 1022   | 0      | 21      | 92.7%    |

**Q1.1** In `t1-before`, p50 can look _better_ than the baseline. Why is p50 misleading here? Which column tells the real story?

_ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง_

> p50 บอกแค่ request ตรงกลาง จึงไม่สะท้อน request ที่ค้างหรือล้มเหลวซึ่งอยู่ปลายหาง ความจริงอยู่ที่ column `errors` (18 รายการที่ไม่ได้คำตอบภายใน 10 วินาที) และเวลาที่ run ใช้ทั้งหมด

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

_การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway_

> request ที่ค้างถือ connection และ task ไว้ใน review service กับ gateway ตลอด 20 วินาที และ large model รับได้พร้อมกันแค่ 4 calls ถ้า calls ที่ค้างจองช่องไว้ request ปกติอีก 90% ก็ต้องต่อคิวรอด้วย ผลกระทบจึงลามทั้งระบบ ไม่ใช่แค่ 10% ที่ช้า

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

_รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น_

> ควรเก็บรีวิวเป็น pending ไว้ก่อน แล้วให้ batch job มาติด label ทีหลังนอก request path ตามหลัก batch prediction

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

_เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที_

> ถ้า 0.1 วินาที: large call ปกติใช้ประมาณ 200 ms จึง timeout เกือบทุกครั้ง รีวิวเกือบทั้งหมดกลายเป็น pending และยังเสียค่า large model ไปเปล่าๆ
> ถ้า 10 วินาที: ผิด QR2 (รอได้ไม่เกิน 1 วินาที) และ request ที่ค้างจะถือ resource นานมาก

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run    | threshold | p50 ms | p95 ms | accuracy | large calls |
| ------ | --------- | ------ | ------ | -------- | ----------- |
| t2-0.6 | 0.6       | 17     | 533    | 91.5%    | 38%         |
| t2-0.7 | 0.7       | 375    | 591    | 93.0%    | 53%         |
| t2-0.8 | 0.8       | 447    | 587    | 93.5%    | 64%         |
| t2-0.9 | 0.9       | 485    | 607    | 93.5%    | 77%         |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

_จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป_

> เลือก threshold 0.7 เพราะได้ accuracy 93.0% (ผ่าน QR3) และส่ง large model แค่ 53% เทียบกับ 64% ที่ 0.8 (QR4) แต่แลกกับ accuracy ที่ลดลงเมื่อเทียบกับ 0.8 เพื่อแลกกับ cost และ capacity ที่ดีขึ้น

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

_ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก_

> ที่ 0.7 p50 ลดจาก 506 เหลือ 375 ms แต่ p95 เกือบเท่าเดิม เพราะ request ส่วนใหญ่ small model มั่นใจและตอบเองเลยจึงเร็ว แต่ request ที่ small ไม่มั่นใจยังต้องไปถาม large ซึ่งช้าเท่าเดิม

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

_cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้_

> small model อ่านประโยคปฏิเสธ ("not cold") หรือประโยคที่มี "but" ผิดได้ทั้งที่มั่นใจสูง รีวิวจึงไม่ถูกส่งต่อไป large ผลคือรีวิวที่ไม่พอใจอาจถูกติดเป็น positive และเจ้าของร้านไม่เห็นลูกค้าที่ควรรีบแก้ปัญหา

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls | cache hits |
| -------- | ------ | ------ | -------- | ----------- | ---------- |
| t3-cache | 14     | 529    | 93.0%    | 38%         | 35%        |

**Q3.1** Which reviews benefit from the cache, and which never will?

_รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์_

> รีวิวที่ซ้ำกันได้ประโยชน์ ส่วนรีวิวที่ไม่ซ้ำใครไม่มีวันได้ประโยชน์

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

_สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร_

> คำตอบใน cache มาจาก Model เก่า จึงต้องล้าง cache หรือไม่ให้ใช้คำตอบเก่าต่อ วิธีออกแบบคือใส่เลขเวอร์ชันของ Model ลงใน cache key (หรือกำหนดอายุ TTL) เพื่อให้ Model ใหม่ไม่เจอคำตอบเก่า

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

_ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture_

> p95 ของ `t1-after` สูงขึ้นเป็น 1022 ms (เกิน 1 วินาทีของ QR1) เพราะรีวิวที่ pending ต้องรอ timeout ครบก่อน fallback ทำให้ error เป็น 0 แต่ไม่ได้ทำให้ระบบเร็วขึ้น และ cache ลด p50 ได้มาก (375 เหลือ 14 ms) แต่แทบไม่ช่วย p95
