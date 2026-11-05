# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:**

**Student ID / รหัสนิสิต:**

This is **individual work**: each student completes and submits their own copy, in their own fork.

_งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง_

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls |
| -------- | ------ | ------ | -------- | ----------- |
| t0-large | 490    | 588    | 93.5%    | 100%        |
| t0-small | 11     | 25     | 72.5%    | 0%          |

**Q0.** What does the large model buy, and what does it cost?

_large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร_

> Large model ให้ Accuracy สูงกว่า Small model (93.5% เทียบกับ 72.5%) แต่ต้องแลกกับ Latency ที่สูงขึ้นมาก โดย p50 เพิ่มจาก 11 ms เป็น 490 ms และต้องเรียกใช้ Large model 100% ซึ่งอาจทำให้ต้นทุนการประมวลผลสูงขึ้น

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run       | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
| --------- | ------------ | ------ | ------ | ------ | ------- | -------- |
| t1-before | 33.1         | 236    | 426    | 22     | 0       | 92.7%    |
| t1-after  | 9.7          | 395    | 1010   | 0      | 28      | 93.0%    |

**Q1.1** In `t1-before`, p50 can look _better_ than the baseline. Why is p50 misleading here? Which column tells the real story?

_ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง_

> ค่า p50 วัดเฉพาะเวลาตอบสนองของ Request ที่ได้รับผลลัพธ์ จึงไม่สะท้อน Request ที่ค้างหรือเกิดข้อผิดพลาด ทำให้ดูเหมือนระบบเร็วขึ้น ทั้งที่มี Errors ถึง 22 รายการ ดังนั้นควรพิจารณา column errors ร่วมด้วย

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

_การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway_

> Request ที่ค้างจะถือทรัพยากรของ Review Service และ Gateway เช่น Connection, Memory และช่องรองรับ Request ที่กำลังประมวลผล ทำให้ทรัพยากรถูกใช้งานสะสม และอาจส่งผลให้ Request อื่นช้าลงหรือเกิดข้อผิดพลาดตามไปด้วย

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

_รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น_

> ควรนำรีวิวที่ Pending ไปประมวลผลภายหลังผ่าน Background Worker และอัปเดต Label เมื่อประมวลผลเสร็จ โดยใช้ Asynchronous Queue-Based Topology เพื่อแยกการรับ Request ออกจากการประมวลผล

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

_เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที_

> หากตั้ง Timeout 0.1 วินาที อาจตัด Request ที่ยังทำงานปกติแต่ใช้เวลานานกว่า 100 ms ทำให้เกิด Timeout โดยไม่จำเป็น ส่วน 10 วินาทีจะทำให้ Request ที่ค้างถือทรัพยากรนานเกินไป ส่งผลให้ Latency สูงขึ้นและระบบรองรับ Request พร้อมกันได้น้อยลง

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run    | threshold | p50 ms | p95 ms | accuracy | large calls |
| ------ | --------- | ------ | ------ | -------- | ----------- |
| t2-0.6 | 0.6       | 6      | 559    | 91.5%    | 38%         |
| t2-0.7 | 0.7       | 357    | 588    | 93.0%    | 53%         |
| t2-0.8 | 0.8       | 501    | 739    | 93.5%    | 64%         |
| t2-0.9 | 0.9       | 474    | 631    | 93.5%    | 77%         |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

_จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป_

> เลือก Threshold 0.7 เนื่องจากให้ Accuracy 93.0% ใกล้เคียงกับ Large model (93.5%) แต่ลด Large model calls เหลือ 53% และมี p50 เท่ากับ 357 ms โดยยอมเสีย Accuracy เล็กน้อย 0.5 percentage points เพื่อแลกกับประสิทธิภาพและต้นทุนที่ดีขึ้น

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

_ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก_

> Cascade ช่วยให้ Request ที่ Small model มี Confidence สูงได้รับผลลัพธ์ทันที จึงลด Latency ของ Request จำนวนมาก แต่ Request ที่มี Confidence ต่ำยังต้องเรียก Large model ทำให้กลุ่ม Request ที่ช้าที่สุดยังมี Latency สูง และ p95 ลดลงได้น้อยกว่า p50

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

_cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้_

> เช่น รีวิวที่ใช้คำประชดว่า "อาหารอร่อยมาก รอไปแค่สองชั่วโมงเอง" Small model อาจมี Confidence สูงว่าเป็นรีวิวเชิงบวก ทั้งที่มีความหมายเชิงลบ ทำให้ระบบจัดประเภทรีวิวผิดและส่งผลต่อความน่าเชื่อถือของผลการวิเคราะห์

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls | cache hits |
| -------- | ------ | ------ | -------- | ----------- | ---------- |
| t3-cache | 8      | 565    | 93.0%    | 38%         | 35%        |

**Q3.1** Which reviews benefit from the cache, and which never will?

_รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์_

> รีวิวที่มีข้อความซ้ำกับที่เคยประมวลผลและบันทึกไว้ใน Cache จะได้รับผลลัพธ์เร็วขึ้นโดยไม่ต้องเรียก Model ใหม่ ส่วนรีวิวใหม่ที่ไม่เคยปรากฏใน Cache หรือมีข้อความแตกต่างจากเดิมจะไม่ได้ประโยชน์จาก Cache ในการประมวลผลครั้งแรก

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

_สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร_

> ควรทำ Cache Invalidation เพื่อป้องกันการนำผลทำนายจาก Model เก่ามาใช้ โดยออกแบบ Cache Key ให้รวม Model Version และ Prediction Configuration เพื่อให้ระบบแยกผลลัพธ์ของแต่ละเวอร์ชัน และสามารถลบ Cache เก่าได้อย่างเหมาะสม

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

_ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture_

> ได้เรียนรู้ว่า Cache ไม่ได้ช่วยลด Latency ทุกกรณี โดยผลการทดสอบ t3-cache มี p50 เพียง 8 ms แต่ p95 ยังสูงถึง 565 ms แสดงให้เห็นว่า Cache ช่วยให้ Request ที่พบข้อมูลซ้ำตอบสนองเร็วขึ้นมาก แต่ Request ที่ไม่พบใน Cache ยังต้องใช้เวลาประมวลผลผ่าน Model ตามปกติ
