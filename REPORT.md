MSSV: 2A202602367

# Lab 11 · Scenario Forge — báo cáo

Họ tên: Nguyễn Đức Đông. Mọi con số dưới đây trích từ `evidence/` của lần chạy đầy đủ (không `--quick`), chạy với `LAB11_MSSV=2A202602367` (MSSV có chữ "A" nên regex dòng đầu của `run_all.py` không đọc được; grader truyền MSSV qua biến môi trường này).

Thị trấn của bạn (`meta.json: _run.town`): **B** (thị trấn "mưa nhiều": P(đường ướt) = 0,30). Seed: `meta.json: seeds = {mc: 3505141612, search: 1344846987, test: 913688236}`.

## 1. Phát lại log có trung thực không? (bước A)

| | số |
|---|---|
| kịch bản (`replay.json: n`) | 400 |
| bỏ vì chính người lái va chạm (`dropped_human_crash_logs`) | 2 |
| dùng (`n_used`) | 398 |
| planner trên **log phát lại** (`replay.crashes`) | **120**: `rear` 106, `side` 14, `front` 0 |
| planner trong **thế giới phản ứng** (`reactive.crashes`) | **0** |
| va chạm ở cả hai (`crash_in_both`) | 0 |

Phần lớn va chạm khi phát lại là `rear` vì planner thận trọng hơn người lái trong log: gần vạch qua đường bị xe buýt che, nó giảm về `cautionV` và nhường người đi bộ sớm hơn. Xe bám đuôi TG trong log phát lại vẫn chạy đúng quỹ đạo đã ghi, tức quỹ đạo bám theo một người lái phanh **muộn hơn**. Log không biết planner phanh sớm hơn, nên TG không phanh và tông vào đuôi xe ta. Đó là va chạm **giả**: trong thế giới phản ứng TG lái theo IDM, thấy xe ta chậm lại và phanh theo, nên số va chạm là 0 và không kịch bản nào va chạm ở cả hai. 14 va chạm `side` cũng là chuyện của log: xe C2 nhập làn theo quỹ đạo ghi sẵn vào chỗ mà người lái đã để trống, nhưng xe ta (chậm hơn) đang nằm đúng chỗ đó.

Ví dụ `replay.json: replay_only[0]` (kịch bản 3, ảnh `evidence/shots/q1_replay_only_R-01.png`, R-01 trong chương 9): đường ướt (μ = 0,5), người đi bộ có ngó qua xe buýt, TG bám với khoảng `tgT` = 1,53 s. Xe ta đến gần vạch bị che thì giảm tốc rồi dừng nhường người đi bộ; TG phát lại theo log vẫn giữ tốc độ của lần người lái chạy và tông đuôi xe ta lúc 17,75 s. Trong thế giới phản ứng, TTC nhỏ nhất của cùng kịch bản là `min_ttc_reactive` = 7,53 s, tức không có gì nguy hiểm.

## 2. Kịch bản bất khả thi (bước B)

Luật trong `is_feasible` (`lab11/feasible.py`):

| luật | căn cứ vật lý |
|---|---|
| a_lat = (π/cutDur)²·3,5/2 > μg → loại | chuyển làn 3,5 m theo dạng cosin có gia tốc ngang cực đại đó; lốp chỉ tạo được lực ngang tối đa μ·m·g (vòng ma sát). Đường ướt μ = 0,5 → cutDur < 1,88 s là bất khả thi; khô μ = 0,8 → cutDur < 1,48 s. |
| cutBrake > μg → loại | gia tốc phanh không vượt quá ma sát lốp–mặt đường (μg = 4,9 m/s² khi ướt, 7,85 m/s² khi khô). |
| pedV > 2,5 m/s → loại | người đi bộ băng qua với tốc độ đi bộ, nhanh nhất là đi vội/chạy chậm ~2,5 m/s; nhanh hơn là chạy nước rút, không phải mô hình "người qua đường". |
| cutV ∉ (0, 22] m/s → loại | xe chuyển làn trong phố phải đang chạy (> 0), và 22 m/s ≈ 80 km/h là trần hợp lý cho một con phố đô thị. |

