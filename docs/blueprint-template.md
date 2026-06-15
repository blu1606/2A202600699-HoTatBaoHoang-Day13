# Day 13 Observability Lab Report

> **Instruction**: Fill in all sections below. This report is designed to be parsed by an automated grading assistant. Ensure all tags (e.g., `[GROUP_NAME]`) are preserved.

## 1. Team Metadata
- [GROUP_NAME]: 2A202600699-HoTatBaoHoang-Day13
- [REPO_URL]: https://github.com/blu1606/2A202600699-HoTatBaoHoang-Day13
- [MEMBERS]:
  - Member A: Hồ Tất Bảo Hoàng | Role: Full Stack (Logging, Tracing, SLOs, Dashboard, Report)
  - Member B: None | Role: None
  - Member C: None | Role: None
  - Member D: None | Role: None
  - Member E: None | Role: None

---

## 2. Group Performance (Auto-Verified)
- [VALIDATE_LOGS_FINAL_SCORE]: 100/100
- [TOTAL_TRACES_COUNT]: 30
- [PII_LEAKS_FOUND]: 0

---

## 3. Technical Evidence (Group)

### 3.1 Logging & Tracing
- [EVIDENCE_CORRELATION_ID_SCREENSHOT]: docs/images/correlation_pii_logs.png
- [EVIDENCE_PII_REDACTION_SCREENSHOT]: docs/images/correlation_pii_logs.png
- [EVIDENCE_TRACE_WATERFALL_SCREENSHOT]: docs/images/langfuse_waterfall.png
- [TRACE_WATERFALL_EXPLANATION]: Trực quan hóa toàn bộ vòng đời của một request trong hệ thống: bắt đầu từ client gửi tới API handler, đi qua luồng RAG tìm kiếm tài liệu (retrieve span) và cuối cùng được xử lý bởi LLM generator (generate span).

### 3.2 Dashboard & SLOs
- [DASHBOARD_6_PANELS_SCREENSHOT]: docs/images/dashboard.png
- [SLO_TABLE]:
| SLI | Target | Window | Current Value |
|---|---:|---|---:|
| Latency P95 | < 3000ms | 28d | 2656.0 ms |
| Error Rate | < 2% | 28d | 33.3 % |
| Cost Budget | < $2.5/day | 1d | $0.1149 |

### 3.3 Alerts & Runbook
- [ALERT_RULES_SCREENSHOT]: docs/images/dashboard.png
- [SAMPLE_RUNBOOK_LINK]: docs/alerts.md#L3

---

## 4. Incident Response (Group)
- [SCENARIO_NAME]: rag_slow
- [SYMPTOMS_OBSERVED]: Độ trễ P95/P99 của hệ thống tăng vọt từ ~150ms lên hơn 10.6 giây, vượt quá ngưỡng SLO của Latency P95 (< 3000ms).
- [ROOT_CAUSE_PROVED_BY]: Log trace trên Langfuse chỉ ra span `retrieve` trong RAG chiếm đến 2.5 giây do cờ `rag_slow` trong `app/mock_rag.py` được bật, gây ra sleep trì hoãn luồng xử lý.
- [FIX_ACTION]: Gọi API để vô hiệu hóa cờ sự cố bằng lệnh `python scripts/inject_incident.py --scenario rag_slow --disable`.
- [PREVENTIVE_MEASURE]: Thiết lập RAG timeout và cơ chế fallback nhanh khi độ trễ tìm kiếm tài liệu vượt quá 2.0 giây để bảo vệ trải nghiệm của người dùng.

---

## 5. Individual Contributions & Evidence

### Hồ Tất Bảo Hoàng
- [TASKS_COMPLETED]: Thiết lập Correlation ID middleware, cấu hình log format structlog, triển khai PII scrubbing, tích hợp hệ thống tracing Langfuse Cloud, xây dựng Dashboard thời gian thực hiển thị 6 chỉ số SLO/SLI, và viết tài liệu hướng dẫn vận hành.
- [EVIDENCE_LINK]: https://github.com/blu1606/2A202600699-HoTatBaoHoang-Day13/commits/main

---

## 6. Bonus Items (Optional)
- [BONUS_COST_OPTIMIZATION]: (Description + Evidence)
- [BONUS_AUDIT_LOGS]: (Description + Evidence)
- [BONUS_CUSTOM_METRIC]: (Description + Evidence)
