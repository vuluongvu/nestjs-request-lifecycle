# NestJS Request Lifecycle — Vòng Đời Của 1 Request (ví dụ `POST /orders`)

Tài liệu này giải thích **thứ tự thực sự** mà 1 request đi qua bên trong NestJS, đúng theo sơ đồ: `Middleware → Guard → Interceptor → Pipe → Controller → Service/DB → Interceptor → Client`, và `Exception Filter` sẵn sàng bắt lỗi ở bất kỳ bước nào.

Hiểu rõ sơ đồ này giúp bạn biết chính xác **nên đặt logic ở đâu** — rất nhiều lỗi thiết kế sai xuất phát từ việc đặt nhầm chỗ (ví dụ validate trong Controller thay vì Pipe, log trong Service thay vì Interceptor).

---

## 0. Sơ đồ tổng quan

```
Client
  │  POST /orders
  ▼
① Middleware        (chạy sớm nhất, trước khi Nest biết route nào sẽ xử lý)
  ▼
② Guard             (quyết định request có ĐƯỢC PHÉP đi tiếp hay không)
  ▼
③ Interceptor (before)  (can thiệp TRƯỚC khi vào Controller)
  ▼
④ Pipe              (biến đổi / validate dữ liệu đầu vào)
  ▼
⑤ Controller        (nhận request đã "sạch", gọi Service)
  ▼
⑥ Service & DB       (xử lý nghiệp vụ, lưu database)
  ▼
③ Interceptor (after)   (can thiệp SAU khi Controller trả kết quả, trước khi gửi về Client)
  ▼
Client nhận response

⑦ Exception Filter: nếu BẤT KỲ bước nào ở trên ném lỗi (throw),
   luồng bị ngắt ngay lập tức và nhảy thẳng tới đây để trả lỗi cho Client.
```

**Ghi nhớ quan trọng nhất:** Interceptor là **duy nhất** chạy 2 lần — 1 lần trước Controller, 1 lần sau khi Controller trả kết quả. Đây là lý do trong sơ đồ bạn thấy "Interceptor" xuất hiện ở cả bước 3 và bước cuối.

---

## ① Middleware — người gác cổng đầu tiên

Middleware chạy **trước cả khi Nest xác định route nào sẽ xử lý request**. Dùng để làm những việc chung cho mọi request: gắn Request ID, log thời gian bắt đầu, đọc header thô...

```ts
// src/common/middleware/request-id.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Gắn 1 mã định danh riêng cho mỗi request, dùng để trace log xuyên suốt hệ thống
    req['requestId'] = uuidv4();
    console.log(`[${req['requestId']}] ${req.method} ${req.originalUrl} — bắt đầu`);
    next(); // BẮT BUỘC gọi next(), nếu không request sẽ bị "treo" mãi mãi
  }
}
```

Đăng ký trong module:
```ts
// src/app.module.ts
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(RequestIdMiddleware).forRoutes('*'); // áp dụng cho mọi route
  }
}
```

**Vì sao dùng Middleware chứ không phải Interceptor cho việc này?**
Middleware chạy sớm hơn — nó không biết (và không cần biết) request này sẽ được xử lý bởi Controller/method nào. Phù hợp cho các việc **không phụ thuộc route cụ thể**: log, nén response, đọc cookie... Interceptor thì gắn với route cụ thể và có thể truy cập vào `ExecutionContext` (biết chính xác class/method nào sắp chạy).

---

## ② Guard — người quyết định cho qua hay chặn

Guard trả lời câu hỏi: **"Request này có được phép đi tiếp không?"** — đúng/sai (`true`/`false`), không sửa đổi dữ liệu.

