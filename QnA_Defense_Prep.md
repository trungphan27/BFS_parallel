# 🎯 BỘ CÂU HỎI VÀ TRẢ LỜI BẢO VỆ PROJECT (Q&A CHEAT SHEET)
**Project:** Parallel Breadth-First Search (Hybrid MPI + OpenMP)
**Tài liệu tham khảo:** Báo cáo PDF, Outline, Guide, Criteria và Mã nguồn.

Tài liệu này tổng hợp toàn bộ các câu hỏi dự kiến (từ cơ bản đến nâng cao) mà giảng viên có thể hỏi trong buổi bảo vệ (Defense). Trả lời tốt các câu hỏi này sẽ giúp nhóm đạt trọn vẹn điểm cộng `+0.5 answer the teacher's question` và chứng minh tính `bold idea`.

---

## PHẦN 1: TỔNG QUAN VÀ ĐỘNG LỰC (MOTIVATION)

### Q1. Tại sao nhóm lại chọn đề tài Parallel BFS? Đề tài này có phải là một ý tưởng đột phá (bold idea) giúp thay đổi thế giới không?
**Trả lời:**
- Dạ thưa thầy/cô, Breadth-First Search (BFS) không chỉ là một thuật toán tìm kiếm đơn giản mà là **hạt nhân (building block)** nền tảng cho vô số ứng dụng phân tích Dữ liệu Lớn (Big Data) như: phân tích mạng xã hội (tính Betweenness Centrality), định tuyến mạng viễn thông, hay ráp nối bản đồ gen sinh học.
- Đề tài này thực sự mang tính đột phá vì dữ liệu đồ thị hiện nay lớn đến mức (hàng tỷ đỉnh) **không thể nạp vừa vào một máy tính đơn lẻ (Memory Wall)**. 
- Việc song song hóa BFS cực kỳ khó do tính chất **bất quy tắc (irregular)** của đồ thị và tỷ lệ tính toán trên giao tiếp rất thấp (low arithmetic intensity). Chính vì sự khó khăn này, Parallel BFS được chọn làm bài test chuẩn mực cho **Graph500** – bảng xếp hạng đánh giá sức mạnh của các siêu máy tính hàng đầu thế giới. Giải quyết tốt bài toán này là bước đệm để xử lý các siêu đồ thị làm nền tảng cho AI và khoa học dữ liệu trong tương lai.

### Q2. Hệ thống của nhóm cài đặt chạy trên môi trường nào?
**Trả lời:**
Dạ, nhóm cài đặt chạy trên một **Cluster phân tán gồm 3 máy tính vật lý** kết nối qua mạng LAN, với tổng cộng 8 nhân vật lý (Physical Cores). Trong đó:
- 1 máy Master có 4 cores.
- 2 máy Slaves, mỗi máy có 2 cores.
Hệ điều hành sử dụng là Ubuntu 24.04 LTS, thư viện Open MPI 4.1.6 và trình biên dịch GCC 13 (hỗ trợ OpenMP).

---

## PHẦN 2: KIẾN TRÚC VÀ PHƯƠNG PHÁP SONG SONG

### Q3. Tại sao nhóm lại sử dụng mô hình lập trình lai (Hybrid MPI + OpenMP) thay vì chỉ dùng một loại?
**Trả lời:**
- Kiến trúc phần cứng ngày nay đều phân cấp: một cụm nhiều máy (cluster), mỗi máy lại có nhiều CPU đa nhân.
- Bọn em dùng **MPI** để giải quyết bài toán **Distributed Memory** (giao tiếp và đồng bộ hóa trạng thái đồ thị giữa 3 máy vật lý thông qua mạng LAN).
- Đồng thời dùng **OpenMP** để giải quyết bài toán **Shared Memory** (tận dụng tối đa 4 nhân hoặc 2 nhân bên trong từng máy vật lý bằng các luồng - threads).
- Nếu chỉ dùng MPI, hệ thống sẽ sinh ra quá nhiều process trên 1 máy, gây tốn bộ nhớ và tăng chi phí overhead. Việc kết hợp Hybrid giúp giữ chi phí giao tiếp mạng thấp nhất trong khi vẫn tận dụng được toàn bộ phần cứng.

### Q4. Nhóm đang sử dụng loại song song nào? Data Parallelism hay Task Parallelism?
**Trả lời:**
Dạ, hệ thống áp dụng mô hình **Data Parallelism** (Song song hóa Dữ liệu) theo kiến trúc **SPMD** (Single Program, Multiple Data).
- Không có quá trình Master giao việc cho Slave (Task Parallelism).

