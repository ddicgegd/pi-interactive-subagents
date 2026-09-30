# Đánh giá Kiến trúc Kỹ thuật: pi-interactive-subagents

Tài liệu phân tích và đánh giá chuyên sâu về mô hình kiến trúc, ưu nhược điểm kỹ thuật và tính khả thi trong thực chiến của **`pi-interactive-subagents`**.

---

## 1. Kiến trúc Cốt lõi (Core Architecture)

Khác với các framework đa tác nhân (multi-agent framework) thông thường chạy in-process async loops hoặc container hóa (Docker/K8s), `pi-interactive-subagents` lựa chọn mô hình **OS Process + Terminal Multiplexer (tmux)**:

1. **Phân tách tiến trình hoàn toàn (Full Process Isolation):**
   - Mỗi sub-agent được khởi chạy dưới dạng một tiến trình CLI độc lập trên hệ điều hành (`pi` hoặc `claude` CLI).
   - Không chia sẻ memory heap, V8 runtime hay event loop với tiến trình orchestrator chính. Lỗi panic/crash hoặc rò rỉ bộ nhớ ở sub-agent không làm gián đoạn phiên làm việc chính.

2. **Tmux làm GUI & Process Manager:**
   - Thay vì tự xây dựng giao diện terminal phức tạp (TUI) hoặc server-client dashboard, dự án tận dụng trực tiếp các pane của tmux làm bề mặt hiển thị.
   - Sub-agent chạy trong một split pane bên phải (`even-horizontal`), không cướp tiêu điểm bàn phím (`focus`) của người dùng.
   - Người dùng có thể quan sát trực tiếp các lệnh bash, kết quả tool call theo thời gian thực và tương tác bằng phím nếu cần (Human-in-the-loop).

3. **Giao tiếp bất đồng bộ qua Steer Notifications (Async IPC):**
   - Lệnh gọi `subagent()` trả về ngay lập tức (non-blocking).
   - Khi sub-agent hoàn tất lượt làm việc, kết quả cuối cùng được chuyển hướng (steer) trở lại phiên chính dưới dạng một thông báo kích hoạt turn mới của orchestrator.
   - Cơ chế hai chiều: Sub-agent có thể tạm dừng (`waiting`) để hỏi orchestrator thông qua công cụ `ask_question`.

4. **Loadout Snapshot & Sandbox Bất biến (Immutable Resume):**
   - Tại thời điểm khởi tạo, toàn bộ cấu hình thực thi (tool allowlist, extension, model, thinking level, system prompt, allowed targets, cwd) được snapshot thành `<session>.loadout.json`.
   - Khi tiếp tục phiên (`subagent_message`), sandbox gốc được phục hồi nguyên trạng từ snapshot, ngăn chặn nguy cơ leo thang đặc quyền (privilege escalation).

---

## 2. Điểm Sáng Kỹ thuật (Technical Strengths)

- **Kiểm soát truy cập Tool chặt chẽ (Strict Whitelist Tool Access):**
  - Mọi sub-agent đều được khởi chạy với cờ `--no-extensions` và `--tools <allowlist>`. Chỉ các extension hỗ trợ trực tiếp các tool được khai báo mới được nạp vào child process.
  - Phân quyền theo mô hình phân tầng: Sub-agent chỉ được phép sinh thêm tác nhân con nếu khai báo rõ ràng trong trường `subagent_agents`. Không có cơ chế spawn tùy tiện.
- **Quản lý Vòng đời & Trạng thái Rõ ràng (State Machine & Auto-exit):**
  - Cơ chế `auto-exit: true` tự động dọn dẹp và trả về tóm tắt khi hoàn thành turn.
  - Tự động hoãn thoát (park ở trạng thái `waiting`) khi có yêu cầu đang chờ xử lý (`ask_question` chưa được phản hồi, hoặc các child sub-agent cấp dưới đang chạy dở dang). Tránh triệt để lỗi kết thúc non hoặc deadlock.
- **Khả năng quan sát tự nhiên (Zero-friction Observability):**
  - Trực quan hóa tiến độ bằng thanh trạng thái widget (`starting`, `active`, `waiting`, `stalled`, `running`) kết hợp quan sát trực tiếp màn hình thực thi của agent.
