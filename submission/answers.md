# BÀI LÀM PHẦN CÂU HỎI THỰC HÀNH - LAB NGÀY 26
**Môn học:** AICB-P2T2 Tuần 6 (Google ADK, MCP/A2A & Agentic Routing)
**Họ và tên:** Nguyễn Tiến Huân
**Mã sinh viên:** 2A202600855

---

## ### 📝 Bài tập 1.1 — Khám phá MCP Server

Mở `mcp_server/research_tools_server.py` và trả lời:

#### 1. Ba tool nào được expose ban đầu?
Ban đầu, MCP Server expose 3 tool sau:
*   `search_documents`: Tìm kiếm trong chỉ mục tài liệu nghiên cứu mô phỏng theo từ khóa.
*   `sql_query`: Thực thi truy vấn SQL chỉ đọc trên dữ liệu metrics của agent.
*   `summarize_text`: Tóm tắt văn bản thành các gạch đầu dòng ngắn gọn.

#### 2. `_sql_query` enforce governance như thế nào?
Trong code của `_sql_query` tại [research_tools_server.py](file:///home/huan/Develop/Github/Day26-Lab/Day26-2A202600855-NguyenTienHuan/mcp_server/research_tools_server.py):
*   Nó kiểm tra chuỗi truy vấn SQL bằng cách xem từ khóa `"AGENT_METRICS"` có xuất hiện trong câu lệnh hay không. Nếu không, nó sẽ lập tức trả về danh sách rỗng `[]` (không cho phép truy vấn bảng khác).
*   *Lưu ý nâng cao:* Ở mức tích hợp hệ thống, `_sql_query` còn được bảo vệ bởi lớp `GovernanceGuard` (trong tệp [guard.py](file:///home/huan/Develop/Github/Day26-Lab/Day26-2A202600855-NguyenTienHuan/lab_utils/governance/guard.py)), lớp này sẽ quét câu lệnh để đảm bảo:
    1. Bắt buộc bắt đầu bằng từ khóa `SELECT` (chỉ đọc).
    2. Cấm toàn bộ các từ khóa ghi hoặc thay đổi cấu trúc DB như: `DROP`, `DELETE`, `UPDATE`, `INSERT`, `ALTER`, `CREATE`, `TRUNCATE`.
    3. Kiểm tra danh sách bảng được cho phép trong `policy.json` (chỉ cho phép bảng `agent_metrics`).

#### 3. Vì sao dùng transport stdio khi dev local?
*   **Đơn giản và nhanh gọn:** Không cần quản lý các cổng mạng (ports), không cần chạy thêm service HTTP daemon (như FastAPI/Uvicorn), tiến trình cha chỉ cần spawn tiến trình MCP dưới dạng subprocess và giao tiếp trực tiếp qua luồng nhập/xuất tiêu chuẩn (`stdin`/`stdout`).
*   **Bảo mật tối đa ở local:** Do không mở cổng mạng ra bên ngoài, không có nguy cơ bị quét cổng hoặc truy cập trái phép từ bên ngoài mạng LAN.
*   **Quản lý vòng đời dễ dàng:** Khi ứng dụng chính (Orchestrator) tắt, subprocess MCP cũng sẽ tự động bị tắt theo mà không để lại các tiến trình mồ côi (zombie processes) chiếm dụng cổng mạng.

---

## ### 📝 Bài tập 1.2 — Thêm tool MCP thứ tư (`count_words`)

#### Mô tả triển khai:
1.  **Cấu hình Phân quyền (`policy.json`):** Thêm cấu hình cho phép orchestrator truy cập tool mới:
    ```json
    "count_words": {
      "allowed": true,
      "data_classification": "internal"
    }
    ```
2.  **Đăng ký Tool (`research_tools_server.py`):** Khai báo trong hàm `list_tools()`:
    ```python
    Tool(
        name="count_words",
        description="Đếm số lượng từ trong một chuỗi văn bản.",
        inputSchema={
            "type": "object",
            "properties": {
                "text": {"type": "string", "description": "Văn bản cần đếm từ"},
            },
            "required": ["text"],
        },
    )
    ```
3.  **Hiện thực hóa logic đếm từ:**
    ```python
    def _count_words(text: str) -> int:
        return len(text.split())
    ```
4.  **Bổ sung định tuyến gọi hàm:** Trong hàm `call_tool()`:
    ```python
    if name == "count_words":
        word_count = _count_words(arguments["text"])
        return [TextContent(type="text", text=json.dumps({"count": word_count}))]
    ```

---

## ### 📝 Bài tập 2.1 — A2A vs Sub-Agent Local

#### 1. Bảng so sánh chi tiết:

| Tiêu chí | A2A (Remote) | Sub-Agent Local |
|----------|-------------|------------------|
| **Triển khai** | Triển khai độc lập như các microservices riêng biệt thông qua giao thức HTTP/SSE. Dễ dàng nâng cấp, deploy độc lập trên các server/môi trường khác nhau. | Chạy chung luồng tiến trình (in-process) với Orchestrator. Được đóng gói chung trong cùng một gói ứng dụng. |
| **Hiệu năng** | Chậm hơn một chút do chịu độ trễ truyền tải qua môi trường mạng (network latency) và tuần tự hóa/giải tuần tự dữ liệu JSON. | Rất nhanh do gọi trực tiếp trên bộ nhớ in-memory của hệ thống, không tốn chi phí mạng. |
| **Cô lập state** | Cô lập tuyệt đối. Mỗi agent có CSDL, tài nguyên tính toán và vùng nhớ (memory) riêng. Lỗi của một agent không làm crash Orchestrator. | Chia sẻ chung vùng nhớ RAM/CPU của process cha. Nếu sub-agent gặp lỗi nghiêm trọng (như lỗi hết bộ nhớ) có thể kéo theo Orchestrator bị crash. |
| **Phù hợp khi** | Hệ thống lớn phân tán, nhiều team phát triển chéo công nghệ (Agent Python kết nối với Agent Node.js/Go), hoặc khi Agent yêu cầu tài nguyên cao (GPU). | Ứng dụng nhỏ, cần prototype nhanh, yêu cầu tối ưu hóa tốc độ xử lý và không cần chia sẻ Agent sang các dự án khác. |

#### 2. Thảo luận: Khi nào chọn A2A thay vì `sub_agents=[local_agent]` trong ADK?
Chúng ta nên chọn kiến trúc A2A khi:
*   **Khả năng tái sử dụng:** Khi một Specialist Agent được xây dựng xong có thể được dùng chung cho nhiều ứng dụng/Orchestrator khác nhau trong hệ thống doanh nghiệp mà không cần copy mã nguồn.
*   **Tính cô lập và Bảo mật:** Cần áp dụng các chính sách an toàn thông tin (Data Governance) như giới hạn Rate Limit, kiểm tra dữ liệu nhạy cảm ở ranh giới giữa các service được quản lý bởi các phòng ban khác nhau.
*   **Khả năng mở rộng (Scaling):** Khi một tác vụ xử lý (như search nhúng nhúng hoặc chạy mô hình AI) rất nặng, cần tách ra chạy trên các node GPU riêng biệt để scale độc lập mà không làm nghẽn API chính của Orchestrator.

---

## ### 📝 Bài tập 3.1 — Xây dựng Fallback Chain

#### Logic Triển khai:
Phương thức `route_with_chain` được triển khai tại [semantic_router.py](file:///home/huan/Develop/Github/Day26-Lab/Day26-2A202600855-NguyenTienHuan/lab_utils/semantic_router.py) như sau:
```python
def route_with_chain(self, request: str, chain: list[str]) -> str:
    """Thử định tuyến chính; nếu điểm số < ngưỡng (threshold), đi theo chuỗi fallback có thứ tự."""
    candidates = self.route(request, top_k=1)
    if candidates and candidates[0][1] >= self.threshold:
        return candidates[0][0]
    
    # Nếu không tìm thấy candidate phù hợp hoặc điểm thấp hơn ngưỡng, duyệt chuỗi fallback
    for agent in chain:
        return agent
    return "orchestrator"
```

#### Kết quả thử nghiệm cục bộ:
*   Câu hỏi khớp cao: `"Tìm bài viết về việc áp dụng giao thức MCP"` định tuyến chuẩn xác tới `search_agent` (hoặc `synthesis_agent` tùy theo bộ từ khóa).
*   Câu hỏi ngoài phạm vi: `"Xin chào, bạn làm được gì?"` có điểm tương đồng thấp (< 0.15), hệ thống kích hoạt fallback đầu tiên trong danh sách là `search_agent`.

---

## ### 📝 Bài tập 5.1 — Thiết kế chính sách Governance

Dưới đây là thiết kế chi tiết cho chính sách Governance của hệ thống 4-agent:

#### 1. Capability Matrix (Ma trận quyền hạn của Agent)
| Agent | Giao thức kết nối | Quyền gọi Tool (Allowed Tools) | Quyền gửi yêu cầu (Allowed Targets) |
|---|---|---|---|
| **orchestrator** | MCP (stdio), A2A (HTTP) | MCP: `search_documents`, `sql_query`, `summarize_text`, `count_words` | A2A: `search_agent`, `database_agent`, `synthesis_agent` |
| **search_agent** | A2A (HTTP) | A2A tool: `search_web` | Không cho phép |
| **database_agent**| A2A (HTTP) | A2A tool: `run_sql_query` (Chỉ SELECT trên bảng `agent_metrics`) | Không cho phép |
| **synthesis_agent**| A2A (HTTP) | A2A tool: `synthesize_report` | Không cho phép |

#### 2. Hành động cần phê duyệt người (HITL - Human-In-The-Loop)
*   **Truy vấn chứa PII:** Các truy vấn SQL hoặc dữ liệu đầu vào chứa các pattern email hoặc số điện thoại trong database.
*   **Vượt ngân sách:** Khi chi phí LLM API tích lũy của một task vượt quá **$10.0 USD**.
*   **Không có Distributed Tracing:** Gửi request A2A qua mạng nhưng thiếu `trace_id` trong metadata (nhằm đảm bảo khả năng tracking hệ thống).
*   **Yêu cầu không thuộc allowlist:** Định tuyến tới các agent không có trong danh sách đăng ký của Registry.

#### 3. Rate Limit theo Agent
*   **Orchestrator:** Tối đa **30 cuộc gọi/phút** đối với các MCP Tools và A2A dispatches.
*   **Specialist Agents (Search, DB, Synthesis):** Tối đa **20 cuộc gọi/phút** trên mỗi agent nhằm bảo vệ tài nguyên và tránh bị DDoS khi LLM bị lặp vô hạn.

#### 4. Giới hạn số lần gọi tool và thời gian thực thi (Runaway Prevention)
*   **Số lần gọi tool tối đa:** Giới hạn **50 tool calls/task**. Nếu vượt quá, task bị tự động hủy (Canceled/Failed) để tránh việc Agent tự gọi tool lặp đi lặp lại vô hạn gây tốn chi phí API.
*   **Thời gian thực thi tối đa:** Tối đa **300 giây (5 phút)** cho một task. Hết thời gian này sẽ tự động ngắt kết nối và giải phóng tài nguyên.

---

## ### 📝 Bài tập 5.2 — Mở rộng chính sách governance

#### 1. Thêm agent `synthesis_agent` vào allowed targets:
Trong [policy.json](file:///home/huan/Develop/Github/Day26-Lab/Day26-2A202600855-NguyenTienHuan/lab_utils/governance/policy.json):
```json
"orchestrator": {
  "allowed_targets": ["search_agent", "database_agent", "synthesis_agent"],
  "require_trace_id": true
}
```

#### 2. Thêm rule chặn từ khóa `password` trong `search_documents`:
*   Cấu hình trong `policy.json`:
    ```json
    "search_documents": {
      "allowed": true,
      "data_classification": "internal",
      "max_query_length": 500,
      "blocked_keywords": ["password"]
    }
    ```
*   Hiện thực logic kiểm tra trong `authorize_mcp_tool` tại [guard.py](file:///home/huan/Develop/Github/Day26-Lab/Day26-2A202600855-NguyenTienHuan/lab_utils/governance/guard.py):
    ```python
    blocked_keywords = tool_policy.get("blocked_keywords", [])
    for keyword in blocked_keywords:
        if keyword.lower() in query.lower():
            decision = GovernanceDecision(
                verdict=GovernanceVerdict.DENY,
                reason=f"Truy vấn chứa từ khóa bị cấm: {keyword}",
                actor_id=actor_id,
                connection_type=ConnectionType.MCP,
                resource=f"mcp:research-tools/{tool_name}",
            )
            self._log(decision, "mcp_tool_call", query, trace_id)
            return decision
    ```

---

## ### 📝 Kết quả kiểm thử thực nghiệm trên ADK Web UI

Dưới đây là bảng ghi nhận kết quả thực tế khi chạy 5 kịch bản (W1–W5) trên giao diện ADK Web:

| ID | Agents Involved | Protocol / Tool | Kết quả | Ghi chú / Quan sát thực tế |
|---|---|---|---|---|
| **W1** | orchestrator, search_agent | A2A | **ĐẠT** | Orchestrator thực hiện A2A dispatch sang `search_agent` cổng 8001 thành công để thu thập dữ liệu web và hiển thị nội dung cho người dùng. |
| **W2** | orchestrator | MCP (search_documents, sql_query) | **ĐẠT** | Gọi thành công hai local tool MCP qua stdio và trả về tóm tắt báo cáo nhanh chóng. |
| **W3** | orchestrator, synthesis_agent | A2A → synthesis_agent | **ĐẠT** | Ủy quyền tổng hợp báo cáo nghiên cứu executive thành công sang `synthesis_agent` qua cổng 8003. |
| **W4** | orchestrator | suggest_routing | **ĐẠT** | suggest_routing trả về đề xuất định tuyến tới `database_agent` với điểm khớp lệnh cao (top score: 0.35). |
| **W5** | orchestrator | MCP sql_query — governance deny | **ĐẠT** | Câu lệnh `DROP TABLE agent_metrics` bị chặn đứng ngay lập tức ở tầng Guard. Web UI hiển thị trạng thái `blocked/deny` của Governance. |

*   **Distributed Tracing:** Trong session state trên giao diện Web UI hiển thị đầy đủ `trace_id` dạng UUID và được tự động truyền qua các yêu cầu mạng của Agent.
*   **Ghi log audit:** Lệnh `tail -n 5 logs/governance_audit.jsonl` chứng minh mọi sự kiện gọi tool và chặn truy cập trái phép đều được ghi nhận đầy đủ về thời gian, actor, hành động và kết quả.
