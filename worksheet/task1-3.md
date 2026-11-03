# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** พีรวัส พิบูลย์วรกุล

**Student ID / รหัสนิสิต:** 6732033921

This is **individual work**: each student completes and submits their own copy, in their own fork.

_งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง_

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls |
| -------- | ------ | ------ | -------- | ----------- |
| t0-large | 501    | 619    | 93.5%    | 100%        |
| t0-small | 28     | 62     | 72.5%    | 0%          |

**Q0.** What does the large model buy, and what does it cost?

_large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร_

> ให้ % accuracy ที่ดีกว่า small model แต่ใช้เวลาในการ process มากกว่าและมีการใช้ large_calls ซึ่งแพงกว่า

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run       | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
| --------- | ------------ | ------ | ------ | ------ | ------- | -------- |
| t1-before | 32.9         | 241    | 409    | 22     | 0       | 92.7%    |
| t1-after  | 9.2          | 392    | 1021   | 0      | 27      | 93.6%    |

**Q1.1** In `t1-before`, p50 can look _better_ than the baseline. Why is p50 misleading here? Which column tells the real story?

_ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง_

> เพราะ p_50 คือค่ากลางของเวลาที่เรียงจาก requests ที่สำเร็จทั้งหมด เพราะฉะนั้นหากจะดูภาพรวมต้องไปดูที่ errors และ run time เพราะของ case before มีค่ามากเพราะมีการ hang เกิดขึ้น

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

_การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway_

> request ที่ hang ประมาณ 10% จะค้างอยู่ที่ Inference service ซึ่งถูกเรียกใช้โดย Review service ที่ถูกเรียกใช้โดย API Gateway อีกรอบ แสดงว่ามันก็จะเกิด bottleneck ใน case ที่มัน hang ทำให้ request ที่เหลือนั้นอาจไม่ถูก API Gateway,Review Service หรือ Inference Service อ่านด้วยซ้ำเพราะ connection หรือทรัพยากรที่ใช้รองรับ request พร้อมกันอาจถูกใช้จนเต็ม ทำให้ requestเหล่านั้นไม่ถูกนำมาคิด ทำให้ต้องรอหรือ timeout หรือนับเป็น error และไม่นำมาคิดใน p50 และ p95

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

_รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น_

> เก็บ pending review ไว้ใน buffer สำหรับ pending ถ้าไม่มี label ก็นำมาประมวลผลใหม่อีกครั้งนึงเพื่อให้มี Label ตรงกับ Asynchronous/ batch

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

_เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที_

> ถ้าเป็น 0.1 sec จะใช้เวลาน้อยไป ทำให้ request ที่ไม่น่าจะเกิด hang ดันไม่สามารถ process ได้ทัน จึงจะเกิด pending เยอะและต้องเสียเวลามากขึ้นในการมา label อีกรอบ ส่วนถ้าเป็น 10 วินาทีจะใช้เวลามากไป ทำให้ gateway กับ review service ต้องถือทรัพยากรระหว่างรอนานขึ้น

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run    | threshold | p50 ms | p95 ms | accuracy | large calls |
| ------ | --------- | ------ | ------ | -------- | ----------- |
| t2-0.6 | 0.6       | 16     | 522    | 91.5%    | 38%         |
| t2-0.7 | 0.7       | 385    | 562    | 93.0%    | 53%         |
| t2-0.8 | 0.8       | 419    | 590    | 93.5%    | 64%         |
| t2-0.9 | 0.9       | 474    | 593    | 93.5%    | 77%         |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

_จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป_

> 0.6 เพราะมี accuracy มากเพียงพอ requirements และใช้เวลาน้อย แถมถูกกว่า โดยเราเสีย quality ในส่วนของ accuracy ไปเพียงไม่กี่ % เมื่อเทียบกับ threshold ที่มากกว่า

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

_ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก_

> เพราะ request จะถูกคิดโดย small model ก่อนซึ่งใช้เวลาเร็ว การ cascade จึงมีผลมากเมื่อเทียบกับ large model ที่ช้าในพาร์ท p95 โดย request 38% เท่านั้นที่ไปถึง large cell แสดงว่ากลุ่ม p50 ก็จะไม่ต้องรอ large model

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

_cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้_

> small model อาจมีการตีความผิดได้หากเจอคำแบบ not bad เป็นต้น มันอาจจะเกิดการ label ว่าแย่ได้ทั้งๆที่มันอาจจะดี และถ้า small model มั่นใจเกิน อาจจะมี confidence มากก่า treshold ทำให้ตกไม่ถึง large model ทำให้ผู้ใช้เข้าใจผิด หรืออาจอารมณ์เสียถ้าเจอ false negative , false positive บ่อยๆ

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls | cache hits |
| -------- | ------ | ------ | -------- | ----------- | ---------- |
| t3-cache | 10     | 536    | 91.5%    | 31%         | 36%        |

**Q3.1** Which reviews benefit from the cache, and which never will?

_รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์_

> รีวีวที่เป็นข้อความที่เคยพบแบบเป๊ะๆ โดยที่ case insensitive จะถูกนำมาเทียบกับ cache ซึ่งเป็นประโยชน์ เช่น "Good" กับ "good" หรือ "Well done" กับ "well done " ส่วนที่ไม่มีประโยชน์เช่น "So good" กับ "So gooddddd" จะไม่ถูกนำมาคิดว่าเป็นตัวเดียวกัน และกรณีที่ข้อความไม่เคยซ้ำก็จะไม่ได้ใช้ cache

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

_สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร_

> เพิ่มการเก็บ model versioning ใน cache โดยเก็บให้ version ของ small model แยกกันจาก large model และหากเปลี่ยน treshold ก็อาจจะมี field เก็บเพิ่มใน cache สำหรับแต่ละ treshold ด้วย และสำหรับข้อมูลจาก cache เก่าเราก็ลบทิ้งเพื่อไม่ให้เปลือง storage

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

_ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture_

> รู้เรื่องการที่เราจะใช้ small model ร่วมกับ large model อย่างมีประสิทธิภาพอย่างไร เช่น นำ small model มา label ก่อนแล้วค่อยเอาที่เหลือที่ confidence level < threshold ให้ large จัดการโดยสำหรับ threshold 0.6 เราจะได้ accuracy ถึง 91.5% โดยมี request 38% ที่ถูก label โดย large model