Số kịch bản bất khả thi bị bỏ trong `hunt.json`: `bus_bypass` 6, `cut_in` 3, `crosswalk_block` 5, **`rear_ended` 9** (nhiều nhất). Điểm nguy hiểm của cuộc săn `rear_ended` là TTC của xe bám đuôi; cách dễ nhất để TG tông xe ta là xe ta phải phanh gấp vì một xe cắt làn **đột ngột** trên **đường ướt**. Đề xuất cuối của nó có p(ướt) = 0,83 và độ lệch chuẩn của toạ độ `cutDur` lớn (1,35), nên nó rút được nhiều mẫu cutDur ngắn trên đường ướt, vượt μg. CEM bị kéo về phía đó vì kịch bản bất khả thi chính là kịch bản "nguy hiểm" nhất theo điểm số: một cú lách không lốp nào làm nổi cho xe ta ít thời gian phản ứng nhất. Nếu **đếm** chúng là va chạm, elite sẽ toàn là kịch bản bất khả thi, q dồn hẳn về vùng đó, và ước lượng IS bị thổi phồng bởi những "va chạm" không thể xảy ra ngoài đời (cận trên lẫn ước lượng đều chệch lên, và kiểu thất bại tìm được là giả).

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: **k = 3**, **n = 3 000**, p̂ = `p` = **1,00·10⁻³**, Wilson 95 % = [`lo` 3,40·10⁻⁴, `hi` 2,94·10⁻³]. Cả 3 va chạm là `bus_bypass` (`modes`); 11 mẫu bất khả thi bị bỏ (`infeasible`).

Để khoảng tin cậy hẹp còn ±20 % của p: 1,96·√(p(1−p)/n) = 0,2p ⇒ n ≈ 1,96²/(0,2²·p) ≈ **96/p ≈ 96 000** lần chạy, gấp 32 lần ngân sách MC này.

Các kiểu **chưa hề xuất hiện** trong 3 000 lần chạy: `cut_in`, `crosswalk_block`, `rear_ended` và `other` (k = 0). Con số đúng cần báo **không phải 0** mà là cận trên: Wilson 95 % với k = 0, n = 3 000 cho `mc.json: per_mode.<kiểu>.hi` = **1,28·10⁻³** (quy tắc ba: 3/3 000 = 1,0·10⁻³). Tức "dưới khoảng 10⁻³", không phải "không bao giờ".

## 4. Importance sampling, ESS và độ suy biến (bước C3)

`estimate.json: seeds[*].per` (n = 600 mẫu mỗi kiểu mỗi seed):

| kiểu | seed | p | hits | ess | top_share |
|---|---|---|---|---|---|
| bus_bypass | 0 | 8,27·10⁻³ | 28 | 136 | **0,937** |
| bus_bypass | 1 | 1,59·10⁻⁴ | 26 | 127 | 0,302 |
| bus_bypass | 2 | 9,46·10⁻⁴ | 36 | 115 | 0,317 |
| cut_in | 0 / 1 / 2 | 0 (cận `upper_zero_hits` = 0,025) | 0 / 0 / 0 | 118 / 145 / 141 | 0 |
| crosswalk_block | 0 | 8,63·10⁻⁶ | 114 | 113 | 0,378 |
| crosswalk_block | 1 | 4,43·10⁻⁶ | 110 | 118 | 0,476 |
| crosswalk_block | 2 | 3,69·10⁻⁶ | 101 | 128 | 0,151 |
| rear_ended | 0 / 1 / 2 | 0 (cận `upper_zero_hits` = 0,025) | 0 / 0 / 0 | 131 / 131 / 124 | 0 |

Tổng (`total`): seed 0 = 8,28·10⁻³, seed 1 = 1,63·10⁻⁴, seed 2 = 9,50·10⁻⁴; trung vị **9,50·10⁻⁴**. `other` = 0 với cận 1,28·10⁻³ (`estimate.json: other`, `source: mc.json`).

