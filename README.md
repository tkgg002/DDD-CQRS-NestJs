# DDD-CQRS-NestJs

triển khai DDD kết hợp CQRS trong NestJs. Hôm nay mình mạn phép chia sẻ kiến trúc mình đang sử dụng thực tế trong các dự án của bên minh.
=====
Các dự án bên mình tất cả đều triển khai tuân theo kiến trúc monorepos. Trong monorepos sẽ triển khai các app con.
Đa số các app bên mình được xây dựng theo mô hình Domain-Driven Design (DDD), Clean Architecture và CQRS (Command Query Responsibility Segregation) với Event-Driven Architecture. Kiến trúc này được tổ chức thành 4 lớp chính:
1. Domain Layer (src/domain/)
Đây là lớp trung tâm chứa logic nghiệp vụ cốt lõi:
- DTOs (dtos/): Định nghĩa các đối tượng truyền dữ liệu cho domain
- Entities (entities/): Các thực thể nghiệp vụ (được import từ @modules/data)
- Enums (enums/): Các hằng số và enum định nghĩa trạng thái, loại dữ liệu
- Events (events/): Các sự kiện domain như UserRegisteredEvent, EcoDocumentCreated
- Exceptions (exceptions/): Các ngoại lệ nghiệp vụ cụ thể
- Services (services/): Các dịch vụ domain như tính toán carbon footprint, chuyển đổi trạng thái tài liệu
- Helpers (helpers/): Các hàm tiện ích hỗ trợ logic nghiệp vụ
2. Application Layer (src/application/)
Lớp này điều phối các use case và logic ứng dụng:
- Commands (commands/): Xử lý các thao tác thay đổi dữ liệu (Create, Update, Delete)
- Mỗi command có cấu trúc: {Entity}{Action}.command.ts
- Handler tương ứng: handlers/{Entity}{Action}.command.handler.ts
- Ví dụ: UserRegister.command.ts và UserRegister.command.handler.ts
- Queries (queries/): Xử lý các thao tác đọc dữ liệu
- Cấu trúc tương tự commands nhưng cho việc truy vấn
- Ví dụ: UserSettingsGet.query.ts
- Event Handlers (event-handlers/): Xử lý các sự kiện domain
- Tự động xử lý side effects khi events được phát ra
Ví dụ: UserRegistered.event.handler.ts tự động tạo ví MXT và EcoAccount
3. Infrastructure Layer (src/infrastructure/)
Lớp này cung cấp các implementation cụ thể:
- Repositories (repositories/): Implementation các repository pattern
- Services (services/): Các dịch vụ hạ tầng như caching, external APIs
- DTOs (dtos/): DTOs cho tích hợp với hệ thống bên ngoài
4. Presentation Layer (src/presentation/)
Lớp giao diện với bên ngoài:
- Controllers (http/controllers/): Các REST API endpoints
- Không chứa logic nghiệp vụ, chỉ delegate cho commands/queries
- Không sử dụng prefix trong @Controller() decorator
- DTOs (http/dtos/): DTOs cho HTTP requests/responses
- Decorators (decorators/): Custom decorators cho presentation
Mối tương quan giữa các thành phần
1. Luồng xử lý Request
HTTP Request → Controller → Command/Query → Handler → Domain Services → Repository → Database
2. Event-Driven Flow
Command Handler → Phát Event → Event Handler → Xử lý Side Effects
3. Dependency Direction
- Presentation phụ thuộc vào Application
- Application phụ thuộc vào Domain
- Infrastructure implement interfaces từ Domain
- Domain không phụ thuộc vào lớp nào khác (Dependency Inversion)
Đặc điểm kiến trúc
CQRS Pattern
- Tách biệt rõ ràng giữa Commands (ghi) và Queries (đọc)
- Commands có thể phát Events, Queries chỉ đọc dữ liệu
- Mỗi use case có handler riêng biệt
- Event-Driven Architecture
- Command handlers phát events sau khi thực hiện thành công
- Event handlers tự động xử lý side effects
- Giảm coupling giữa các module, tăng tính mở rộng
Repository Pattern
- Repositories trong @modules/data chỉ làm CRUD cơ bản
- Repositories phức tạp được đặt trong application layer
Module chính (mxt.module.ts)
Module này tích hợp tất cả các thành phần:
- Import các modules cần thiết (Auth, Data, Notifications, etc.)
- Đăng ký tất cả command/query/event handlers
- Cấu hình các services như Redis, JWT, Esign
- Đăng ký tất cả controllers
Ưu điểm của kiến trúc này
- Tách biệt rõ ràng: Mỗi lớp có trách nhiệm cụ thể
- Dễ test: Logic nghiệp vụ tách biệt khỏi infrastructure
- Dễ mở rộng: Event-driven cho phép thêm tính năng mà không ảnh hưởng code cũ
- Maintainable: CQRS giúp tối ưu hóa riêng cho read/write operations
- Domain-centric: Logic nghiệp vụ được bảo vệ và tập trung
Kiến trúc này phù hợp cho các ứng dụng enterprise phức tạp với nhiều domain logic và yêu cầu về scalability, maintainability cao.

---
Note: 
Value Objects là các đối tượng không có identity, được định nghĩa bởi các thuộc tính của chúng và bất biến (immutable).
- Immutable (không thể thay đổi sau khi tạo)
- Không có identity (được so sánh bằng giá trị)
- Có thể có business logic
- Sử dụng static factory methods

---
Entities có identity duy nhất và có thể thay đổi trạng thái theo thời gian.
- Có identity duy nhất (ID)
- Có thể thay đổi trạng thái
- Kế thừa từ BaseDomainModel
- Sử dụng static factory methods
- ---

Aggregate Root là entity chính quản lý tính nhất quán của aggregate và là điểm truy cập duy nhất từ bên ngoài.
- Kế thừa từ AggregateRoot (NestJS CQRS)
- Quản lý domain events
- Đảm bảo business invariants
- Có validation logic
- Là điểm truy cập duy nhất cho aggregate

- ---
Domain Services chứa business logic không thuộc về một entity hoặc value object cụ thể nào.

Repository cung cấp interface để truy cập aggregate roots.
- Kế thừa từ EntityRepository
- Chuyển đổi giữa Entity và Domain Model
- Cung cấp interface domain-friendly
- Ẩn chi tiết persistence

Validation và Business Rules:
- Validation trong constructor của domain models
- Business rules trong domain services
- Invariants được đảm bảo bởi aggregate roots
Immutability và Encapsulation:
- Value Objects hoàn toàn immutable
- Entities có controlled mutability
- Private constructors với static factory methods
Event-Driven Architecture:
- Domain events cho side effects
- Aggregate roots publish events
- Event handlers xử lý cross-aggregate logic

- ---