### Q4b. Tại sao máy Master có 4 core, mà mỗi Slave lại chỉ cần 2 core? Mối quan hệ giữa Master và 2 Slaves thể hiện như thế nào trong quá trình chạy thuật toán (3 máy phối hợp ra sao)?
**Trả lời:**
- **Về cấu hình (Master 4 core, Slave 2 core):** Việc thiết lập này xuất phát từ điều kiện phần cứng vật lý thực tế mà nhóm có (1 laptop mạnh hơn làm Master, 2 laptop yếu hơn làm Slave). Tuy nhiên, môi trường bất đối xứng (heterogeneous) này lại là một bài test tuyệt vời để chứng minh sức mạnh của mô hình Hybrid: OpenMP có thể linh hoạt sinh ra 4 luồng trên Master và 2 luồng trên mỗi Slave để khai thác triệt để sức mạnh của từng máy, thay vì bắt tất cả phải chạy bằng cấu hình của máy yếu nhất.
- **Về mối quan hệ phối hợp giữa Master - Slave:** Mặc dù được đặt tên là Master và Slave, kiến trúc thuật toán Parallel BFS của nhóm là **Ngang hàng (Flat Peer-to-Peer)**, KHÔNG PHẢI kiểu Master "ngồi chỉ đạo" hay phân phát việc (Dispatch) cho Slave (Worker) làm.
  + Cả 3 máy đều tự nạp đồ thị, tự nhận dải đỉnh của riêng mình, và đồng loạt thực thi vòng lặp tìm kiếm song song (SPMD).
  + **Sự phối hợp:** Ở pha tính toán, 3 máy chạy hoàn toàn độc lập (mạnh ai nấy chạy). Nhưng khi kết thúc mỗi một cấp độ duyệt (Level), cả 3 máy bắt buộc phải "hội quân" tại một rào cản toàn cục (Global Barrier) thông qua lệnh `MPI_Allreduce`. Máy nào chạy nhanh xong trước (ví dụ Master) bắt buộc phải đứng chờ máy chậm hơn (Slaves). Tại rào cản này, 3 máy sẽ gộp chung mảng khoảng cách và tập biên (Frontier) của nhau lại, tạo thành một dữ liệu thống nhất, rồi tất cả mới cùng bước sang Level tiếp theo.
  + Máy Master (Rank 0) chỉ đóng vai trò "Master" ở đoạn đầu và cuối: in thông báo ra màn hình, và gom nhặt thông số thời gian của 2 máy kia để xuất file báo cáo tổng.


### Q5. Kỹ thuật phân rã dữ liệu (Decomposition) của nhóm là gì?
**Trả lời:**
Nhóm sử dụng **1D Block Partitioning** (Phân hoạch không gian 1 chiều tĩnh) kết hợp với nguyên tắc **Owner-Computes Rule**.
- Cụ thể: Tổng số đỉnh $N$ được chia đều cho $P$ tiến trình. Tiến trình thứ $r$ sẽ "sở hữu" liên tục các đỉnh từ $\lfloor N \cdot r / P \rfloor$ đến $\lfloor N \cdot (r+1) / P \rfloor$.
- Owner-Computes Rule có nghĩa là: Tiến trình nào sở hữu đỉnh nào thì tiến trình đó chịu trách nhiệm tính toán và cập nhật giá trị khoảng cách `dist[]` cho đỉnh đó.

### Q6. Tại sao nhóm lại nạp toàn bộ đồ thị (Replicate) vào bộ nhớ của tất cả các máy thay vì chia nhỏ mảng đồ thị ra?
**Trả lời:**
- Việc sao chép toàn bộ đồ thị (Graph Replication) lên mọi máy giúp **loại bỏ hoàn toàn chi phí giao tiếp Point-to-Point**. Khi một tiến trình đang duyệt đỉnh của nó và cần xem đỉnh kề, nó có thể tra cứu ngay lập tức trong RAM cục bộ mà không cần phải gửi tin nhắn `MPI_Recv` sang máy khác để xin dữ liệu.
- *Nhược điểm:* Giới hạn kích thước đồ thị lớn nhất có thể xử lý bị phụ thuộc vào dung lượng RAM của chiếc máy yếu nhất trong Cluster. Đối với Supercomputer thật, người ta sẽ phải chia nhỏ cả đồ thị (2D Partitioning), nhưng với Cluster nhỏ của bài tập này, kỹ thuật Replicate mang lại tốc độ code và độ ổn định cao nhất.

---