- **Khôi phục trạng thái linh hoạt (Resilient Messaging):**
  - `subagent_message` tự động phân nhánh: nếu agent đang chạy, nó inject thông điệp vào lượt tiếp theo; nếu agent đã kết thúc, nó phục hồi lại phiên cũ với ngữ cảnh loadout ban đầu.

---

## 3. Rủi ro, Giới hạn & Điểm Nghẽn (Risks & Bottlenecks)

### 3.1. Phụ thuộc Terminal Emulation & IPC Kém Ổn Định
- **Giao tiếp qua `tmux send-keys`:** Việc inject prompt hoặc thông điệp qua bàn phím ảo (flattened newlines) tiềm ẩn rủi ro khi shell gặp độ trễ, terminal buffer đầy hoặc gặp ký tự điều khiển đặc biệt.
- **Heuristic Delay:** Sự tồn tại của biến môi trường `PI_SUBAGENT_SHELL_READY_DELAY_MS` (mặc định 500ms, khuyến nghị tăng lên 2500ms nếu shell khởi động chậm) phản ánh việc chờ đợi dựa trên phỏng đoán thời gian thay vì cơ chế bắt tay (handshake/IPC socket/pipe) xác định.

### 3.2. Xung đột Tài nguyên Disk & Git Concurrency
- Khi spawn nhiều sub-agent có quyền thao tác file (`write`, `edit`, `bash`) trong cùng thư mục dự án (`cwd` chung):
  - Dễ gặp xung đột khóa Git (`.git/index.lock`).
  - Ghi đè file của nhau tạo race condition trên hệ thống tệp.
  - Dự án chưa tích hợp sẵn cơ chế cô lập workspace tự động (chẳng hạn như `git worktree`).

### 3.3. Giới hạn Không gian Hiển thị của Tmux
- Khi chạy song song từ 4-6 agents trở lên, layout `even-horizontal` hoặc `tiled` sẽ chia nhỏ màn hình terminal thành các ô quá hẹp, dẫn đến wrap chữ liên tục, làm vỡ hiển thị widget và khó đọc nội dung log.

### 3.4. Chi phí Context & Token Burn
- Mỗi sub-agent mang theo snapshot system prompt, tool schemas và có thể kế thừa lịch sử nếu dùng chế độ `fork`. Việc sinh các agent phân cấp sâu (deep nesting) dễ làm tăng đột biến lượng token tiêu thụ và chi phí API nếu không được kiểm soát chặt chẽ.

---

## 4. Đánh Giá Thực Chiến & Khuyến Nghị (Production Recommendations)

### Trường hợp Sử dụng Tối ưu (Best Fit)
- **Codebase Reconnaissance:** Giao cho agent chuyên biệt (như `scout`) quét và lập chỉ mục codebase, tra cứu tài liệu trong khi lập trình viên tiếp tục luồng suy nghĩ chính.
- **Tác vụ Web & Research Độc lập:** Chạy `researcher` thu thập tài liệu bên ngoài mà không ảnh hưởng tới ngữ cảnh lập trình chính.
- **Supervised Development:** Phù hợp cho lập trình viên muốn quan sát chi tiết từng thao tác terminal của agent trên màn hình lớn.

### Trường hợp Nên Tránh (Unsuitable)
- **Hệ thống Batch/Headless CI/CD:** Các môi trường tự động hóa không có màn hình tương tác hoặc không hỗ trợ tmux session ổn định.
- **Parallel Code Modification quy mô lớn:** Nhiều agent cùng can thiệp sâu vào source code mà thiếu cơ chế cô lập nhánh (branch) hoặc Git worktree riêng biệt.

### Đề xuất Cải tiến Kỹ thuật
1. **Worktree Isolation:** Bổ sung flag tự động tạo temporary `git worktree` cho mỗi sub-agent thực thi tác vụ ghi mã nguồn.
2. **Robust IPC Handshake:** Bổ sung cơ chế IPC dựa trên Unix Domain Socket hoặc Named Pipe thay vì dựa hoàn toàn vào `tmux send-keys` và heuristic delay.
