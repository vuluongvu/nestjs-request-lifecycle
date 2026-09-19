# Cài Đặt JWT Cho NestJS — Copy/Paste Được Ngay

Làm theo đúng thứ tự từ trên xuống. Mỗi khối code có thể copy nguyên vào file tương ứng trong dự án của bạn. Ví dụ dùng đăng nhập bằng `username`.

---

## Bước 1: Cài thư viện

```bash
npm install @nestjs/passport passport passport-jwt @nestjs/jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
```

| Thư viện | Vai trò |
|---|---|
| `passport`, `@nestjs/passport` | Nền tảng xác thực |
| `passport-jwt` | Chiến lược xác thực bằng JWT |
| `@nestjs/jwt` | Tạo (sign) và kiểm tra (verify) token |
| `bcrypt` | Băm mật khẩu |

---

## Bước 2: Thêm biến môi trường

Mở file `.env`, thêm 2 dòng:

```env
JWT_SECRET=doi_chuoi_nay_thanh_chuoi_ngau_nhien_that_dai_cua_ban
JWT_EXPIRES_IN=1d
```

> Đổi `JWT_SECRET` thành chuỗi ngẫu nhiên riêng của bạn (có thể tạo nhanh bằng lệnh `openssl rand -hex 32` trong terminal). Không commit `.env` lên Git.

---

## Bước 3: Tạo hàm hash mật khẩu

Tạo file mới: `src/common/utils/hash.util.ts`

```ts
import * as bcrypt from 'bcrypt';

const SALT_ROUNDS = 10;

export async function hashPassword(plainPassword: string): Promise<string> {
  return bcrypt.hash(plainPassword, SALT_ROUNDS);
}

export async function comparePassword(
  plainPassword: string,
  hashedPassword: string,
): Promise<boolean> {
  return bcrypt.compare(plainPassword, hashedPassword);
}
```

**Giải thích:** `hashPassword` biến mật khẩu thường thành chuỗi mã hóa 1 chiều (không thể đảo ngược) để lưu vào DB. `comparePassword` dùng lúc đăng nhập — không "giải mã" mà hash lại mật khẩu vừa nhập rồi so sánh 2 chuỗi hash.

---

## Bước 4: Đảm bảo `User` entity có đủ field

Nếu entity `User` của bạn chưa có, thêm vào (giữ nguyên field đã có, chỉ bổ sung phần thiếu):

```ts
// src/users/entities/user.entity.ts
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';
import { Exclude } from 'class-transformer';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  username: string;

  @Exclude() // không bao giờ trả password về client
  @Column()
  password: string;
}
```

Trong `users.service.ts`, đảm bảo có hàm tạo user (hash password trước khi lưu) và hàm tìm theo username kèm password:

```ts
// src/users/users.service.ts
import { Injectable, ConflictException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';
import { hashPassword } from '../common/utils/hash.util';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async create(dto: { username: string; password: string }): Promise<User> {
    const existing = await this.userRepository.findOneBy({ username: dto.username });
    if (existing) {
      throw new ConflictException('Username đã tồn tại');
    }

    const hashedPassword = await hashPassword(dto.password);
    const user = this.userRepository.create({ ...dto, password: hashedPassword });
    return this.userRepository.save(user);
  }

  // Dùng riêng cho lúc login — password bị @Exclude nên phải select tường minh
  async findByUsernameWithPassword(username: string): Promise<User | null> {
    return this.userRepository
      .createQueryBuilder('user')
      .addSelect('user.password')
      .where('user.username = :username', { username })
      .getOne();
  }
}
```

Trong `users.module.ts`, nhớ export `UsersService`:

```ts
// src/users/users.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // BẮT BUỘC để AuthModule dùng lại được
})
export class UsersModule {}
```

---

## Bước 5: Tạo thư mục `auth` và các file bên trong

Chạy lệnh tạo nhanh (hoặc tự tạo file thủ công theo cấu trúc bên dưới):