- **Ba seed có đồng ý không?** `crosswalk_block` đồng ý khá tốt (3,7–8,6·10⁻⁶, chênh ~2 lần). `bus_bypass` thì không: chênh nhau **52 lần** (1,6·10⁻⁴ đến 8,3·10⁻³). Trung vị 9,5·10⁻⁴ khớp với Monte Carlo (1,0·10⁻³, Wilson [3,4·10⁻⁴; 2,9·10⁻³]); seed 1 nằm dưới khoảng Wilson của MC, seed 0 nằm trên. `crosswalk_block` ~5·10⁻⁶ không mâu thuẫn MC (k = 0 trong 3 000 lần, cận 1,28·10⁻³).
- **`top_share` lớn:** `bus_bypass` seed 0 có `top_share` = 0,937: **một mẫu duy nhất** mang 94 % ước lượng. Khi đó p̂ ≈ w_max/n, tức con số thực chất chỉ là trọng số p/q của một mẫu rơi vào vùng q phủ mỏng, và ±1,96·se (ở đây [0; 2,35·10⁻²]) không đáng tin. Đó là lý do seed 0 cao gấp ~9 lần trung vị. ESS = 136 không cảnh báo điều này, vì ESS đo trọng số của **mọi** mẫu, còn ước lượng chỉ phụ thuộc vào các mẫu có y = 1. Ngược lại `top_share` thấp (0,15 ở `crosswalk_block` seed 2) cũng không chứng minh là đúng: q có thể bỏ sót cả một vùng va chạm.
- **Ngân sách:** `budget` = 16 800 lần chạy (4 cuộc săn × 8 vòng × 300 + 3 seed × 4 kiểu × 600) cho mọi kiểu cùng lúc. Để có cùng độ chính xác tương đối như IS, MC cần n ≈ (1−p)/(p·(se/p)²):
  - `bus_bypass` seed 2 (se/p = 0,41): n_MC ≈ 6 200, IS không lợi bao nhiêu với kiểu "không hiếm" này;
  - `crosswalk_block` seed 2 (p = 3,69·10⁻⁶, se/p = 0,27): n_MC ≈ **3,8 triệu** lần chạy, so với 600 mẫu IS (cộng 2 400 lần săn). Ở đây IS lợi hơn ~1 000 lần.
- **Phần `other`:** các va chạm không thuộc kiểu nào không có điểm nguy hiểm để CEM leo dốc (không biết "gần va chạm kiểu other" nghĩa là gì), nên không săn được. Phần của nó lấy từ Monte Carlo C1: `other` k = 0/3 000, cận 1,28·10⁻³, và được cộng vào tổng.

## 5. Các kiểu thất bại (bước C2 + 3D)

`hunt.json: <cuộc săn>.found`:

| cuộc săn | found | bất khả thi |
|---|---|---|
| bus_bypass | bus_bypass 87 | 6 |
| cut_in | **bus_bypass 29**, crosswalk_block 1 | 3 |
| crosswalk_block | crosswalk_block 103, bus_bypass 5 | 5 |
| rear_ended | **bus_bypass 3, crosswalk_block 5** | 9 |

