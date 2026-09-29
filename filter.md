# Hướng dẫn cài đặt Query lọc động (NestJS + TypeORM)

> Áp dụng cho: module `orders`, endpoint `GET /api/v1/orders/search`
> Mục tiêu: cho phép client lọc theo **điều kiện tuỳ chọn** (status, keyword, from, to), có phân trang, an toàn và dễ bảo trì.

---

## 1. Tổng quan luồng xử lý
```page + limit
     ↓
skip = (page - 1) × limit
     ↓
findAndCount({
    skip,
    take: limit
})
     ↓
orders + total
     ↓
data + meta
```

```
Client  →  Query string  →  ValidationPipe  →  Controller  →  Service  →  TypeORM  →  DB
           ?status=...      (validate +         (lấy userId    (dựng where   (sinh SQL)
                             ép kiểu)            từ token)      động)
```

| Lớp | Trách nhiệm | KHÔNG làm |
|---|---|---|
| **DTO** | Khai báo và validate dữ liệu đầu vào | Xử lý nghiệp vụ |
| **Controller** | Nhận request, lấy `userId`, gọi service | Dựng câu truy vấn |
| **Service** | Dựng `where` động, gọi repository | Đọc `req`, `res` trực tiếp |
| **Exception Filter** | Chuẩn hoá format lỗi trả về | Che mất message lỗi gốc |

---

## 2. Cấu hình bắt buộc: `ValidationPipe`

Trong `main.ts`:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,            // ép kiểu theo DTO (string → Date, Number...)
    whitelist: true,            // xoá property không có decorator
    forbidNonWhitelisted: true, // báo lỗi 400 nếu client gửi property lạ
  }),
);
```

Giải thích từng option:

- **`transform: true`**: query string luôn là `string`. Nếu không bật, `@Type(() => Date)` không có tác dụng và `from`/`to` vẫn là chuỗi.
- **`whitelist: true`**: chỉ giữ lại property có decorator của `class-validator`.
- **`forbidNonWhitelisted: true`**: property không có decorator sẽ bị **từ chối (400)** thay vì bị xoá âm thầm.

> ⚠️ **Hệ quả quan trọng**: mọi property trong DTO **bắt buộc phải có ít nhất một decorator validate**. Thiếu decorator thì request bị 400 với lỗi `property xxx should not exist`.

---

## 3. Enum dùng chung

Chỉ khai báo **một nơi duy nhất**, cả entity và DTO cùng import. Tránh tạo hai enum trùng tên vì TypeScript coi chúng là hai kiểu khác nhau.

```ts
// src/common/enums/order-status.enum.ts
export enum OrderStatus {
  PENDING = 'PENDING',
  PAID = 'PAID',
  SHIPPING = 'SHIPPING',
  DONE = 'DONE',
  CANCELLED = 'CANCELLED',
}
```

> Client gửi **giá trị** của enum (vế phải), không phải tên key. Ví dụ `PENDING = 'pending'` thì phải gọi `?status=pending`.

---

## 4. DTO: `SearchOrderDto`

```ts
// src/orders/dto/search-order.dto.ts
import { Type } from 'class-transformer';
import {
  IsDate,
  IsEnum,
  IsInt,
  IsOptional,
  IsString,
  Max,
  MaxLength,
  Min,
} from 'class-validator';
import { OrderStatus } from '../../common/enums/order-status.enum';

export class SearchOrderDto {
  @IsOptional()
  @IsEnum(OrderStatus)
  status?: OrderStatus;

  @IsOptional()
  @IsString()
  @MaxLength(100)
  keyword?: string;

  @IsOptional()
  @Type(() => Date)
  @IsDate()
  from?: Date;