```bash
nest g module auth
nest g controller auth
nest g service auth
mkdir src/auth/dto src/auth/strategies src/auth/guards src/auth/decorators
```

Cấu trúc sau khi xong:
```
src/auth/
├── dto/
│   ├── register.dto.ts
│   └── login.dto.ts
├── strategies/
│   └── jwt.strategy.ts
├── guards/
│   └── jwt-auth.guard.ts
├── decorators/
│   └── current-user.decorator.ts
├── auth.controller.ts
├── auth.service.ts
└── auth.module.ts
```

---

## Bước 6: DTO đăng ký & đăng nhập

`src/auth/dto/register.dto.ts`
```ts
import { IsString, IsNotEmpty, MinLength, MaxLength } from 'class-validator';

export class RegisterDto {
  @IsString()
  @IsNotEmpty()
  @MaxLength(100)
  username: string;

  @IsString()
  @MinLength(6, { message: 'Mật khẩu phải có ít nhất 6 ký tự' })
  @MaxLength(32)
  password: string;
}
```

`src/auth/dto/login.dto.ts`
```ts
import { IsString, IsNotEmpty } from 'class-validator';

export class LoginDto {
  @IsString()
  @IsNotEmpty({ message: 'Username không được để trống' })
  username: string;

  @IsString()
  @IsNotEmpty({ message: 'Mật khẩu không được để trống' })
  password: string;
}
```

> **Lưu ý:** `LoginDto` không cần rule kiểm tra mật khẩu mạnh (`@MinLength`...) — chỉ cần kiểm tra có nhập hay không.

---

## Bước 7: `jwt.strategy.ts` — nơi kiểm tra token

`src/auth/strategies/jwt.strategy.ts`
```ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

export interface JwtPayload {
  sub: number;
  username: string;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.get<string>('JWT_SECRET'),
    });
  }

  // Chạy SAU KHI token đã được xác minh hợp lệ (đúng chữ ký, chưa hết hạn)
  // Giá trị return được Nest tự động gắn vào request.user
  async validate(payload: JwtPayload) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

---

## Bước 8: `jwt-auth.guard.ts` — công tắc bảo vệ route

`src/auth/guards/jwt-auth.guard.ts`
```ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

---

## Bước 9: `current-user.decorator.ts` — lấy user hiện tại tiện lợi

`src/auth/decorators/current-user.decorator.ts`
```ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user; // được JwtStrategy.validate() gắn vào
  },
);
```

---

## Bước 10: `auth.service.ts` — logic đăng ký/đăng nhập

`src/auth/auth.service.ts`
```ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { comparePassword } from '../common/utils/hash.util';

@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
  ) {}

  async register(dto: RegisterDto) {
    const user = await this.usersService.create(dto);
    return this.signToken(user.id, user.username);
  }

  async login(dto: LoginDto) {
    const user = await this.usersService.findByUsernameWithPassword(dto.username);
    if (!user) {
      throw new UnauthorizedException('Username hoặc mật khẩu không đúng');
    }

    const isMatch = await comparePassword(dto.password, user.password);
    if (!isMatch) {
      throw new UnauthorizedException('Username hoặc mật khẩu không đúng');
    }

    return this.signToken(user.id, user.username);
  }

  private async signToken(userId: number, username: string) {
    const payload = { sub: userId, username };
    const accessToken = await this.jwtService.signAsync(payload);
    return { accessToken };
  }
}
```

> Thông báo lỗi luôn giống nhau dù sai username hay sai password — tránh lộ thông tin username nào đã tồn tại trong hệ thống.

---

## Bước 11: `auth.module.ts` — kết nối tất cả lại