- **`bus_bypass` (lách qua xe buýt)**, ảnh `evidence/shots/q5_bus_bypass_B-01.png` (B-01, va chạm 12,95 s): xe buýt đỗ ở làn phải chắn đường của C2 (xe hàng xóm); C2 chạy chậm, lách qua đuôi xe buýt sang làn của xe ta. Đường ướt, C2 chấp nhận một khe chỉ 9,2 m. Planner chỉ coi C2 là "xe phía trước" khi độ lệch ngang < `cutinLat` = 2,15 m, nên khi C2 còn đang nhập làn nó chưa phanh, và khi nhận ra thì không đủ quãng phanh trên mặt đường μ = 0,5. Xe ta tông vào hông/đuôi C2.
- **`crosswalk_block` (chặn lối người đi bộ)**, ảnh `evidence/shots/q5_crosswalk_block_C-01.png` (C-01, 21,35 s): người đi bộ P1 đứng sau xe buýt, xe ta không nhìn thấy họ (người đi bộ không ngó qua xe buýt). Khi xe buýt rời bến/khi P1 bước ra khỏi vùng bị che thì xe ta mới thấy, phanh gấp trên đường ướt và dừng với thân xe nằm trên vạch qua đường, chắn lối P1 đang qua.
- **Không tìm thấy:** `cut_in` và `rear_ended`. Cuộc săn `cut_in` kéo về `bus_bypass` (29 lần) vì cách dễ nhất để TTC tới C2 về 0 là để C2 lách qua xe buýt chậm, mà theo định nghĩa đó lại là `bus_bypass`. Cuộc săn `rear_ended` chỉ đẩy được mức elite đến −1,19 (TTC của TG còn 1,19 s) và mọi va chạm nó gặp là kiểu khác. Mức tin: trung bình. MC 3 000 lần cũng không thấy hai kiểu này (cận 1,28·10⁻³), IS báo cận `upper_zero_hits` = 0,025 cho mỗi kiểu, nên điều chắc chắn chỉ là "dưới khoảng 10⁻³"; ngân sách này không loại trừ được chúng ở mức 10⁻⁵.
- **Một cuộc săn chung:** nhìn `found` thì thấy ngay: cuộc săn `cut_in` và `rear_ended` cũng bị hút về `bus_bypass`/`crosswalk_block`. Một cuộc săn chung cho mọi kiểu sẽ **co về một kiểu** dễ nhất (`bus_bypass`: elite chạm mức 0 từ vòng 4) và lặp mãi kiểu đó, nên `crosswalk_block` (p ~5·10⁻⁶, khó hơn ~200 lần) gần như không bao giờ được thấy. Vì thế mỗi kiểu có một cuộc săn riêng với điểm nguy hiểm riêng.

## 6. Sửa planner (bước E + 3D)

Policy được chọn (`fix.json: chosen.policy`): **`{"cutinLat": 2.6, "cautionV": 7.0, "pedBuffer": 3.5}`**, chọn bằng `choose_policy` chỉ trên TRAIN.

| | fail | fail_hi_seen | fail_hi (hợp lệ) | quãng đường | phanh gấp (khung/lần chạy) |
|---|---|---|---|---|---|
| bản sửa, TRAIN (`chosen`) | 3,59·10⁻⁵ | 9,73·10⁻⁵ | 9,73·10⁻⁵ (`unseen`: bus_bypass) | 199,7 m | 9,30 |
| gốc, TRAIN (`baseline_train`) | 5,60·10⁻⁴ | 1,04·10⁻³ | 1,04·10⁻³ | 207,1 m | 8,72 |
| **bản sửa, TEST** (`chosen_test`) | **7,26·10⁻⁵** | **2,12·10⁻⁴** | **1,49·10⁻³** | 207,3 m | 9,31 |
| **gốc, TEST** (`baseline_test`) | **5,74·10⁻⁴** | 9,77·10⁻⁴ | 2,26·10⁻³ | 214,3 m | 8,64 |

TRAIN và TEST tách bạch: trên TRAIN bản sửa không gặp `bus_bypass` nào (`unseen`), nên TRAIN "0 va chạm" kiểu đó không chứng minh gì. Con số báo cáo là TEST: seed mới và một cuộc săn mới nhắm vào chính bản sửa. Trên TEST, cuộc săn `bus_bypass` mới chỉ tìm được **1** va chạm (gốc: 29), và tỉ lệ thất bại giảm ~8 lần (7,26·10⁻⁵ so với 5,74·10⁻⁴), đổi lại mất 3,3 % quãng đường (207,3 so với 214,3 m) và phanh gấp nhiều hơn một chút.

