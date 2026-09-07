# Bài 3: Xây dựng và sửa lỗi cấu hình Config Server

## 1. Mô tả lỗi

FoodX dựng một Config Server để cấp cấu hình tập trung cho các service. Khi `restaurant-service` gọi tới Config Server thì nhận lỗi 500. Sau khi kiểm tra, có hai lỗi chính:

- Class main thiếu `@EnableConfigServer`.
- Cấu hình Git repository bị sai: `uri` thiếu `.git` và `default-label` đang là `master` trong khi nhánh thật là `main`.

## 2. Sửa lỗi thiếu @EnableConfigServer

Code cũ chỉ có:

```java
@SpringBootApplication
public class ConfigServerApplication {
}
```

Thiếu `@EnableConfigServer` thì ứng dụng vẫn chạy như một Spring Boot app bình thường, nhưng nó chưa được kích hoạt vai trò Config Server. Nghĩa là các endpoint lấy cấu hình như:

```text
GET /restaurant-service/prod
```

sẽ không hoạt động đúng như mong muốn.

Code đã sửa:

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

File code:

```text
src/main/java/com/foodx/configserver/ConfigServerApplication.java
```

## 3. Sửa cấu hình Git URI và default-label

Cấu hình cũ:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/foodx/config-repo
          default-label: master
```

Vấn đề:

- `uri` nên ghi đầy đủ `.git` để trỏ rõ tới Git repository.
- `default-label` sai vì repo đang dùng nhánh `main`, không phải `master`.

Cấu hình đã sửa:

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/foodx/config-repo.git
          default-label: main
```

File cấu hình:

```text
src/main/resources/application.yml
```

## 4. Luồng xử lý khi gọi GET /restaurant-service/prod

Khi `restaurant-service` hoặc tester gọi:

```http
GET http://localhost:8888/restaurant-service/prod
```

Config Server xử lý theo các bước:

1. Nhận request với application name là `restaurant-service` và profile là `prod`.
2. Dựa vào `spring.cloud.config.server.git.uri`, Config Server truy cập repository `https://github.com/foodx/config-repo.git`.
3. Dựa vào `default-label: main`, Config Server đọc cấu hình trên nhánh `main`.
4. Config Server tìm các file phù hợp, ví dụ `application.yml`, `restaurant-service.yml`, `restaurant-service-prod.yml`.
5. Các cấu hình được gom lại thành một response JSON và trả về cho client.

Như vậy `restaurant-service` không cần lưu cấu hình trực tiếp trong project. Khi đổi cấu hình production, chỉ cần sửa Git config repo.

## 5. Cách kiểm thử độc lập

Không cần chạy `restaurant-service` thật vẫn có thể test Config Server:

### Cách 1: Gọi trực tiếp bằng curl

```bash
curl http://localhost:8888/restaurant-service/prod
```

Nếu hoạt động đúng, response sẽ có `name`, `profiles`, `label` và danh sách `propertySources`.

### Cách 2: Test file cụ thể

Có thể kiểm tra Config Server có đọc đúng file production không bằng cách đảm bảo trong response có các key như:

```text
restaurant.menu-cache-ttl-seconds
restaurant.order-timeout-ms
```

### Cách 3: Kiểm tra log khi startup

Khi Config Server chạy, log không được báo lỗi clone Git repo hoặc lỗi không tìm thấy branch `main`.

### Cách 4: Kiểm tra project biên dịch

```bash
./gradlew test
```

## 6. Kết luận

Nguyên nhân lỗi 500 là Config Server chưa được bật đúng vai trò vì thiếu `@EnableConfigServer`, đồng thời cấu hình Git repo sai. Sau khi thêm annotation, sửa URI có `.git` và đổi nhánh sang `main`, Config Server có thể trả cấu hình đúng cho request `/restaurant-service/prod`.