`src/auth/auth.module.ts`
```ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';
import { JwtStrategy } from './strategies/jwt.strategy';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.get<string>('JWT_SECRET'),
        signOptions: { expiresIn: config.get<string>('JWT_EXPIRES_IN') },
      }),
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

---

## Bước 12: `auth.controller.ts` — route đăng ký/đăng nhập/profile

`src/auth/auth.controller.ts`
```ts
import { Controller, Post, Body, Get, UseGuards, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { CurrentUser } from './decorators/current-user.decorator';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('register')
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto);
  }

  @Post('login')
  @HttpCode(HttpStatus.OK)
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto);
  }

  @Get('me')
  @UseGuards(JwtAuthGuard)
  getProfile(@CurrentUser() user: { userId: number; username: string }) {
    return user;
  }
}
```

---

## Bước 13: Đăng ký vào `app.module.ts`

Mở `src/app.module.ts`, thêm `AuthModule` vào mảng `imports`:

```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersModule } from './users/users.module';
import { AuthModule } from './auth/auth.module'; // ← thêm dòng này

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRootAsync({ /* cấu hình DB đã có sẵn */ }),
    UsersModule,
    AuthModule, // ← thêm dòng này
  ],
})
export class AppModule {}
```

---

## Bước 14: Kiểm tra `main.ts` đã bật `ValidationPipe` chưa

Nếu chưa có, thêm vào `src/main.ts`:

```ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

> Không có bước này, `@IsString()`, `@MinLength()`... trong DTO sẽ **không có tác dụng gì cả** — đây là lỗi rất hay gặp khi mới cài JWT mà quên bước này.

---

## Bước 15: Chạy thử

```bash
npm run start:dev
```

**Đăng ký:**
```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "vu123", "password": "Abc123456"}'
```
Kết quả:
```json
{ "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }
```

**Đăng nhập:**
```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "vu123", "password": "Abc123456"}'
```

**Gọi route được bảo vệ (thay `<token>` bằng `accessToken` vừa nhận):**
```bash
curl http://localhost:3000/auth/me \
  -H "Authorization: Bearer <token>"
```
Kết quả:
```json
{ "userId": 1, "username": "vu123" }
```

---

## Áp dụng bảo vệ cho route khác trong dự án

Chỉ cần 3 dòng ở bất kỳ Controller nào muốn bắt buộc đăng nhập:

```ts
import { UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { CurrentUser } from '../auth/decorators/current-user.decorator';

@Post()
@UseGuards(JwtAuthGuard)
create(@CurrentUser() user: { userId: number }, @Body() dto: CreateSomethingDto) {
  return this.someService.create(user.userId, dto);
}
```

---

## Lỗi thường gặp khi mới cài

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Unauthorized` dù token đúng | Quên đặt `JWT_SECRET` trong `.env`, hoặc `.env` không được load | Kiểm tra `ConfigModule.forRoot()` đã có trong `app.module.ts` |
| DTO không validate | Chưa bật `ValidationPipe` trong `main.ts` | Xem lại Bước 14 |
| `password` vẫn hiện trong response | Chưa bật `ClassSerializerInterceptor` | Thêm `app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)))` vào `main.ts` |
| `Cannot find module '../users/users.service'` | Đường dẫn import sai theo cấu trúc thư mục thực tế | Kiểm tra lại đường dẫn tương đối giữa `auth/` và `users/` |
| `UsersService` không inject được vào `AuthService` | Quên `exports: [UsersService]` trong `users.module.ts` | Xem lại Bước 4 |

---

## Checklist sau khi cài xong

- [ ] `npm install` đủ 6 package ở Bước 1
- [ ] `.env` có `JWT_SECRET` và `JWT_EXPIRES_IN`
- [ ] `UsersModule` có `exports: [UsersService]`
- [ ] `main.ts` đã bật `ValidationPipe`
- [ ] Test được `/auth/register`, `/auth/login`, `/auth/me` bằng `curl` hoặc Postman
- [ ] Route khác trong dự án có thể bảo vệ chỉ bằng `@UseGuards(JwtAuthGuard)`