**Vì sao `fail_hi` của bản sửa (1,49·10⁻³) lớn hơn `fail` của gốc (5,74·10⁻⁴) dù bản sửa tốt hơn thật:** `fail_hi` cộng cận trên của **mọi** va chạm khác (`other_crashes`: k = 0 trên n = 3 000 lần chạy tự nhiên, cận 1,28·10⁻³). Cận này là giới hạn của lượng dữ liệu, không phải của planner: với 3 000 lần chạy không thể chứng nhận gì dưới ~10⁻³. So sánh công bằng là `fail_hi` với `fail_hi` (1,49·10⁻³ < 2,26·10⁻³) hoặc `fail_hi_seen` (2,12·10⁻⁴ < 9,77·10⁻⁴).

Ảnh có bóng mờ: `evidence/shots/q6_ghost_fix_avoids_B-02.png` (B-02, `bus_bypass` tìm được dưới policy gốc). Xe đặc là policy gốc, va chạm C2 lúc 12,4 s; **bóng mờ** là bản sửa trên cùng kịch bản, không va chạm (readout chương 9: "chính sách gốc: không va chạm" là bóng của policy đã chọn). Với `cutinLat` = 2,6 m, bản sửa coi C2 là xe phía trước sớm hơn, khi C2 mới bắt đầu lách qua đuôi xe buýt, nên phanh kịp. Bóng mờ nằm phía sau xe ta.

**Núm "nghe thì an toàn hơn" nhưng phản tác dụng** (`fix.json: table`, TRAIN):

| ứng viên | fail | fail_hi | quãng đường |
|---|---|---|---|
| gốc (cautionV 7,5) | 5,60·10⁻⁴ | 1,04·10⁻³ | 207,1 m |
| `cautionV` 5,0 (chậm hơn gần vạch bị che) | **4,99·10⁻³** | **1,48·10⁻²** | 168,6 m |
| `T` 2,0 (giữ khoảng cách xa hơn) | 5,84·10⁻⁴ | 1,01·10⁻³ | 197,0 m |

Đi **chậm hơn** gần vạch qua đường bị che (cautionV 5,0) làm thất bại tăng **~9 lần** chứ không giảm. Thế giới phản ứng lại: xe ta bò chậm cạnh xe buýt lâu hơn, nên C2 (đang kẹt sau xe buýt và chờ một khe, *gap acceptance*) thấy phía trước xe ta một khe "đủ rộng" và lách vào, đúng lúc xe ta ở ngay cạnh, tạo ra thêm `bus_bypass`. Đồng thời xe ta phanh gấp nhiều hơn (12,9 so với 8,7 khung/lần) và mất 19 % quãng đường. Tương tự, `T` = 2,0 để khoảng trống lớn hơn phía trước: khoảng trống đó mời xe khác cắt vào, nên tỉ lệ thất bại không giảm (5,84·10⁻⁴ so với 5,60·10⁻⁴). Một planner không thể an toàn hơn chỉ bằng cách "rụt rè hơn" khi người khác đọc sự rụt rè đó như một lời mời.

## 7. Từ mô phỏng ra đường thật