```ts
// src/auth/guards/jwt-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

```ts
// src/auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    if (!requiredRoles.includes(user?.role)) {
      throw new ForbiddenException('Bạn không có quyền thực hiện thao tác này');
    }
    return true;
  }
}
```

Áp dụng cho route `POST /orders`:
```ts
@Post()
@UseGuards(JwtAuthGuard, RolesGuard) // JwtAuthGuard chạy trước, xác thực token -> gắn request.user
create(@CurrentUser() user, @Body() dto: CreateOrderDto) { ... }
```

**Vì sao Guard chạy sau Middleware nhưng trước Interceptor/Pipe?**
Vì việc xác thực/phân quyền nên được quyết định **càng sớm càng tốt** — nếu request không hợp lệ (thiếu token, sai quyền), không có lý do gì để tốn công chạy tiếp Interceptor hay Pipe. Guard giống như trạm kiểm soát ở cổng, chặn lại ngay khi phát hiện vé vào cửa không hợp lệ.

---

## ③ Interceptor — can thiệp cả trước lẫn sau

Interceptor là lớp duy nhất "bọc quanh" toàn bộ quá trình xử lý — chạy trước khi vào Controller, và chạy lại sau khi Controller trả kết quả.

### 3.1 Logging Interceptor (chạy trước — ghi log; chạy sau — tính thời gian xử lý)

```ts
// src/common/interceptors/logging.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const start = Date.now();

    console.log(`[${request.requestId}] Trước khi vào Controller`);

    return next.handle().pipe(
      // next.handle() chính là điểm gọi vào Controller
      // .pipe(tap(...)) chạy SAU KHI Controller đã trả kết quả xong
      tap(() => {
        console.log(`[${request.requestId}] Xử lý xong sau ${Date.now() - start}ms`);
      }),
    );
  }
}
```

### 3.2 Transform Interceptor (chuẩn hóa hình dạng response trả về client)

Đây chính là khối vàng "Transform Data" trong sơ đồ của bạn:

```ts
// src/common/interceptors/transform.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const response = context.switchToHttp().getResponse();

    return next.handle().pipe(
      map((data) => ({
        statusCode: response.statusCode,
        message: 'Success',
        timestamp: new Date().toISOString(),
        data,
      })),
    );
  }
}
```

Đăng ký toàn cục trong `main.ts`:
```ts
app.useGlobalInterceptors(new LoggingInterceptor(), new TransformInterceptor());
```

**Vì sao dùng `next.handle().pipe(...)` mà không viết code bình thường?**
Vì NestJS xử lý request theo kiểu **reactive (RxJS Observable)**, không phải Promise thông thường. `next.handle()` trả về 1 Observable đại diện cho "toàn bộ phần còn lại của luồng xử lý" (Pipe → Controller → Service). Gọi `.pipe(tap(...))` hoặc `.pipe(map(...))` nghĩa là "sau khi phần đó chạy xong, làm thêm việc này". Đây là lý do Interceptor có thể chạy code cả trước (trước dòng `return`) và sau (bên trong `.pipe()`).

---

## ④ Pipe — validate và biến đổi dữ liệu đầu vào

Pipe chạy **ngay trước khi dữ liệu được truyền vào tham số của Controller**.

```ts
// src/orders/dto/create-order.dto.ts
import { IsArray, ArrayMinSize, ValidateNested } from 'class-validator';
import { Type } from 'class-transformer';
import { CreateOrderItemDto } from './create-order-item.dto';