  @IsOptional()
  @Type(() => Date)
  @IsDate()
  to?: Date;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100) // chặn client xin quá nhiều bản ghi
  limit: number = 20;
}
```

Giải thích decorator:

| Decorator | Ý nghĩa |
|---|---|
| `@IsOptional()` | Field có thể vắng mặt (`undefined`), khi đó bỏ qua các validator còn lại |
| `@IsEnum(OrderStatus)` | Giá trị phải nằm trong enum |
| `@Type(() => Date)` | Ép `string` → `Date` (cần `transform: true`) |
| `@IsDate()` | Kiểm tra sau khi ép kiểu là `Date` hợp lệ |
| `@Max(100)` | Giới hạn tối đa, chống lạm dụng API |

Quy ước đường dẫn import: dùng **đường dẫn tương đối** (`../../common/...`) để build, test (Jest) không phụ thuộc cấu hình `baseUrl`.

---

## 5. Service: dựng `where` động

### 5.1 Nguyên tắc

1. Bắt đầu bằng **điều kiện gốc luôn có** (lọc theo user).
2. Chỉ **thêm điều kiện khi client có gửi** giá trị đó.
3. Không bao giờ nhét `undefined`/`null` trực tiếp vào `where` (xem mục 8).

### 5.2 Code

```ts
// src/orders/orders.service.ts
import { BadRequestException, Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import {
  Between,
  FindOptionsWhere,
  ILike,
  LessThanOrEqual,
  MoreThanOrEqual,
  Repository,
} from 'typeorm';

async search(userId: number, dto: SearchOrderDto) {
  // (1) Guard: chặn rò rỉ dữ liệu nếu userId bị undefined
  if (userId == null) {
    throw new BadRequestException('userId is required');
  }

  // (2) Kiểm tra logic khoảng thời gian
  if (dto.from && dto.to && dto.from > dto.to) {
    throw new BadRequestException('"from" must be before "to"');
  }

  // (3) Điều kiện gốc: luôn lọc theo user hiện tại
  const base: FindOptionsWhere<Order> = { user: { id: userId } };

  // (4) status: chỉ thêm khi có gửi
  if (dto.status !== undefined) {
    base.status = dto.status;
  }

  // (5) Khoảng thời gian
  const to = dto.to ? new Date(dto.to) : undefined;
  if (to) {
    to.setHours(23, 59, 59, 999); // "to" = hết ngày, không bị mất đơn trong ngày đó
  }

  if (dto.from && to) {
    base.createdAt = Between(dto.from, to);
  } else if (dto.from) {
    base.createdAt = MoreThanOrEqual(dto.from);
  } else if (to) {
    base.createdAt = LessThanOrEqual(to);
  }

  // (6) keyword: OR giữa nhiều field ⇒ where phải là MẢNG
  let where: FindOptionsWhere<Order> | FindOptionsWhere<Order>[] = base;
  const keyword = dto.keyword?.trim();
  if (keyword) {
    const like = ILike(`%${keyword}%`);
    where = [
      { ...base, code: like },                              // ví dụ: mã đơn
      { ...base, items: { product: { name: like } } },      // tên sản phẩm
    ];
  }

  // (7) Truy vấn + phân trang
  const [data, total] = await this.orderRepository.findAndCount({
    where,
    relations: { items: { product: true } },
    order: { createdAt: 'DESC' },
    skip: (dto.page - 1) * dto.limit,
    take: dto.limit,
  });

  return {
    data,
    meta: {
      total,
      page: dto.page,
      limit: dto.limit,
      totalPages: Math.ceil(total / dto.limit),
    },
  };
}
```

### 5.3 Giải thích các bước

| Bước | Vì sao cần |
|---|---|
| (1) Guard `userId` | Nếu `userId` là `undefined`, TypeORM **bỏ điều kiện user** và trả đơn của mọi người |
| (2) Kiểm tra `from > to` | Tránh truy vấn vô nghĩa, báo lỗi rõ cho client |
| (3) Điều kiện gốc | Đảm bảo mọi truy vấn đều bị giới hạn theo chủ sở hữu |
| (4) `!== undefined` | Chỉ lọc khi client gửi; giá trị hợp lệ như `0` hay chuỗi rỗng không bị bỏ sót nhầm |
| (5) Cuối ngày | `to=2026-09-28` mặc định là `00:00`, nếu không chỉnh sẽ mất đơn trong ngày 28 |
| (6) Mảng `where` | Trong TypeORM, **mảng = OR**, **object = AND** |
| (7) `findAndCount` | Trả cả danh sách và tổng số bản ghi để client phân trang |

### 5.4 Lưu ý về `keyword` trên quan hệ

Khi lọc theo `items.product.name`, các `items` trả về có thể chỉ gồm những item **khớp keyword**, không phải toàn bộ item của đơn. Nếu cần đủ items, dùng `QueryBuilder` với `EXISTS`/subquery:

```ts
qb.andWhere(
  `EXISTS (
     SELECT 1 FROM order_items oi
     JOIN products p ON p.id = oi.product_id
     WHERE oi.order_id = o.id AND p.name ILIKE :kw
   )`,
  { kw: `%${keyword}%` },
);
```

(Tên bảng và cột chỉnh theo entity thực tế.)

---

## 6. Controller

```ts
// src/orders/orders.controller.ts
@Controller('orders')
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  // ⚠️ Route tĩnh (search) PHẢI đặt TRƯỚC route động (:id)
  @Get('search')
  searchOrder(
    @Query() dto: SearchOrderDto,
    @CurrentUser('userId') userId: number,
  ) {
    return this.orderService.search(userId, dto);
  }

  @Get(':id')
  getOrderById(
    @Param('id', ParseIntPipe) orderId: number,
    @CurrentUser('userId') userId: number,
  ) {
    return this.orderService.getOrderById(orderId, userId);
  }
}
```

Vì sao thứ tự quan trọng: nếu `@Get(':id')` đứng trước, chữ `search` sẽ bị hiểu là `id`, `ParseIntPipe` ném 400.

Gọi thử:

```
GET /api/v1/orders/search
GET /api/v1/orders/search?status=PENDING
GET /api/v1/orders/search?keyword=iphone&from=2026-09-01&to=2026-09-28
GET /api/v1/orders/search?status=DONE&page=2&limit=10
```

---

## 7. Exception Filter: đừng che mất message

Lỗi validate trả về mảng message chi tiết trong `getResponse()`. Nếu filter chỉ dùng `exception.message` thì chỉ thấy `Bad Request Exception`.

```ts
// src/common/exception-filters/http-exception.filter.ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const req = ctx.getRequest<Request>();

    const status = exception.getStatus();
    const body = exception.getResponse();
    const message = typeof body === 'string' ? body : (body as any).message;

    res.status(status).json({
      statusCode: status,
      message, // ví dụ: ["status must be one of the following values: ..."]
      path: req.url,
      timestamp: new Date().toISOString(),
    });
  }
}
```

Quy tắc log:

- Chỉ log `method`, `url`, `message`.
- **Không** `console.log(request)` cả object: rất dài và lộ header `Authorization` (token).

---

## 8. Vì sao `where: { status: null }` nguy hiểm

Trong TypeORM 0.3.x, `find*` **bỏ qua** thuộc tính có giá trị `null` hoặc `undefined` trong `where`. Điều kiện biến mất khỏi SQL, không thành `IS NULL`.

```ts
// Bạn nghĩ:  WHERE status IS NULL
// Thực tế:   không có điều kiện status → trả về TẤT CẢ
await repo.find({ where: { status: null as any } });