| giả định (`space.py`) | dữ liệu thật thay thế, lấy từ đâu | nếu sai thì con số sai theo hướng nào |
|---|---|---|
| **Khe chấp nhận của xe chuyển làn** `cutGap` (log-normal, trung vị 12 m, σ = 0,5) | Phân bố khe chấp nhận khi nhập làn / lách xe đỗ, đo trên dữ liệu quỹ đạo drone/camera: highD/inD/rounD (RWTH), NGSIM (FHWA), Waymo Open Motion Dataset, lọc riêng các tình huống lách xe buýt đỗ ở trạm. | `bus_bypass` (kiểu trội của tôi) rất nhạy với đuôi trái của phân bố. Nếu người thật chấp nhận khe ngắn hơn (ví dụ ở đô thị đông như Hà Nội), p của tôi **thấp hơn** thực tế; nếu họ cẩn thận hơn thì p bị **thổi phồng**. |
| **Tỉ lệ đường ướt** `wet` = 0,30 (thị trấn B) và **μ ướt = 0,5** | Số giờ mưa theo khí tượng địa phương (trạm khí tượng, ERA5) nhân với lưu lượng theo giờ; μ thật từ đo ma sát mặt đường (máy đo GripTester/SCRIM của cơ quan quản lý đường) theo loại nhựa và độ mòn. | Cả `bus_bypass` lẫn `crosswalk_block` đều xảy ra chủ yếu trên đường ướt (mọi ảnh của tôi là μ = 0,5). Nếu μ ướt thật là 0,6–0,7 thì con số của tôi **quá cao**; nếu mặt đường mòn/trơn (μ 0,3–0,4) thì **quá thấp**, và còn sinh thêm kịch bản mà lab đang loại là "bất khả thi". |
| **Người đi bộ ngó qua xe buýt** `pedLook` = 0,85 và khe chấp nhận `pedGap` (trung vị 4,5 s) | Quan sát video tại trạm buýt (nghiên cứu hành vi qua đường, dữ liệu inD/Waymo có nhãn người đi bộ), khảo sát hiện trường đếm số người nhìn trước khi bước ra sau xe buýt. | `crosswalk_block` gần như chỉ xảy ra khi người đi bộ **không** ngó. Nếu tỉ lệ ngó thật thấp hơn 0,85 (người dùng điện thoại, trẻ em), p của `crosswalk_block` bị **ước lượng thấp**, có thể nhiều lần. |

## 8. Giới hạn

1. **Một con phố, một kịch bản cố định:** mọi xác suất là "mỗi lần đi qua đúng đoạn phố có xe buýt đỗ trước vạch qua đường này", với đúng 1 xe chuyển làn, 1 người đi bộ, 1 xe bám đuôi. Không có giao lộ, rẽ trái, xe máy, đêm tối, nên không suy ra được tỉ lệ va chạm trên km hay trên chuyến.
2. **Người lái và người đi bộ đều là mô hình:** các xe khác là IDM + gap acceptance + nhường, "lịch sự" (`civil = True`); không ai say rượu, mất tập trung, vượt đèn, hay phản ứng chậm ngẫu nhiên. Người đi bộ không bao giờ bước vào thân xe. Các kiểu thất bại do người khác cư xử tệ bị đánh giá thấp.
3. **Cảm biến hoàn hảo trừ phần bị xe buýt che:** không nhiễu, không trễ, không bỏ sót trong mưa (dù 30 % là đường ướt), không nhầm phân loại. Planner chỉ phanh, không đánh lái. Ước lượng của tôi là cận dưới cho một hệ thống có perception thật.