export class CreateOrderDto {
  @IsArray()
  @ArrayMinSize(1, { message: 'Đơn hàng phải có ít nhất 1 sản phẩm' })
  @ValidateNested({ each: true })
  @Type(() => CreateOrderItemDto)
  items: CreateOrderItemDto[];
}
```

Bật `ValidationPipe` toàn cục trong `main.ts`:
```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,            // loại field lạ không khai báo trong DTO
    forbidNonWhitelisted: true, // có field lạ -> báo lỗi 400 ngay
    transform: true,            // tự ép kiểu (string -> number...)
  }),
);
```

**Vì sao Pipe chạy sau Guard nhưng trước Controller?**
Guard chỉ trả lời "được vào hay không", **không quan tâm nội dung dữ liệu**. Pipe mới là nơi kiểm tra "dữ liệu gửi lên có đúng định dạng không". Nếu đặt việc validate ở Controller (thay vì Pipe), Controller sẽ bị lẫn lộn giữa 2 việc: kiểm tra dữ liệu và xử lý nghiệp vụ — vi phạm nguyên tắc mỗi lớp chỉ nên làm đúng 1 nhiệm vụ.

Nếu `dto` không hợp lệ, Pipe tự ném lỗi ngay tại đây — request **không bao giờ chạm tới Controller**, nhảy thẳng tới Exception Filter.

---

## ⑤ Controller — chỉ điều phối, không xử lý logic

Tới bước này, dữ liệu đã đảm bảo: **request hợp lệ (qua Guard), dữ liệu đúng định dạng (qua Pipe)**. Controller chỉ việc gọi Service.

```ts
// src/orders/orders.controller.ts
@Controller('orders')
export class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}

  @Post()
  @UseGuards(JwtAuthGuard, RolesGuard)
  create(@CurrentUser() user: { userId: number }, @Body() dto: CreateOrderDto) {
    return this.ordersService.create(user.userId, dto);
  }
}
```

Chỉ có 1 dòng logic: gọi `ordersService.create()`. Đây chính là chuẩn "Controller mỏng" đã nhắc ở các tài liệu trước.

---

## ⑥ Service & DB — nơi xử lý nghiệp vụ thật

```ts
// src/orders/orders.service.ts
@Injectable()
export class OrdersService {
  constructor(
    @InjectRepository(Order) private readonly orderRepository: Repository<Order>,
    @InjectRepository(Product) private readonly productRepository: Repository<Product>,
  ) {}

  async create(userId: number, dto: CreateOrderDto): Promise<Order> {
    const productIds = dto.items.map((i) => i.productId);
    const products = await this.productRepository.findBy({ id: In(productIds) });

    if (products.length !== productIds.length) {
      // Ném lỗi tại đây -> nhảy thẳng tới Exception Filter, bỏ qua toàn bộ phần còn lại
      throw new NotFoundException('Có sản phẩm không tồn tại');
    }

    const items = dto.items.map((itemDto) => {
      const product = products.find((p) => p.id === itemDto.productId);
      const item = new OrderItem();
      item.product = product;
      item.quantity = itemDto.quantity;
      item.price = product.price;
      return item;
    });

    const order = this.orderRepository.create({ user: { id: userId } as any, items });
    return this.orderRepository.save(order); // ← lưu vào DB
  }
}
```

Đây là nơi **duy nhất** nên chứa: check trùng, tính toán, gọi database. Không có Guard/Pipe/Interceptor nào thay thế được vai trò này.

---

## ③ (lượt 2) Interceptor — bọc lại response trước khi trả về Client

Sau khi `orderRepository.save(order)` trả về, dữ liệu đi ngược trở lại qua `TransformInterceptor` đã đăng ký ở bước ③. Đây chính là lúc `.pipe(map(...))` được thực thi:

```json
{
  "statusCode": 201,
  "message": "Success",
  "timestamp": "2026-09-19T23:03:01.000Z",
  "data": {
    "id": 10,
    "items": [ { "id": 21, "quantity": 2, "price": "150000.00" } ]
  }
}
```

`data` chính là những gì `OrdersController.create()` return ra (`Order` entity), còn `statusCode`, `message`, `timestamp` là do Interceptor **thêm vào bọc bên ngoài** — Controller hoàn toàn không biết tới cấu trúc bọc này, đúng nguyên tắc tách biệt trách nhiệm.

---

## ⑦ Exception Filter — trạm cứu hộ cho mọi lỗi

Nếu **bất kỳ bước nào** ở trên (Guard, Pipe, Controller, Service) ném lỗi (`throw new SomeException()`), luồng xử lý bình thường bị **ngắt ngay lập tức** và nhảy thẳng tới Exception Filter — bỏ qua hoàn toàn phần Interceptor (after) và Controller còn lại.

```ts
// src/common/filters/http-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      message: exception.message,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