// Nguy hiểm hơn: userId undefined → trả đơn của MỌI user (rò rỉ dữ liệu)
await repo.find({ where: { user: { id: undefined } } });
```

Muốn lọc đúng các bản ghi NULL, dùng `IsNull()`:

```ts
import { IsNull } from 'typeorm';

await repo.find({ where: { status: IsNull() } }); // WHERE status IS NULL
```

Quy ước trong dự án:

| Giá trị | Ý nghĩa |
|---|---|
| `undefined` | Không lọc |
| `null` | Lọc `IS NULL` (dùng `IsNull()`) |
| Có giá trị | Lọc bằng giá trị đó |

Với query string, client không gửi được `null` thật. Nếu cần lọc "chưa có status", thêm cờ riêng (ví dụ `?noStatus=true`) rồi ánh xạ sang `IsNull()` trong service.

### Test minh hoạ (Jest)

```ts
it('where: { status: null } bị bỏ qua, IsNull() lọc đúng', async () => {
  await repo.save([
    { status: null },
    { status: OrderStatus.PENDING },
    { status: OrderStatus.DONE },
  ]);

  const withNull = await repo.find({ where: { status: null as any } });
  const withIsNull = await repo.find({ where: { status: IsNull() } });

  expect(withNull).toHaveLength(3);   // nguy hiểm: trả về tất cả
  expect(withIsNull).toHaveLength(1); // đúng: chỉ đơn NULL
});
```

---

## 9. Bảng lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| 400 `property xxx should not exist` | DTO thiếu decorator validate | Thêm `@IsOptional()` + decorator kiểu |
| 400 `status must be one of the following values` | Gửi tên key thay vì giá trị enum | Gửi đúng giá trị enum |
| 400 `from must be a Date instance` | Thiếu `transform: true` hoặc `@Type(() => Date)` | Bật `transform`, thêm `@Type` |
| 400 `numeric string is expected` | `@Get(':id')` đứng trước `@Get('search')` | Đổi thứ tự route |
| 400 nhưng message chỉ là `Bad Request Exception` | Filter dùng `exception.message` | Dùng `exception.getResponse()` |
| Trả về đơn của người khác | `userId` là `undefined`, điều kiện bị bỏ | Guard `userId`, kiểm tra key trong JWT payload |
| `Cannot find module 'src/...'` | Import đường dẫn tuyệt đối | Dùng đường dẫn tương đối |
| Mất đơn của ngày `to` | `to` là 00:00 | `setHours(23, 59, 59, 999)` |

---

## 10. Checklist trước khi merge

- [ ] `ValidationPipe` có `transform`, `whitelist`, `forbidNonWhitelisted`
- [ ] Mọi property của DTO đều có decorator validate
- [ ] Enum khai báo một nơi, import bằng đường dẫn tương đối
- [ ] Service có guard `userId` và điều kiện lọc theo user
- [ ] Không gán `null`/`undefined` trực tiếp vào `where`
- [ ] Có `order`, `skip`, `take`, giới hạn `limit` tối đa
- [ ] Route `search` đặt trước `:id`
- [ ] Filter trả về message chi tiết, không log toàn bộ request/token
- [ ] Đã test: không query, từng điều kiện riêng lẻ, kết hợp nhiều điều kiện, giá trị sai