Thêm một tuần, tôi sẽ: (a) chạy Monte Carlo cỡ ~10⁵ lần (khoảng 7 giờ CPU) để kiểm tra trực tiếp `bus_bypass` ~10⁻³ và giải quyết chênh 52 lần giữa các seed; (b) thêm nhiễu/trễ cảm biến khi trời mưa và đo xem `crosswalk_block` tăng bao nhiêu; (c) săn `cut_in` và `rear_ended` với điểm nguy hiểm loại trừ vùng `bus_bypass` (ví dụ ép C2 ở x > 112 m) để biết chúng có thật sự hiếm hay chỉ bị che khuất.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** một **người đi bộ băng ngang ngay sau đuôi xe buýt đang đỗ** (`hazard.json`: thẻ `ped`, phía `S`, x = 92 m, v = 2,2 m/s, bắt đầu đi khi xe ta còn cách `trig` = 19 m). Ngoài đời: khách vừa xuống xe buýt, không đi tới vạch qua đường ở x = 122 m mà vòng ra sau đuôi xe (buýt chiếm khoảng x 100–112 m) để sang đường. Đây là tình huống kinh điển trong các vụ tai nạn người đi bộ ở trạm buýt.
- **Căn cứ:** đọc `avsim/core.py`, logic dự đoán người đi bộ của planner (`pedBuffer`, `yieldp`) chỉ áp dụng cho **P1 ở vạch qua đường** (`WALK_X`). Người băng ngang ở chỗ khác bị xe buýt che cho đến khi bước vào làn, và chỉ được xử lý khi đã trở thành "vật phía trước" trong làn của xe ta, tức là quá muộn. `cautionV` chỉ làm xe ta chậm lại vì vạch bị che, không vì người đi bộ này.
- **Giả định (`hazard.json: why`, `space`, `exposure`):**
  - `space`: `h0_x` ∈ [90, 100] m (khoảng ngay sau đuôi xe buýt), `h0_trig` ∈ [10, 30] m (người băng ngang chấp nhận khe 1,5–4 s trước xe đang chạy chậm ~7 m/s gần trạm, theo các nghiên cứu gap acceptance của người đi bộ), `h0_v` ∈ [1,0; 2,5] m/s (đi bình thường 1,2–1,5 m/s theo Knoblauch et al. 1996, vội tới 2,5 m/s là giới hạn khả thi của lab).
  - `exposure` = **0,02 mỗi chuyến** ≈ 20 % chuyến đô thị đi qua một xe buýt đang đón trả khách × ~10 % số lần có người băng qua sau đuôi xe (NHTSA Traffic Safety Facts: phần lớn tử vong người đi bộ xảy ra ngoài giao lộ; TCRP Report 125 về an toàn người đi bộ quanh trạm buýt). Đây là ước lượng thô, có thể sai vài lần.
  - `world` = town: quét trên giao thông của thị trấn B, không phải chỉ cảnh tôi dựng.
- **Kết quả** (`mine.json`): cảnh dựng chính xác: `exact.planner` = [ego–H0 lúc 13,65 s]. `accepted` = **0,993** (1 490/1 500 khả thi; 10 bị loại bởi `is_feasible`, cụ thể là cutDur quá ngắn trên đường ướt).

| `mine.json` | policy gốc (`baseline`) | bản sửa (`chosen`) |
|---|---|---|
| n | 1 490 | 1 490 |
| va chạm của xe ta (`crashes`) | 77 | 64 |
| **`avoidable`** (tránh được bằng phanh hết cỡ ngay khi thấy) | **74** | **63** |
| `unavoidable` | 0 | 0 |
| `unattributed` (va chạm thường của thị trấn) | 3 | 1 |
| `low_speed` (< 1 m/s) | 3 | 2 |
| `avoidable_foreknown` | 74 | 63 |
| P(lỗi planner \| rắc rối) `p` [Wilson 95 %] | **4,97·10⁻²** [3,97·10⁻²; 6,19·10⁻²] | **4,23·10⁻²** [3,32·10⁻²; 5,37·10⁻²] |
| tần suất mỗi chuyến `per_trip` (cận `per_trip_hi`) | **9,93·10⁻⁴** (1,24·10⁻³) | 8,46·10⁻⁴ (1,07·10⁻³) |