## PHẦN 3: THUẬT TOÁN VÀ TỐI ƯU (DIRECTION-OPTIMIZING)

### Q7. Cấu trúc dữ liệu dùng để lưu đồ thị trong bộ nhớ là gì? Tại sao không dùng Ma trận kề?
**Trả lời:**
Bọn em dùng định dạng **CSR (Compressed Sparse Row)** gồm 2 mảng contiguous 1 chiều là `row_ptr` và `adj`.
- Đồ thị R-MAT rất thưa (sparse), dùng Ma trận kề sẽ tốn $O(N^2)$ bộ nhớ và lãng phí RAM cho vô số số 0.
- CSR lưu trữ các đỉnh kề nhau liên tiếp trong bộ nhớ, cho phép truy cập với thời gian $O(1)$ và tối ưu hóa tối đa khả năng Spatial Locality của CPU Cache (rất thân thiện với bộ nhớ đệm).

### Q8. Thuật toán Direction-Optimizing (Heuristic của Beamer) là gì?
**Trả lời:**
Đó là một kỹ thuật đảo hướng duyệt đồ thị động cực kỳ thông minh:
- **Top-Down (Duyệt xuôi):** Đi từ tập biên (Frontier) bung ra các đỉnh lân cận. Hiệu quả khi Frontier rất nhỏ (ở những bước đầu hoặc cuối).
- **Bottom-Up (Duyệt ngược):** Thay vì đi từ Frontier, tiến trình quét toàn bộ các đỉnh chưa thăm, và dò ngược xem nó có nối với đỉnh nào trong Frontier không. 
- Nhờ kỹ thuật **Symmetry Breaking** trong Bottom-up, nếu phát hiện ra *DÙ CHỈ MỘT* lân cận nằm trong Frontier, thuật toán sẽ cập nhật khoảng cách và `break` (thoát vòng lặp) ngay lập tức, bỏ qua việc kiểm tra hàng nghìn cạnh còn lại. Phép tối ưu này giúp tiết kiệm hàng triệu phép tính vô ích khi Frontier bùng nổ quá lớn ở giữa đồ thị.

### Q9. Khi nào thuật toán quyết định đổi hướng duyệt?
**Trả lời:**
Quyết định dựa trên tổng số cạnh biên toàn cục (`scout_count`). Bọn em dùng `MPI_Allreduce(MPI_SUM)` để cộng gộp số cạnh biên của tất cả các máy. 
- Nếu tổng số cạnh biên > $\alpha$ lần số cạnh chưa thăm, thuật toán chuyển sang **Bottom-Up** (với $\alpha = 14$).
- Nếu kích thước Frontier thu nhỏ lại < $V / \beta$, thuật toán quay về **Top-Down** (với $\beta = 24$).

---

## PHẦN 4: GIAO TIẾP MẠNG VÀ CÂN BẰNG TẢI

### Q10. Trong quá trình chạy, 3 máy giao tiếp với nhau như thế nào?
**Trả lời:**
Hệ thống sử dụng chiến lược giao tiếp tập thể, đồng bộ và chặn (**Collective Blocking Topology**).
- Không dùng `MPI_Send` hay `MPI_Recv`.
- Khi kết thúc mỗi cấp độ duyệt (Level), hệ thống gọi `MPI_Allreduce(MPI_MAX)` để đồng bộ mảng khoảng cách `dist[]` và `MPI_Allreduce(MPI_BOR)` để đồng bộ mảng bitmap của Frontier.
- Các lệnh này đóng vai trò là một **Global Barrier** (Rào cản toàn cục), ép tất cả các máy phải chờ nhau hợp nhất dữ liệu rồi mới được chạy tiếp cấp độ (Level) sau.

### Q11. Thuật toán sinh đồ thị R-MAT có đặc điểm gì? Tại sao nó gây khó khăn cho Cân bằng tải (Load Balancing)?
**Trả lời:**
- Đồ thị R-MAT mô phỏng mạng xã hội thực tế, có tính chất **lệch bậc cực kỳ nghiêm trọng (power-law degree skew)**. Sẽ có một số ít đỉnh "hub" kết nối với hàng triệu đỉnh khác (như người nổi tiếng), còn đại đa số các đỉnh khác chỉ có vài kết nối.
- Do chia đỉnh tĩnh (1D Partitioning), tiến trình Rank 0 (chứa các đỉnh đầu tiên) trong thí nghiệm của nhóm phải gánh tới **55 triệu cạnh**, trong khi Rank 7 chỉ làm việc với **1.8 triệu cạnh** (Tỷ lệ workload 30:1). Điều này gây ra mất cân bằng tải vật lý.