Đăng ký toàn cục:
```ts
app.useGlobalFilters(new HttpExceptionFilter());
```

**Ví dụ minh họa lỗi xảy ra ở từng bước, kết quả trả về giống hệt nhau nhờ Exception Filter:**

| Lỗi xảy ra ở đâu | Nguyên nhân | Status trả về |
|---|---|---|
| Guard | Thiếu/sai token | `401 Unauthorized` |
| Guard | Đúng token nhưng sai role | `403 Forbidden` |
| Pipe | `items` rỗng, thiếu `productId` | `400 Bad Request` |
| Service | `productId` không tồn tại trong DB | `404 Not Found` |

Dù lỗi xuất phát từ đâu, Client luôn nhận về **1 định dạng lỗi duy nhất, nhất quán** — đây chính là giá trị lớn nhất của Exception Filter: tách biệt hoàn toàn "nơi phát sinh lỗi" và "cách hiển thị lỗi cho client".

---

## Tổng kết: nên đặt logic gì ở đâu

| Lớp | Trả lời câu hỏi | Ví dụ việc nên làm | KHÔNG nên làm |
|---|---|---|---|
| **Middleware** | Việc gì cần chạy cho MỌI request, bất kể route? | Gắn Request ID, log thô, nén response | Kiểm tra quyền, validate dữ liệu |
| **Guard** | Request này có được phép đi tiếp không? | Kiểm tra token, kiểm tra role | Sửa đổi dữ liệu, gọi database ghi |
| **Interceptor** | Cần làm gì trước/sau khi Controller chạy? | Log thời gian, bọc format response, cache | Validate dữ liệu (đó là việc của Pipe) |
| **Pipe** | Dữ liệu đầu vào có đúng định dạng không? | Validate DTO, ép kiểu | Gọi database, xử lý nghiệp vụ |
| **Controller** | Route nào gọi hàm nào? | Nhận request, gọi đúng Service | Chứa logic nghiệp vụ, hash password, query DB trực tiếp |
| **Service** | Xử lý nghiệp vụ thực sự thế nào? | Check trùng, tính toán, gọi Repository | Đọc `req.headers` trực tiếp, biết về HTTP status code |
| **Exception Filter** | Khi có lỗi, hiển thị cho client thế nào? | Chuẩn hóa format lỗi | Chứa logic nghiệp vụ |

---

## Checklist ghi nhớ vòng đời request

| # | Ghi nhớ |
|---|---|
| 1 | Thứ tự chuẩn: Middleware → Guard → Interceptor (before) → Pipe → Controller → Service → Interceptor (after) → Client |
| 2 | Guard chạy **trước** Pipe — vì "có được vào không" phải biết trước "dữ liệu có đúng không" |
| 3 | Interceptor là lớp **duy nhất** chạy cả trước lẫn sau Controller (nhờ RxJS `pipe()`) |
| 4 | Lỗi ở bất kỳ bước nào cũng nhảy thẳng tới Exception Filter, bỏ qua toàn bộ phần còn lại |
| 5 | Controller không bao giờ chứa logic nghiệp vụ — chỉ gọi Service |
| 6 | Muốn áp dụng cho toàn bộ ứng dụng, dùng `useGlobalPipes`/`useGlobalGuards`/`useGlobalInterceptors`/`useGlobalFilters` trong `main.ts` |

---

## Bước tiếp theo

Bạn đã hiểu được "bộ khung xương" cách NestJS xử lý 1 request. Đây là kiến thức nền để đọc hiểu bất kỳ đoạn code NestJS nào, kể cả code người khác viết. Muốn mình viết tiếp tài liệu **Cart (giỏ hàng)** hay **File Upload** theo roadmap cũ không?