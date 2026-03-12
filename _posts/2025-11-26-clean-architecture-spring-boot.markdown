---
layout:     post
title:      "Clean Architecture 在 Spring Boot 中的实践"
subtitle:   "Building Maintainable Backend Systems with Layered Design"
date:       2025-11-26 09:00:00
author:     "Tony L."
header-img: "img/post-bg-css.jpg"
tags:
    - Software Engineering
    - Spring Boot
    - Architecture
    - Java
---

## 为什么需要 Clean Architecture？

在快速迭代的项目中，代码腐化是一个常见问题。最初清晰的模块划分逐渐模糊，业务逻辑渗透到 Controller 层，数据库操作散落在 Service 中，最终代码变成一团混乱的"大泥球"（Big Ball of Mud）。

Clean Architecture 由 Robert C. Martin（Uncle Bob）提出，其核心原则是**依赖规则**：源代码依赖只能指向内层，外层的任何变化不应影响内层。

## Spring Boot 分层设计

在 Spring Boot 中，Clean Architecture 通常映射为以下分层：

```
Controller (API Layer)
    ↓
Service (Business Logic Layer)
    ↓
Repository (Data Access Layer)
    ↓
Entity (Domain Model)
```

### Controller 层

Controller 负责接收 HTTP 请求、参数校验和响应格式化。它不应包含任何业务逻辑。

```java
@RestController
@RequestMapping("/api/deals")
@RequiredArgsConstructor
public class DealController {

    private final DealService dealService;

    @GetMapping
    public ApiResponse<Page<DealDTO>> listDeals(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Page<DealDTO> deals = dealService.listDeals(page, size);
        return ApiResponse.success(deals);
    }

    @PostMapping
    public ApiResponse<DealDTO> createDeal(
            @Valid @RequestBody CreateDealRequest request) {
        DealDTO deal = dealService.createDeal(request);
        return ApiResponse.success(deal);
    }
}
```

关键原则：
- Controller **不直接调用 Repository**
- 使用 DTO 接收请求和返回响应，**不暴露 Entity**
- 参数校验使用 `@Valid` 注解，委托给 Bean Validation
- 统一返回 `ApiResponse` 格式

### Service 层

Service 层是业务逻辑的核心。所有复杂的业务规则、数据转换和事务管理都在这里完成。

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class DealService {

    private final DealRepository dealRepository;
    private final StageRepository stageRepository;
    private final DealMapper dealMapper;

    public Page<DealDTO> listDeals(int page, int size) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by("createdAt").descending());
        return dealRepository.findAll(pageable)
            .map(dealMapper::toDTO);
    }

    @Transactional
    public DealDTO createDeal(CreateDealRequest request) {
        Stage stage = stageRepository.findById(request.getStageId())
            .orElseThrow(() -> new ResourceNotFoundException(
                "Stage not found: " + request.getStageId()));

        Deal deal = dealMapper.toEntity(request);
        deal.setStage(stage);
        deal.setPosition(dealRepository.countByStage(stage));

        Deal saved = dealRepository.save(deal);
        return dealMapper.toDTO(saved);
    }
}
```

关键原则：
- **一个 Service 方法 = 一个业务用例**
- 事务边界在 Service 层定义
- 异常处理：抛出自定义业务异常，由全局异常处理器统一处理
- 使用 MapStruct 或手写 Mapper 做 DTO ↔ Entity 转换

### Repository 层

Repository 层封装所有数据库访问逻辑。Spring Data JPA 让这一层变得非常简洁。

```java
public interface DealRepository extends JpaRepository<Deal, UUID> {

    List<Deal> findByStageOrderByPositionAsc(Stage stage);

    int countByStage(Stage stage);

    @Query("SELECT d FROM Deal d WHERE d.owner.id = :ownerId " +
           "AND d.status = :status")
    Page<Deal> findByOwnerAndStatus(
        @Param("ownerId") UUID ownerId,
        @Param("status") DealStatus status,
        Pageable pageable);
}
```

关键原则：
- 优先使用 Spring Data JPA 的方法名推导查询
- 复杂查询使用 `@Query` 注解
- 避免在 Repository 中写业务逻辑

### Entity 层

Entity 是领域模型，代表业务概念。

```java
@Entity
@Table(name = "deals")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Deal extends BaseEntity {

    @Column(nullable = false)
    private String name;

    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    private DealStatus status;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "stage_id")
    private Stage stage;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id")
    private User owner;

    private Integer position;
}
```

## DTO 设计

DTO（Data Transfer Object）是分层架构中的关键角色。它隔离了内部领域模型和外部 API 契约。

```java
// 请求 DTO
public record CreateDealRequest(
    @NotBlank String name,
    @NotNull UUID stageId,
    @Positive BigDecimal amount
) {}

// 响应 DTO
public record DealDTO(
    UUID id,
    String name,
    BigDecimal amount,
    String stageName,
    String ownerName,
    LocalDateTime createdAt
) {}
```

使用 Java Record 让 DTO 定义更加简洁。请求和响应使用不同的 DTO，因为它们的字段往往不同。

## 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(
            ResourceNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(ApiResponse.error(404, ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(
            MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
            .body(ApiResponse.error(400, message));
    }
}
```

## 统一响应格式

```java
@Data
@AllArgsConstructor
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(200, "success", data);
    }

    public static <T> ApiResponse<T> error(int code, String msg) {
        return new ApiResponse<>(code, msg, null);
    }
}
```

## 总结

Clean Architecture 不是教条，而是一种指导原则。在 Spring Boot 中实践 Clean Architecture 的关键在于：严格遵守分层依赖规则、合理使用 DTO 隔离层级、在 Service 层集中业务逻辑、通过全局异常处理器统一错误响应。这些实践能让你的代码库在长期维护中保持健康。

## References

1. Martin, R.C. (2017). "Clean Architecture: A Craftsman's Guide to Software Structure and Design."
2. Spring Boot Reference Documentation. https://docs.spring.io/spring-boot/