### Q12. Bọn em giải quyết sự mất cân bằng đó như thế nào?
**Trả lời:**
- Ở **tầng MPI** giữa các máy: Em chia rải đều số đỉnh (Static).
- Ở **tầng OpenMP** bên trong từng máy: Bọn em sử dụng `#pragma omp for schedule(dynamic, chunk_size)`. Lập lịch động (Dynamic) giúp các luồng xử lý xong các đỉnh ít cạnh có thể tự động nhảy sang "gánh" phụ các đỉnh Hub nhiều cạnh, hạn chế được việc có luồng chạy rỗng (idle).

---

## PHẦN 5: KẾT QUẢ THỰC NGHIỆM VÀ ĐÁNH GIÁ (EVALUATIONS)

### Q13. Nhóm đánh giá Tính đúng đắn (Correctness) như thế nào?
**Trả lời:**
Bọn em chạy song song 2 phiên bản: một bản chạy Tuần tự (Sequential Baseline) và một bản chạy Phân tán trên Cluster. Sau đó so khớp mảng khoảng cách `dist[]` từng phần tử một. Dù đổi số lượng máy hay số luồng, kết quả vẫn báo `PASSED` tuyệt đối.

### Q14. Giải thích hiện tượng Speedup $S(P)$ tổng thể sụp đổ (crashing to near zero) khi tăng số máy/tiến trình?
**Trả lời:**
- Khi bọn em đo **Thời gian tính toán thuần $T_{comp}$** (đã trừ đi thời gian đợi mạng), hệ thống đạt độ tăng tốc $S'(P)$ gần như tuyến tính tuyệt vời (tăng 2.14 lần). Chứng tỏ thuật toán chia để trị rất hoàn hảo.
- Tuy nhiên, **Thời gian tổng $T_{total}$** lại bị thắt cổ chai trầm trọng do giao tiếp mạng ($T_{comm}$ chiếm 99.9% thời gian). 
- **Nguyên nhân:** Hàm `MPI_Allreduce` ở cuối mỗi vòng lặp yêu cầu các máy phải truyền mảng `dist[]` dung lượng $O(N)$ (hàng chục MB) qua mạng LAN. Qua 6 level duyệt, hệ thống phải bơm qua lại mạng LAN xấp xỉ 200 Megabyte dữ liệu. Tốc độ tính toán của CPU thì tính bằng mili-giây, nhưng độ trễ vật lý (latency) của mạng cáp quang LAN thì mất tới hàng chục giây. Do đó, mạng không theo kịp CPU.

### Q15. Nếu muốn hệ thống này chạy nhanh hơn nữa và Scale tốt hơn thì nhóm sẽ đề xuất cải tiến gì trong tương lai?
**Trả lời:**
1. **Phân rã theo Cạnh (Edge-balanced Partitioning) hoặc 2D Partitioning:** Thay vì chia số đỉnh bằng nhau, sẽ chia sao cho tổng số cạnh mỗi tiến trình bằng nhau. Tránh việc Rank 0 bị quá tải.
2. **Giao tiếp Bất đồng bộ (Asynchronous Communication):** Sử dụng `MPI_Iallreduce` (Non-blocking) để CPU có thể tranh thủ tính toán các đỉnh không phụ thuộc trong lúc chờ card mạng gửi nhận dữ liệu.
3. Chạy trên mạng chuyên dụng cho siêu máy tính như **InfiniBand** thay vì mạng Ethernet LAN thông thường để giảm thiểu tối đa độ trễ.

---

## PHẦN 6: KIẾN THỨC VẬN HÀNH THỰC TẾ CLUSTER (+0.5 SET UP CLUSTER IN 15 MINS)

### Q16. Làm thế nào để setup Cluster nhanh trong 15 phút bảo vệ?
**Trả lời:**
1. Bọn em sẽ cắm cáp LAN nối 3 laptop thông qua 1 Switch Gigabit (hoặc nối chung mạng Wifi nội bộ).
2. Dùng lệnh `ip a` xem IP. Vào file `sudo nano /etc/hosts` trên cả 3 máy để khai báo IP cho `node01, node02, node03`.
3. Khởi động dịch vụ chia sẻ file gốc **NFS server** trên Master và lệnh `sudo mount -a` trên 2 Slave.
4. Kiểm tra **SSH** bằng khóa RSA (bọn em đã config sẵn key public từ trước không cần pass).
5. Sửa file `hostfile` thành:
   `node01 slots=4`
   `node02 slots=2`
   `node03 slots=2`
