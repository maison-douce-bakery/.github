# Bakery Data Platform – Thiết kế giai đoạn 1 (bản sơ khai)

> **Trạng thái:** Đã chốt bản sơ khai, ngày 25/11/2026.
> **Phạm vi:** Giai đoạn 1: tiệm mới mở bán, dữ liệu rất ít, AI chỉ chủ tiệm dùng.
> **Bối cảnh:** Project giả lập để học. Hạ tầng BE/FE được coi là đã có sẵn (do developer khác xây và vận hành); RDS đã có dữ liệu giả.
> **Tài liệu liên quan trong project:**
> - `aws-compute-cho-data-workload.md`: cách chọn compute.
> - `aws-services-cho-data-engineer.md`: cheat sheet các service.
> - `aws-networking-lo-trinh.md`: mạng.

---

## Mục lục

1. [Nhu cầu](#1-nhu-cầu)
2. [Hiện trạng](#2-hiện-trạng)
3. [Tổng quan giải pháp](#3-tổng-quan-giải-pháp)
4. [Các quyết định kiến trúc và lý do](#4-các-quyết-định-kiến-trúc-và-lý-do)
5. [Danh sách tài nguyên](#5-danh-sách-tài-nguyên)
6. [Luồng dữ liệu và luồng AI](#6-luồng-dữ-liệu-và-luồng-ai)
7. [Bảo mật](#7-bảo-mật)
8. [Chi phí ước tính](#8-chi-phí-ước-tính)
9. [Việc phải kiểm tra trước khi triển khai](#9-việc-phải-kiểm-tra-trước-khi-triển-khai)
10. [Câu hỏi còn mở](#10-câu-hỏi-còn-mở)
11. [Khi nào phải vượt ra khỏi thiết kế này](#11-khi-nào-phải-vượt-ra-khỏi-thiết-kế-này)

---

## 1. Nhu cầu

### 1.1. Nhu cầu nghiệp vụ

| # | Nhu cầu | Ghi chú |
|---|---|---|
| N1 | Chủ tiệm xem báo cáo kinh doanh (doanh thu, chi nhánh, kênh bán, sản phẩm, tồn kho, hao hụt, thu chi…) **vào đầu ngày** | Dữ liệu chốt tới hết ngày hôm trước là đủ |
| N2 | Chủ tiệm **hỏi AI bằng ngôn ngữ tự nhiên** qua một web UI chat, kèm biểu đồ | Chỉ **một người dùng**. Nhân viên và khách hàng không dùng |
| N3 | Tích hợp thêm **một nguồn dữ liệu qua API** bên ngoài (giả lập) | Ngoài dữ liệu OLTP trong Postgres |
| N4 | Có **data catalog** (định nghĩa nghiệp vụ, lineage) để AI hiểu đúng dữ liệu | Marmot |
| N5 | Có **semantic layer** để AI truy vấn có kiểm soát | Cube |

### 1.2. Yêu cầu phi chức năng

| # | Yêu cầu | Mức chấp nhận |
|---|---|---|
| P1 | Độ tươi dữ liệu | Batch mỗi ngày lúc **05:00** giờ Việt Nam (dữ liệu T-1) |
| P2 | Độ trễ phản hồi AI | **Chậm được** (vài chục giây vẫn ổn) |
| P3 | Độ sẵn sàng | Không cần HA. Gián đoạn vài phút khi deploy hoặc restart là chấp nhận được |
| P4 | Ảnh hưởng tới OLTP | Tối thiểu: xử lý nặng dồn vào sáng sớm, trước giờ mở cửa |
| P5 | Chi phí | **Tối ưu cho quy mô nhỏ**, tận dụng free tier và hạ tầng sẵn có. **Không** thiết kế sẵn cho việc scale lớn |
| P6 | Công vận hành | Một Data Engineer vận hành; ưu tiên managed và serverless, không phải patch OS |
| P7 | Bảo mật | AI không được chạm bảng gốc; UI quản trị không mở ra internet |
| P8 | Hạ tầng dưới dạng code | Terraform, **một môi trường** (prod) |

---

## 2. Hiện trạng

### 2.1. Hạ tầng BE/FE có sẵn (giả lập, dùng chung)

| Thành phần | Trạng thái | Data platform dùng thế nào |
|---|---|---|
| AWS account, region **ap-southeast-1** | Có | Đặt mọi thứ cùng region |
| VPC, 2 public + 2 private subnet (2 AZ), Internet Gateway | Có | Đặt task ECS và EFS vào đây |
| RDS PostgreSQL (private subnet), đã có dữ liệu giả | Có | Làm nguồn OLTP **và** engine transform (dbt) |
| Domain + Route 53 hosted zone | Có | Thêm record `ai.<domain>` |
| Chứng chỉ ACM | Có | Dùng lại nếu là wildcard; nếu không thì xin thêm `ai.<domain>` (miễn phí) |
| ALB (listener 443) | Có | Thêm một listener rule theo host |
| ECR registry | Có | Thêm repo |
| GitHub Actions OIDC provider | Có | Dùng chung (mỗi account chỉ có một) |
| NAT Gateway | **Chưa rõ** | Xem [câu hỏi mở](#10-câu-hỏi-còn-mở). Thiết kế hiện tại **không cần NAT** |
| Cognito cho khách hàng | Có thể có | **Không** dùng chung; tạo user pool riêng |

### 2.2. Thành phần data platform đã có code

| Thành phần | Công nghệ | Cổng mặc định | Ghi chú |
|---|---|---|---|
| `assistant-ui` | Next.js 16 (standalone), Node 22 | 3000 | Giao diện chat; route `/api/agent` chuyển tiếp **phía server** tới agent, stream SSE. Biểu đồ Perspective chạy ngay trong trình duyệt |
| `bakery-agent` | Python 3.12, FastAPI, LangGraph, AG-UI | 8123 | LLM: **OpenAI** (`ChatOpenAI`). Gọi 3 MCP server. Lịch sử chat lưu trong RAM |
| Cube | Cube (semantic layer) | REST 4000, SQL API 15432 | Đọc các mart |
| Cube MCP server | Python MCP | 8001 | Dẫn tới Cube REST |
| DBHub | SQL MCP server | 8080 | 13 tool báo cáo dựng sẵn + `execute_sql` chỉ đọc, đi qua SQL API của Cube |
| Marmot | Data catalog, có MCP sẵn | 8080 | Lưu metadata trong Postgres |
| Dagu | Orchestrator | 8080 | Chạy 24/7, có UI |
| dbt | dbt-core + dbt-postgres | — | Transform ngay trong RDS |

Hiện mọi thứ chạy bằng docker-compose ở máy local.

---

## 3. Tổng quan giải pháp

**Ý tưởng chính:** dồn toàn bộ data platform vào **một ECS Fargate task** chạy 24/7. **Postgres có sẵn vừa là nguồn dữ liệu vừa là engine transform** (dbt chạy lúc 05:00 mỗi sáng). **Lambda** (free tier) lo phần gọi API bên ngoài. **ALB có sẵn** đưa web UI ra ngoài, có lớp đăng nhập Cognito.

```
Internet
  │
  ├─► Route 53 (ai.<domain>) ─► ALB có sẵn (443, ACM) ─► Cognito login
  │                                   │
  │                                   ▼ chỉ cổng 3000
  │   ┌──────────────── VPC có sẵn ─────────────────────────────────────────┐
  │   │ Public subnet                                                       │
  │   │  ECS Fargate task "data-platform" (1 task, 7 container, có public IP)│
  │   │   assistant-ui:3000 → bakery-agent:8123 ──► OpenAI API (internet)    │
  │   │                          │ MCP                                       │
  │   │            ┌─────────────┼──────────────┐                            │
  │   │       marmot:8081   dbhub:8082     cube-mcp:8001                     │
  │   │            │             │ SQL API        │ REST                     │
  │   │            │             └──► cube:15432/4000                        │
  │   │       dagu:8080 (+dbt) ──► Lambda invoke, SNS, S3 (Gateway Endpoint) │
  │   │            │ NFS                                                     │
  │   │ Private subnet                                                      │
  │   │       EFS (state Dagu)     RDS Postgres:                             │
  │   │                             app.public (BE) · app.raw · app.staging  │
  │   │                             · app.marts · database marmot            │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  └─ Lambda ingest (ngoài VPC) ─► External API ─► S3 datalake raw/
```

---

## 4. Các quyết định kiến trúc và lý do

| # | Quyết định | Đã cân nhắc | Lý do chọn |
|---|---|---|---|
| D1 | **Dùng dbt-postgres trên chính RDS OLTP**, ghi vào schema khác (`staging`, `marts`) trong **cùng database** | Warehouse riêng (Redshift), S3 + Athena, RDS analytics riêng | Dữ liệu rất ít; transform chạy lúc 05:00, trước giờ mở cửa, nên ít ảnh hưởng OLTP; không thêm hạ tầng. Phải cùng database vì Postgres không query chéo database được. dbt đọc thẳng bảng của BE (`source`), không cần bước Extract/Load cho dữ liệu OLTP |
| D2 | **Gộp 7 container vào 1 Fargate task**, chạy 24/7 | 7 ECS service riêng; EC2 tự quản; EKS | Chỉ trả một khối vCPU/RAM và **một public IPv4**. Khoảng $27/tháng, so với khoảng $45 trở lên nếu tách. Không phải patch OS. Đổi lại: deploy một container là restart cả task (gián đoạn khoảng 1 phút), chấp nhận được với một người dùng |
| D3 | **Fargate thay vì EC2** | EC2 `t4g` + Docker Compose | Giá gần như ngang nhau ở quy mô này, nhưng không phải vá OS, có self-healing và rolling deploy |
| D4 | **dbt chạy ngay trong container Dagu** | Lambda; ECS RunTask riêng | dbt-postgres chỉ gửi SQL, Postgres làm phần nặng. Task Dagu đã trả tiền theo kích thước; thêm RAM cho dbt chỉ khoảng $1,3/tháng. Lambda tiết kiệm được tối đa chừng đó nhưng vướng lỗi multiprocessing (`/dev/shm`), chỉ ghi được `/tmp`, timeout 30 phút, và cần đường tới secret trong VPC. RunTask thêm IAM, image và code polling không cần thiết |
| D5 | **Lambda (không gắn VPC) làm compute cho ingest API** | Dagu tự gọi API; Lambda trong VPC + NAT | Việc ngắn, chờ mạng, chạy theo lịch: hợp Lambda, nằm trong **free tier Always Free** (1 triệu request + 400.000 GB-giây/tháng). Lambda gắn VPC thì mất internet, cần NAT (~$33/tháng trở lên), nên để ngoài VPC |
| D6 | **S3 làm điểm trung chuyển** giữa Lambda và Postgres (lưu JSON thô) | Lambda trả dữ liệu thẳng cho Dagu | Lambda ngoài VPC không vào được RDS; Dagu thì vào được. S3 là chỗ cả hai cùng tới được mà không cần NAT. Thêm vào đó có **bản thô để chạy lại hoặc backfill**. Chi phí vài cent |
| D7 | **Không dùng NAT Gateway**: task ở public subnet, có public IPv4, security group **không mở inbound** (trừ cổng 3000 từ ALB) | NAT Gateway; interface endpoint | Public IP khoảng $3,65/tháng, so với NAT khoảng $33 hay 5 interface endpoint khoảng $35 trở lên. Không có inbound thì không ai từ ngoài vào được. Đây là đường ra cho ECR, Logs, SSM, Lambda API, SNS và OpenAI |
| D8 | **S3 Gateway Endpoint** cho VPC | — | Miễn phí. Traffic S3 từ trong VPC (kể cả layer image của ECR) không bao giờ phải đi qua NAT |
| D9 | **Web UI đi qua ALB có sẵn** (listener rule theo host) + **Cognito authenticate tại ALB** | ALB riêng (~$26/tháng gồm IPv4); Cloudflare Tunnel; mở thẳng public IP | Chi phí thêm gần như $0. App chưa có đăng nhập nên Cognito tại ALB là lớp bảo vệ bắt buộc. Không mở thẳng IP được vì IP của Fargate đổi sau mỗi lần restart và không gắn được Elastic IP |
| D10 | **UI quản trị (Dagu, Marmot, Cube) chỉ truy cập qua ECS Exec + SSM port forwarding** | ALB + đăng nhập; whitelist IP | Chỉ một Data Engineer dùng. $0, không mở port nào ra internet |
| D11 | **AI chỉ đọc schema `marts`, qua Cube/DBHub, bằng user chỉ đọc có `statement_timeout`** | Cho AI đọc thẳng bảng gốc | SQL do LLM sinh ra có thể rất tệ. Lớp semantic giới hạn phạm vi và bảo vệ OLTP |
| D12 | **State của Dagu trên EFS**; `desiredCount = 1`; khi deploy `minHealthy = 0% / max = 100%` | Ổ đĩa tạm của Fargate | Dagu lưu state ra file, ổ tạm mất khi restart. Chỉ được có **một** scheduler, nếu không job sẽ chạy hai lần |
| D13 | **SSM Parameter Store (SecureString)** cho secret | Secrets Manager | Bậc Standard miễn phí, đủ dùng, ECS inject trực tiếp vào container. Không dùng secret hay user master của BE |
| D14 | **Giữ OpenAI làm LLM** | Amazon Bedrock | Code và prompt hiện tại viết cho OpenAI. Có thể chuyển sang Bedrock sau (dùng IAM thay API key, tiền tính chung hóa đơn AWS) bằng `langchain-aws` |
| D15 | **Fargate ARM64 (Graviton)** | x86 | Rẻ hơn khoảng 20%. Điều kiện: mọi image đều có bản arm64 |
| D16 | **Một môi trường, Terraform, state trên S3** (`use_lockfile`) | Làm tay trên console; tách dev/prod | Tái tạo được, review được; không tốn gấp đôi |
| D17 | **Cảnh báo qua SNS → email** | Telegram | Đơn giản, nằm trong free tier |

---

## 5. Danh sách tài nguyên

### 5.1. Mạng và truy cập

| Tài nguyên | Mới hay dùng chung | Cấu hình |
|---|---|---|
| VPC, subnet, IGW, route table | Dùng chung | VPC phải bật DNS hostnames và DNS support |
| S3 Gateway Endpoint | Mới (nếu chưa có) | Gắn vào route table của public và private subnet |
| Public IPv4 cho task | Mới | `assignPublicIp = ENABLED` |
| ALB listener rule | Mới, trên ALB dùng chung | Host `ai.<domain>` → `authenticate-cognito` → forward tới target group. Priority không trùng rule của BE |
| Target group | Mới | Kiểu `ip`, cổng 3000, health check `/api/health` |
| Chứng chỉ ACM `ai.<domain>` | Mới nếu chứng chỉ của BE không phải wildcard | Gắn vào listener bằng `aws_lb_listener_certificate` |
| Route 53 record `ai.<domain>` | Mới | Alias trỏ tới ALB |
| Cognito user pool + app client | Mới | 1 user (chủ tiệm) |
| SG `sg-dp-task` | Mới | Inbound: 3000 chỉ từ SG của ALB. Outbound: mở hết |
| SG `sg-efs` | Mới | 2049 chỉ từ `sg-dp-task` |
| Rule thêm vào `sg-rds` | Mới, trên SG dùng chung | 5432 từ `sg-dp-task` |
| Rule thêm vào SG của ALB (nếu egress bị giới hạn) | Mới, trên SG dùng chung | Outbound tới `sg-dp-task:3000`; outbound 443 (để gọi endpoint token của Cognito) |

### 5.2. Compute

| Tài nguyên | Cấu hình |
|---|---|
| ECS cluster | Dùng chung cluster của BE hoặc tạo riêng (miễn phí) |
| ECS service `data-platform` | Fargate, `desiredCount = 1`, `minHealthy = 0`, `max = 100`, `enableExecuteCommand = true` |
| Task definition | **0,5 vCPU / 4 GB, ARM64** (chỉnh lại sau khi đo `docker stats`); dùng `dependsOn` + health check cho từng container |
| Container `assistant-ui` | Cổng 3000; `BAKERY_AGENT_URL=http://localhost:8123/agent` |
| Container `bakery-agent` | Cổng 8123; `CUBE_MCP_URL=http://localhost:8001/mcp`, `MARMOT_MCP_URL=http://localhost:8081/api/v1/mcp`, `DBHUB_MCP_URL=http://localhost:8082/mcp`; `OPENAI_API_KEY`, `MARMOT_API_KEY` lấy từ SSM |
| Container `cube` | REST 4000, SQL API 15432; user DB `cube_ro` |
| Container `cube-mcp` | Cổng 8001 |
| Container `dbhub` | Cổng **8082** (đổi khỏi 8080); trỏ tới `localhost:15432` |
| Container `marmot` | Cổng **8081** (đổi khỏi 8080); database `marmot` |
| Container `dagu` (đã cài dbt) | Cổng 8080; `DAGU_HOME` gắn EFS; múi giờ `Asia/Ho_Chi_Minh` |
| EFS | File system Standard, mount target ở 2 AZ, access point cho Dagu |
| Lambda `ingest-<api>` | Python, arm64, không gắn VPC, RAM 256–512 MB, timeout 5–10 phút |

### 5.3. Dữ liệu

| Tài nguyên | Cấu hình |
|---|---|
| S3 `<prefix>-datalake` | Block Public Access, SSE-S3, versioning; prefix `raw/<api>/dt=YYYY-MM-DD/`; lifecycle cho `raw/` |
| S3 `<prefix>-tfstate` | Versioning, khóa bằng `use_lockfile` |
| ECR repos | `assistant-ui`, `bakery-agent`, `cube-mcp`, `dagu-dbt` (có thể mirror image public của Cube, Marmot, DBHub); lifecycle giữ 5 image gần nhất |
| RDS: schema | `raw`, `staging`, `marts` trong database `app` |
| RDS: database | `marmot` |
| RDS: user | `dp_loader` (chỉ ghi `raw`), `dbt` (đọc `public`/`raw`, ghi `staging`/`marts`), `cube_ro` (chỉ đọc `marts`, `statement_timeout = 30s`, giới hạn số kết nối), `marmot` (chủ database `marmot`) |
| RDS instance class (nếu tự tạo cho lab) | Đề xuất `db.t4g.small`, Single-AZ, gp3 20 GB |

### 5.4. Danh tính và secrets

| Tài nguyên | Quyền |
|---|---|
| Task execution role | Kéo image ECR; đọc SSM parameter; ghi CloudWatch Logs |
| Task role | `lambda:InvokeFunction` (chỉ hàm ingest); S3 đọc/ghi bucket datalake; `ssmmessages:*` (ECS Exec); `sns:Publish` |
| Lambda execution role | `s3:PutObject` chỉ trên `raw/<api>/*`; `ssm:GetParameter` chỉ trên key của nó; ghi log |
| GitHub Actions role | Tin cậy OIDC provider dùng chung; push ECR, deploy ECS, chạy Terraform |
| SSM parameters (SecureString) | `OPENAI_API_KEY`, `MARMOT_API_KEY`, `CUBEJS_API_SECRET`, API key của nguồn ingest, mật khẩu từng user DB |

### 5.5. Vận hành và chi phí

| Tài nguyên | Cấu hình |
|---|---|
| CloudWatch log group | Một nhóm cho mỗi container và Lambda; retention 14 ngày |
| CloudWatch alarm | Số task ECS đang chạy < 1; Lambda có lỗi; RDS `CPUCreditBalance` thấp |
| SNS topic `data-platform-alerts` | Gửi email cho Data Engineer; Dagu publish khi có step lỗi |
| AWS Budgets | Khoảng $40/tháng, cảnh báo ở 50/80/100% |
| Cost Anomaly Detection | Bật |
| Tag | `project=bakery-dp`, `env=prod`; kích hoạt làm cost allocation tag |

### 5.6. Cố ý không dùng

NAT Gateway, interface endpoint, ALB riêng, Elastic IP, Secrets Manager, EKS, MWAA, Glue/EMR, Redshift, Multi-AZ cho data platform, Cube Store, database cho lịch sử chat.

### 5.7. Nguyên tắc Terraform với tài nguyên dùng chung

- **Chỉ đọc** tài nguyên của BE bằng `data` source: `aws_vpc`, `aws_subnets`, `aws_lb`, `aws_lb_listener`, `aws_db_instance`, `aws_route53_zone`, `aws_acm_certificate`, `aws_iam_openid_connect_provider`.
- **Chỉ thêm tài nguyên đứng riêng** lên đồ dùng chung: `aws_lb_listener_rule`, `aws_lb_listener_certificate`, `aws_vpc_security_group_ingress_rule`, `aws_vpc_security_group_egress_rule`.
- **Không sửa thuộc tính chung** (idle timeout của ALB, parameter group của RDS…) nếu chưa thống nhất với team BE.

---

## 6. Luồng dữ liệu và luồng AI

### 6.1. Pipeline hằng ngày (Dagu, 05:00 giờ Việt Nam)

**Quy ước về ngày chạy (`run_date`):**
- **`run_date` = D-1 theo giờ Việt Nam.** Pipeline chạy lúc 05:00 ngày D để chủ tiệm xem báo cáo lúc 07:00, nên dữ liệu là của ngày hôm qua.
- **Ngày bán hàng = ngày lịch theo `Asia/Ho_Chi_Minh` (00:00–24:00).** Bánh được làm trong ngày chứ không phải hàng có sẵn, nên sau khi quán đóng cửa không phát sinh đơn bán. Lúc 05:00, ngày D-1 đã khép lại trọn vẹn. Nếu sau này có bán qua nửa đêm (đơn online, sàn giao đồ ăn), cần định nghĩa lại giờ chốt ngày ở tầng staging.
- **Tính ngày theo giờ Việt Nam, không theo UTC.** 05:00 giờ Việt Nam là 22:00 UTC ngày hôm trước, nên nếu tính "hôm qua" theo UTC sẽ ra **D-2** và lấy sai dữ liệu mà không báo lỗi. Cách làm:
  - Dagu chạy với `TZ=Asia/Ho_Chi_Minh`.
  - Tính `run_date` một lần duy nhất ở Dagu (ví dụ `TZ=Asia/Ho_Chi_Minh date -d yesterday +%F`), rồi truyền tham số xuống Lambda, bước load và dbt (`--vars`).
  - Trong dbt, lọc theo `(<cột thời gian> AT TIME ZONE 'Asia/Ho_Chi_Minh')::date`, vì RDS lưu UTC.
- **Dữ liệu đến muộn:** hoàn tiền, đồng bộ trễ từ POS hoặc nguồn API có thể sửa dữ liệu của những ngày trước. Hãy xử lý lại một cửa sổ ngắn (ví dụ D-1 và D-2), hoặc lấy theo `updated_at`. Nhờ merge theo khóa, chạy lại không tạo bản trùng.

1. Ban ngày, BE ghi giao dịch vào `app.public`.
2. Dagu kích hoạt DAG `nightly` với `run_date` là ngày hôm trước.
3. **ingest:** Dagu gọi `lambda invoke` (thêm `--cli-read-timeout 0`). Lambda lấy API key từ SSM, gọi API bên ngoài, ghi JSON vào `s3://<datalake>/raw/<api>/dt=<run_date>/`.
4. **load:** Dagu đọc S3 (qua Gateway Endpoint) rồi upsert vào `raw.<api>` (cột `id`, `payload jsonb`, `ingested_at`, `source_file`).
5. **dbt build:** `public` + `raw` → `staging` → `marts`; chạy test và snapshot.
6. *(Tùy chọn)* Cập nhật metadata vào Marmot.
7. Step nào lỗi thì `onFailure` → SNS → email. Lịch sử chạy lưu trên EFS.

### 6.2. Chủ tiệm hỏi AI

1. `https://ai.<domain>` → Route 53 → ALB → đăng nhập Cognito → `assistant-ui:3000`.
2. Câu hỏi được POST tới `/api/agent` → `bakery-agent:8123` (stream SSE).
3. Agent gọi OpenAI, rồi gọi tool qua MCP theo thứ tự:
   - Marmot: hiểu nghiệp vụ.
   - DBHub: tool báo cáo dựng sẵn hoặc SQL chỉ đọc → SQL API của Cube.
   - Cube MCP: query semantic qua REST.
   - Cube đọc `marts` bằng `cube_ro`.
4. Câu trả lời stream ngược về trình duyệt. Biểu đồ Perspective được vẽ ngay trong trình duyệt từ dữ liệu đã trả về.

### 6.3. Data Engineer quản trị

`aws ssm start-session` (port forwarding qua ECS Exec) → mở `localhost:8080` (Dagu), `:8081` (Marmot), `:4000` (Cube) trên laptop.

---

## 7. Bảo mật

| Lớp | Biện pháp |
|---|---|
| Biên internet | Chỉ ALB:443 mở ra ngoài; chỉ host `ai.<domain>`, sau khi đăng nhập Cognito |
| Mạng | `sg-dp-task` chỉ nhận cổng 3000 từ ALB; RDS ở private subnet, không có đường ra internet; EFS chỉ nhận từ task |
| Truy cập quản trị | ECS Exec/SSM, có ghi log qua IAM; không SSH, không mở port |
| Dữ liệu cho AI | Chỉ đọc `marts`, qua Cube/DBHub; user `cube_ro` có timeout và giới hạn số kết nối |
| Database | Mỗi thành phần một user, quyền tối thiểu; không dùng user master của BE |
| IAM | Mỗi role chỉ có quyền đúng việc (giới hạn tới prefix S3, tên parameter, ARN của Lambda) |
| Secrets | SSM SecureString; không hard-code, không để trong `.env` của image |
| S3 | Block Public Access, mã hóa mặc định, versioning |

---

## 8. Chi phí ước tính

Giá us-east-1 để so sánh. Singapore thường cao hơn một chút; kiểm tra lại bằng AWS Pricing Calculator.

| Khoản | ~ USD/tháng |
|---|---|
| Fargate 0,5 vCPU / 4 GB ARM, 24/7 (x86 thì khoảng $28) | ~22 |
| Public IPv4 của task | 3,65 |
| EFS, S3, ECR | ~1 |
| Lambda, EventBridge, SSM Parameter Store, CloudWatch Logs (trong 5 GB), SNS, Cognito, Budgets | ~0 (free tier) |
| ALB, Route 53, RDS | 0 thêm (dùng chung với BE) |
| OpenAI API | Theo mức dùng |
| **Tổng** | **~$27/tháng + OpenAI** |

---

## 9. Việc phải kiểm tra trước khi triển khai

1. **Trùng cổng 8080:** Dagu, Marmot và DBHub đều mặc định nghe 8080, mà các container trong một task dùng chung `localhost`. Đổi sang 8080 / 8081 / 8082.
2. **Hostname kiểu docker-compose → `localhost`:** đặt lại các biến `*_URL` như ở mục 5.2.
3. **Image agent phải chạy độc lập, không cần volume:** kiểm tra import `data_platform.assistant_ui.agent...` và thư mục `BAKERY_SKILLS_DIR=/app/skills` khi chạy `docker run` không mount gì. Trên ECS không có bind mount.
4. **Kiến trúc arm64:** chạy `docker manifest inspect` cho mọi image. Chỉ một image thiếu bản arm64 là cả task phải chạy x86.
5. **Thứ tự khởi động:** agent chỉ thử kết nối MCP khoảng 40 giây. Dùng `dependsOn` với điều kiện `HEALTHY`, và khai báo health check cho cube-mcp, dbhub, marmot.
6. **SSE qua ALB:** idle timeout mặc định 60 giây là thiết lập chung của ALB. Nếu stream bị cắt, cần thống nhất với team BE.
7. **WAF trên ALB của BE** (nếu có) có thể chặn request chat dài. Thử lại sau khi deploy.
8. **RAM thực tế:** đo `docker stats` khi chạy cả 7 container và hỏi vài câu nặng.
9. **RDS:** tổng số kết nối (BE + Cube + DBHub + Marmot + dbt) phải nằm trong `max_connections`; cửa sổ maintenance và backup không được trùng khung 05:00–06:00 giờ Việt Nam; theo dõi `CPUCreditBalance`.
10. **Múi giờ:** định nghĩa "ngày bán hàng" theo `Asia/Ho_Chi_Minh` ngay ở tầng staging (RDS thường lưu UTC).

---

## 10. Câu hỏi còn mở

| # | Câu hỏi | Mặc định nếu chưa trả lời | Ảnh hưởng |
|---|---|---|---|
| Q1 | BE có NAT Gateway không? | Giả định **không** | Nếu có: chuyển task sang private subnet, bỏ public IP. Phí NAT theo GB ở quy mô này rất nhỏ, và S3 vẫn đi qua Gateway Endpoint |
| Q2 | Chứng chỉ ACM của BE có phải wildcard `*.<domain>` không? | Giả định **không** | Nếu không: xin thêm chứng chỉ `ai.<domain>` (miễn phí) |
| Q3 | Có chuyển LLM sang Bedrock không? | Giữ **OpenAI** | Đổi code sang `langchain-aws`, đổi IAM, bỏ OpenAI key |
| Q4 | API nguồn: lấy full hay incremental (`updated_since`)? Mỗi lần khoảng bao nhiêu bản ghi? | Lấy theo `run_date` | Thiết kế watermark và bảng `raw` |

---

## 11. Khi nào phải vượt ra khỏi thiết kế này

Thiết kế này **cố ý** chỉ phục vụ quy mô nhỏ. Các dấu hiệu sau báo hiệu cần sang giai đoạn tiếp theo:

| Dấu hiệu | Gợi ý hướng đi |
|---|---|
| dbt làm RDS chậm, xả hết CPU credit, hoặc kéo dài sang giờ mở cửa | Tách RDS analytics riêng (logical replication/DMS), hoặc chuyển sang S3 + Iceberg + Athena |
| Có thêm người dùng AI (quản lý chi nhánh, nhân viên) | Đăng nhập theo người dùng, phân quyền dữ liệu, lưu lịch sử chat bền vững, cân nhắc tách service |
| Cần dữ liệu tươi hơn T-1 | Ingest nhiều lần trong ngày hoặc CDC; cân nhắc cache/pre-aggregation của Cube |
| Nhiều nguồn API, job Python nặng | Mỗi nguồn một Lambda; job nặng chuyển sang ECS RunTask / AWS Batch |
| Task gộp quá nặng, deploy ảnh hưởng lẫn nhau | Tách các ECS service (trước hết là web UI/agent và Dagu) |
| Cần độ sẵn sàng cao | Nhiều task cho các service stateless, Multi-AZ, tách scheduler |