# Worklog khảo sát và lập kế hoạch triển khai GoClaw

Ngày: 2026-07-29

## Mục tiêu

Clone và đọc kỹ repo GoClaw, sau đó lập kế hoạch chuyên sâu để đưa source lên GitHub riêng, triển khai lên VPS và truy cập sử dụng qua domain.

## Thay đổi đã làm

- Clone `https://github.com/nextlevelbuilder/goclaw` vào `/Users/bexaycop/cop/goclaw`.
- Giữ remote tên `upstream` để không nhầm với GitHub repo đích của Sếp.
- Xác nhận nhánh mặc định upstream là `dev`.
- Xác nhận snapshot khảo sát:
  - commit `496b7ffce648e12380b913f92753cd0c6792ce76`;
  - tag `v3.15.0-beta.180`.
- Xác nhận stable release hiện tại:
  - tag `v3.14.0`;
  - commit `2f3d68e806c11a20b59b0591bca75410ed1237f7`.
- Đọc các phần chính:
  - `AGENTS.md`;
  - README Anh/Việt;
  - kiến trúc và dependency;
  - Dockerfile/entrypoint;
  - Compose base và các overlay;
  - auth, gateway token, browser pairing;
  - Web UI login và WebSocket;
  - health endpoint;
  - migration/schema;
  - CI/release/GHCR;
  - security docs;
  - VPS/deployment docs;
  - license.
- Tạo kế hoạch tại `doc/2026-07-29-goclaw-vps-github-deployment-plan.md`.

## Lỗi gặp

- Máy local hiện không có `docker`, `go` và `pnpm`, nên không thể chạy build/test/compose runtime trong phiên lập kế hoạch.
- Trang docs động không mở trực tiếp được toàn bộ route bằng web reader.
- Tài liệu quick-start có một số chỗ lệch giữa cổng `3000` của frontend rời và `18790` của Web UI nhúng.

## Cách xử lý

- Không coi thiếu tool local là bằng chứng source lỗi; giới hạn kết luận ở static inspection và Git evidence.
- Đọc docs Markdown trực tiếp từ `docs.goclaw.sh/*.md`.
- Đối chiếu tài liệu với source thật:
  - Compose/Dockerfile xác nhận UI nhúng chạy cùng gateway `18790`;
  - UI source xác nhận WebSocket route `/ws` và tự dùng `wss` khi domain là HTTPS;
  - gateway source xác nhận `/health` chỉ là liveness, không ping DB.
- Ghi rõ phần runtime chưa verify thay vì tuyên bố đã deploy/chạy.

## Lý do kỹ thuật

- Dùng stable tag + digest tránh deploy nhầm nhánh `dev` beta.
- Dùng GHCR + VPS pull image tách build khỏi production server.
- Dùng compose production độc lập tránh merge `ports` và vô tình public `5432/18790`.
- Dùng Caddy để TLS/redirect/WebSocket đơn giản và lặp lại được.
- Giữ runtime secret ở VPS để giảm blast radius của GitHub Actions.
- Backup DB phải đi cùng encryption key vì provider credentials được AES-256-GCM mã hóa.
- `/health` không phản ánh DB nên cần thêm `pg_isready` và authenticated API probe.

## Kết quả verify

- `git ls-remote` xác nhận upstream HEAD trỏ `refs/heads/dev`.
- Local clone sạch trước khi thêm tài liệu.
- `git rev-parse HEAD` xác nhận `496b7ffc...`.
- GitHub Releases API xác nhận latest non-prerelease là `v3.14.0`.
- `git show v3.14.0:internal/upgrade/version.go` xác nhận schema stable là `80`.
- Source `dev` xác nhận schema hiện tại là `95`.
- `LICENSE` xác nhận `CC BY-NC 4.0`.
- Source gateway xác nhận route `/health` và `/ws`.
- Source Web UI xác nhận token login và WebSocket cùng origin.

## Rà soát chéo kế hoạch

Đã spot-check 22 claim quan trọng với source/live upstream, tất cả đều pass:

1. snapshot SHA;
2. snapshot beta tag;
3. nhánh mặc định `dev`;
4. stable tag SHA;
5. latest non-prerelease từ GitHub API;
6. schema version của `dev`;
7. schema version của stable;
8. Go version;
9. PostgreSQL/pgvector image;
10. gateway port upstream đang publish ra host;
11. PostgreSQL port upstream đang publish ra host;
12. `/health` route;
13. `/health` chỉ trả liveness tĩnh;
14. `/ws` route;
15. HTTPS tự chuyển sang WSS ở UI;
16. auth token được persist trong localStorage;
17. non-loopback bind không token sẽ fail;
18. allowed origins rỗng cho phép mọi origin;
19. ba format encryption key;
20. điều khoản NonCommercial;
21. build tag `embedui`;
22. entrypoint hạ quyền sang user `goclaw`.

## Việc còn lại

- Xác nhận mục đích sử dụng và quyền theo license nếu có yếu tố thương mại.
- Có domain/DNS, VPS/SSH/sudo và GitHub repo đích.
- Preflight VPS read-only và snapshot.
- Xây deploy artifacts/CI/CD theo kế hoạch.
- Chạy full test/build.
- Deploy, DNS, TLS, login, provider, agent và chat E2E.
- Backup/restore rehearsal và monitoring.

## Quy tắc làm việc có gì thay đổi

Không thay đổi quy tắc làm việc chung.

## Mắc lỗi gì trong quy tắc làm việc

Không phát hiện vi phạm quy tắc. Phần chưa thể runtime-verify đã được nêu rõ, không đánh dấu là hoàn tất deployment.

## Đã khắc phục chưa

Không có lỗi quy trình cần khắc phục. Các hạn chế tool local đã được xử lý bằng static inspection và ghi rõ gate runtime cho phase thực hiện.

## Đã lưu thành nguyên tắc làm việc chưa

Không có nguyên tắc mới cần cập nhật vào `AGENTS.md` hoặc memory.

---

## Bổ sung: đồng bộ toàn bộ dữ liệu an toàn lên GitHub

### Mục tiêu

- Đưa snapshot GoClaw đã clone cùng toàn bộ tài liệu kế hoạch/worklog mới lên repo GitHub thuộc tài khoản của Sếp.
- Giữ nguyên lịch sử upstream và tách rõ remote nguồn với remote fork.

### Thay đổi đã làm

- Fetch và đối chiếu local `dev` với `upstream/dev`.
- Xác nhận khóa SSH GitHub thuộc tài khoản `bexaycop`.
- Chuẩn bị tạo fork `bexaycop/goclaw`, đặt remote `origin` trỏ fork và giữ `upstream` trỏ repo nguồn.
- Rà soát toàn bộ file thay đổi trước khi commit/push.

### Lỗi gặp

- Máy chưa có GitHub CLI.
- SSH mặc định chưa tự chọn đúng khóa GitHub.
- Repo `bexaycop/goclaw` chưa tồn tại tại thời điểm bắt đầu đồng bộ.

### Cách xử lý

- Cài GitHub CLI bằng Homebrew.
- Dùng riêng khóa `id_ed25519_github` với `IdentitiesOnly=yes`; không in nội dung khóa hoặc credential.
- Chọn fork chính thức của `nextlevelbuilder/goclaw` làm repo đích để giữ quan hệ upstream và lịch sử nguồn.

### Lý do kỹ thuật

- Không push vào remote `upstream` của nhà phát triển.
- Fork phù hợp hơn một repo rời vì hỗ trợ theo dõi và cập nhật bản gốc.
- Tách `origin`/`upstream` giúp quy trình nâng cấp sau này rõ ràng và giảm nguy cơ push nhầm.

### Kết quả verify

- `git fetch upstream --prune` hoàn tất.
- `HEAD...upstream/dev` trả `0 0` trước khi tạo commit tài liệu.
- SSH xác thực thành công đúng tài khoản `bexaycop`.
- Các kiểm tra staged content, secret, file lớn, whitespace, SHA local/remote được ghi bổ sung sau bước push.

### Việc còn lại

- Hoàn tất xác thực GitHub CLI nếu GitHub yêu cầu.
- Tạo fork, commit, push và đối chiếu SHA ba chiều.

### Quy tắc làm việc có gì thay đổi

Không thay đổi quy tắc làm việc chung.

### Mắc lỗi gì trong quy tắc làm việc

Không có vi phạm. Lỗi phụ thuộc công cụ và chọn khóa SSH đã được xử lý nội bộ trước khi báo cáo.

### Đã khắc phục chưa

Đã khắc phục phần GitHub CLI và SSH; bước tạo fork/push đang được thực hiện.

### Đã lưu thành nguyên tắc làm việc chưa

Không phát sinh nguyên tắc mới cần cập nhật.