So với lách xe buýt: `mc.json` cho `bus_bypass` = 1,0·10⁻³ mỗi lần đi qua đoạn phố (trung vị IS 9,5·10⁻⁴). Rắc rối của tôi, ở `exposure` 0,02, góp **~1,0·10⁻³ mỗi chuyến**: **cùng cỡ** với kiểu thất bại lớn nhất mà cuộc săn ở bước C tìm được, và bản sửa ở bước E chỉ giảm nó ~15 % (74 → 63), trong khoảng Wilson chồng nhau, tức gần như **không cứu**.

  Phân biệt các loại: **`avoidable`** là va chạm mà, nếu phanh hết cỡ từ khung hình đầu tiên H0 vừa bắt đầu đi **và** cảm biến xe ta nhìn thấy nó, thì không va chạm: đó là lỗi của planner. **`unavoidable`** = 0: không phiên bản nào của rắc rối tôi dựng là "trò ăn gian" mà không ai tránh nổi. **`unattributed`** (3 và 1) là va chạm thường của thị trấn xảy ra trước khi người băng ngang bắt đầu đi, không nói gì về rắc rối này. `low_speed` (3 và 2) đếm va chạm khi xe ta dưới 1 m/s, tức bị đâm khi gần như đứng yên; chúng vẫn nằm trong loại của mình (ví dụ với bản sửa, `chosen.examples` có một vụ `avoidable` ở 0,23 m/s: xe ta gần dừng nhưng vẫn chạm H0). `avoidable_foreknown` = `avoidable`: phanh từ lúc H0 bắt đầu đi (dù còn bị che) không cứu thêm vụ nào, vì xe ta thấy H0 ngay khi H0 bước ra khỏi đuôi xe buýt, và vẫn còn ~1 s ở ~5 m/s (ví dụ đầu tiên trong `baseline.examples`: H0 bắt đầu 10,55 s, thấy được 12,25 s, va chạm 13,3 s ở 5,2 m/s). Planner có thời gian, nhưng không coi H0 là nguy hiểm.

  **"Tránh được" là chuẩn lạc quan theo cả hai hướng:** (1) người phanh "oracle" biết ngay rằng H0 sẽ đi vào làn và phanh tối đa không chần chừ, không có thời gian phản ứng hay do dự để phân biệt người đứng chờ với người sắp bước ra, nên `avoidable` **đếm nhiều hơn** số vụ một planner thực tế tránh được (là cận trên của lỗi planner theo nghĩa này); (2) ngược lại, chỉ thử phanh, không thử đánh lái hay vừa phanh vừa lách, nên một số vụ đang bị xếp `unavoidable` có thể tránh được, tức nó **đếm ít đi** ở chiều kia (ở đây `unavoidable` = 0 nên chiều này không ảnh hưởng).
- **Hình:** `evidence/shots/q9_mine_GB-01.png` (GB-01, chương 9, rắc rối của bạn): va chạm H0 lúc 13,3 s dưới policy gốc; bóng mờ là bản sửa, cũng va chạm lúc 13,6 s.
- **Bạn tự lái thì sao:** tôi không ghi lại một lần tự lái trong chương 10 (`hazard.json` không có `egoAcc`, `mine.json: exact` không có trường `you`), nên không có số. Với ~1 s và ~5 m/s, một người lái chú ý có thể dừng kịp (quãng phanh ở μ = 0,8 chỉ ~1,6 m cộng ~1,5 m quãng phản ứng), nhưng người lái thật sẽ làm điều planner không làm: thấy xe buýt đang trả khách thì **đi chậm hẳn và nhìn đuôi xe buýt**, không chỉ nhìn vạch.
- **Cách sửa:** không núm nào trong `POLICY` sửa được triệt để: `cautionV`/`cautionRange` chỉ phụ thuộc khoảng cách tới **vạch** (bản sửa cautionV 7,0 chỉ giảm 15 %, và cautionV thấp hơn làm `bus_bypass` tăng, câu 6), còn `pedBuffer` chỉ áp dụng cho P1. Planner cần **logic mới**: (1) dự đoán đường đi cho **mọi** người đi bộ nhìn thấy, không chỉ người ở vạch: nếu thời điểm họ vào làn của xe ta (khoảng cách ngang / tốc độ) sớm hơn thời điểm xe ta tới chỗ đó cộng một biên an toàn thì nhường như với P1; (2) một vùng thận trọng quanh **xe buýt đang đỗ trả khách** (không chỉ quanh vạch): giới hạn tốc độ sao cho có thể dừng trong khoảng cách tới đuôi/đầu xe buýt, giống luật "đi chậm khi qua xe buýt trường học"; (3) coi một người đi bộ vừa xuất hiện từ vùng bị che, đang di chuyển ngang về phía làn, là chướng ngại ưu tiên cao (phanh mạnh ngay thay vì đợi họ vào làn).