6. Biên dịch lại code một lần trên master (`make`) và chạy lệnh `mpirun -np 8 --hostfile hostfile ./bfs_hybrid ...`. Mọi thứ đã sẵn sàng hoạt động.

### Q17. Lệnh `mpirun` hoạt động như thế nào khi chạy trên 3 máy? Làm sao nó biết máy nào để chạy?
**Trả lời:**
- Khi bọn em gõ lệnh trên máy Master, `mpirun` sẽ đọc file `--hostfile hostfile` để biết danh sách các node (máy) và số lượng `slots` (số tiến trình tối đa) trên từng máy. 
- Sau đó, `mpirun` tự động dùng giao thức **SSH** ngầm kết nối sang 2 máy Slave, và khởi tạo các tiến trình MPI trên đó sao cho tổng số tiến trình bằng đúng số `np` (ví dụ `-np 8`). 
- Khi các tiến trình đã được sinh ra trên 3 máy, chúng tự động liên lạc với nhau qua mạng LAN thông qua môi trường MPI để bắt đầu phối hợp tính toán.

### Q18. Thư mục chia sẻ NFS (Network File System) đóng vai trò gì trong Cluster của nhóm?
**Trả lời:**
- NFS giúp **đồng bộ mã nguồn và file thực thi** (`bfs_hybrid`) giữa tất cả 3 máy theo thời gian thực. 
- Bọn em chỉ cần lập trình và compile (`make`) **một lần duy nhất** trên máy Master, file thực thi sẽ ngay lập tức xuất hiện trên các máy Slave ở đúng đường dẫn đó. 
- Điều này giúp `mpirun` có thể ra lệnh cho Slave chạy file thực thi mà không báo lỗi "File not found". Nếu không có NFS, bọn em sẽ phải dùng lệnh `scp` copy thủ công file chạy sang 2 máy Slave mỗi khi sửa một dòng code, rất tốn thời gian.

### Q19. Quy trình chuẩn để kiểm tra kết nối 3 máy (Ping, NFS) trước khi chạy thuật toán là gì?
**Trả lời:**
Dạ thưa thầy/cô, quy trình chuẩn của bọn em gồm 3 bước:
- **Bước 1 (Mạng):** Gõ `ip a` xem IP, khai báo vào `/etc/hosts`. Dùng lệnh `ping node02` để đảm bảo mạng thông suốt.
- **Bước 2 (File System):** Trên Master kiểm tra file `/etc/exports` chứa subnet của mạng. Trên Slaves kiểm tra file `/etc/fstab` và chạy `sudo mount -a`. Dùng lệnh `ls` để xem file của Master đã hiện lên Slave chưa.
- **Bước 3 (MPI Connection):** Chạy lệnh test: `mpirun -np 12 --hostfile hostfile hostname`. Nếu màn hình in ra được hỗn hợp tên của cả 3 máy chủ thì có nghĩa là kết nối đã thành công mỹ mãn. Lúc này mới bắt đầu chạy code BFS thực sự.

### Q20. Khi nhóm chạy lệnh `OMP_NUM_THREADS=1 mpirun -np 8 --hostfile hostfile ./bfs_hybrid 500000 16 42`, ý nghĩa của các tham số là gì?
**Trả lời:**
- `OMP_NUM_THREADS=1`: Bọn em ép biến môi trường này để giới hạn mỗi tiến trình MPI chỉ sinh ra 1 luồng OpenMP (dùng để cô lập hiệu năng khi muốn test chế độ chạy thuần MPI).
- `-np 8`: Chạy tổng cộng 8 tiến trình (phân bổ theo hostfile: 4 cho Master, 2 cho Slave 1, 2 cho Slave 2).
- `./bfs_hybrid`: Tên file thực thi.
- `500000 16 42`: Các tham số truyền vào hàm sinh đồ thị R-MAT:
  + `500000`: Kích thước đồ thị $N = 500,000$ đỉnh (thực tế sẽ làm tròn lên lũy thừa của 2 là $2^{19} \approx 524,288$).
  + `16`: Scale factor. Số cạnh = $16 \times N$ (tương đương khoảng hơn 8 triệu cạnh).
  + `42`: Seed khởi tạo số ngẫu nhiên. Seed cố định giúp thuật toán R-MAT của cả 8 tiến trình trên 3 máy luôn sinh ra đồ thị **giống hệt nhau 100%**, đảm bảo tính tái lập (reproducibility) trong thực nghiệm.
