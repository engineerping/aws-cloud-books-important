# FlexInvest 复现项目 —— 完整开发路线图（基于真实产品文档）
> 作者视角：架构师 + 技术负责人 + 全栈开发  
> 技术栈：Java 25 · Spring Boot 3.3 · React 18 · AWS EKS · Terraform · GitHub Actions  
> 参考文档：HSBC FlexInvest User Guide (26页完整产品文档，已逐页分析)

---

## 一、基于 PDF 的完整产品功能分析

> 这一步是做项目前最重要的一步：作为架构师，你必须吃透产品文档，提炼出每一个技术实体和业务规则。

### 1.1 核心业务实体（从 PDF 截图中提炼）

| 实体 | 关键字段 | 来源页面 |
|------|----------|----------|
| 用户 (User) | 风险承受等级 1~5、投资账户、结算账户 | p3, p15 |
| 基金 (Fund) | 代码、名称、类型、NAV、风险等级、年管理费、资产配置 | p7, p12 |
| 基金类型 | MONEY_MARKET / BOND_INDEX / EQUITY_INDEX / MULTI_ASSET | p4, p10 |
| 持仓 (Holding) | 持有单位数、总市值、未实现盈亏 | p22 |
| 订单 (Order) | 类型(一次性/月定投)、金额、参考号、状态、结算账户 | p9, p14 |
| 投资计划 (Plan) | 月金额、下次扣款日、已完成订单数、总投入金额 | p23 |
| 组合构建 (Portfolio) | 选中基金列表 + 各自分配比例(总和=100%) | p18 |
| 参考资产配置 (ReferenceAssetMix) | 按风险等级推荐的各类资产占比 | p16 |

### 1.2 三种投资路径（核心业务流程）

**路径 A：直接买单只基金（Money Market / Bond Index / Equity Index）**
```
浏览基金列表（支持排序+过滤）→ 查看基金详情（图表/持仓/风险等级/费用）
→ 点击"Invest now" → 选投资类型(月定投/一次性) → 填金额(最低100HKD)+开始日期
→ 选投资账户+结算账户 → Review → 阅读T&C → Confirm → 显示订单参考号
```

**路径 B：直接买多资产组合（Multi-asset Portfolio）**
```
进入 Multi-asset portfolios → 看5个风险等级Tab → 每Tab展示一只组合基金
→ 查看组合详情（资产配置饼图/6个月回报率/风险等级对比） → 点击"Invest now"
→ 同路径A下单流程（最低100HKD）
```

**路径 C：自建组合（Build your own portfolio）【仅风险等级4或5可用】**
```
进入 Build a portfolio → 查看参考资产配置 → 
Step 1/5 选基金（多选，支持排序/过滤，可查看单只基金详情）→
Step 2/5 分配比例（每只基金分配%，必须合计100%）→
Step 3/5 投资细节（月定投/一次性，最低500HKD，填金额/开始日期/账户）→
Step 4/5 逐基金 Review（每只基金单独弹出确认，有序号提示"1 of 4"）→
Step 5/5 汇总 Buy confirmation（每只基金各自订单参考号）
```

### 1.3 持仓与事务管理（从 p22-24 提炼）

**我的持仓 (My Holdings)：**
- 总市值（Total market value）
- 我的交易（My transactions）：显示 pending 笔数
- 我的投资计划（My investment plans）：显示 active 笔数
- 持仓列表（各基金持有情况 + 未实现盈亏）

**我的交易 (My Transactions)：**
- 两个 Tab：Orders（订单）/ Platform fees（平台费）
- 按月份分组展示
- 订单状态：Pending（橙点）/ Cancelled（灰点）/ Completed（绿点）
- 记录展示90天，完整历史看 eStatement（24个月）

**投资计划详情：**
- Fund name、Next contribution date、Monthly amount、Completed orders、Total invested
- 参考详情（Plan creation date、Order reference number）
- 【终止计划】按钮 → 4步确认流程

**取消待处理订单：**
- 只有 Pending 状态可取消
- 显示 Cancel order → 确认页 → 完成

### 1.4 关键业务规则（必须在代码中实现）

| 规则 | 说明 |
|------|------|
| 最低投资金额 | 单只基金：100 HKD；自建组合：500 HKD（总额） |
| 自建组合权限 | 仅风险等级 4（Adventurous）或 5（Speculative）可用 |
| 组合分配比例 | 各基金分配% 必须合计恰好 100% |
| 最后一只基金金额计算 | 总金额 - 其他基金已分配金额（非按比例，避免舍入误差）|
| 周末/节假日处理 | 提交的订单顺延到下一个工作日处理 |
| 月定投扣款日 | 若选定日为周末/节假日，顺延到当月下一个工作日 |
| 交易记录保留 | App 内显示 90 天；eStatement 保留 24 个月 |
| 基金风险等级 | 0~5 级色彩标尺（0=灰、1=深蓝、2=蓝、3=黄、4=橙、5=红）|
| 参考资产配置 | 仅供参考，不是投资建议，用户可自行调整 |

---

## 二、系统架构设计

### 2.1 整体架构（考虑到这是简历展示项目，适度简化但保持专业性）

```
┌───────────────────────────────────────────────────────────────┐
│                    用户 (浏览器)                               │
└──────────────────────────┬────────────────────────────────────┘
                           │ HTTPS (ACM 证书)
┌──────────────────────────▼────────────────────────────────────┐
│              AWS ALB (Application Load Balancer)              │
│              Route 53 → flexinvest.yourdomain.com             │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│                  EKS Cluster (3节点 t3.medium)                 │
│                                                               │
│   ┌────────────────────────────────────────────────────┐     │
│   │              Nginx Ingress Controller               │     │
│   └──────┬──────────────────────┬──────────────────────┘     │
│          │                      │                             │
│   ┌──────▼──────┐       ┌───────▼──────────────────────┐    │
│   │ Frontend    │       │     API Gateway (Spring Cloud │    │
│   │ (React SPA) │       │     Gateway / Kong)           │    │
│   │ Nginx Pod   │       └───────┬──────────────────────┘    │
│   └─────────────┘               │ 路由到各微服务              │
│                     ┌───────────┼───────────┐                │
│              ┌──────▼──┐ ┌─────▼────┐ ┌────▼──────┐         │
│              │user-svc │ │fund-svc  │ │order-svc  │         │
│              │:8081    │ │:8082     │ │:8083      │         │
│              └─────────┘ └──────────┘ └───────────┘         │
│              ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│              │portfolio │ │plan-svc  │ │notify-svc│         │
│              │-svc:8084 │ │:8085     │ │:8086     │         │
│              └─────────┘ └──────────┘ └──────────┘          │
│              ┌──────────────────────────────────────┐        │
│              │  scheduler-svc (月定投/NAV更新) :8087 │        │
│              └──────────────────────────────────────┘        │
└───────────────────────────────────────────────────────────────┘
         │                    │                    │
┌────────▼──────┐   ┌─────────▼──────┐   ┌────────▼──────────┐
│ RDS PostgreSQL│   │ ElastiCache    │   │ Amazon SES        │
│ (主数据库)    │   │ Redis          │   │ (邮件通知)        │
└───────────────┘   └────────────────┘   └───────────────────┘
```

### 2.2 微服务职责划分

| 服务 | 职责 | 关键 API |
|------|------|----------|
| `user-service` | 注册/登录/JWT/用户资料/风险等级 | POST /auth/**, GET/PUT /users/me |
| `fund-service` | 基金目录/NAV历史/资产配置/Top10持仓 | GET /funds/**, GET /funds/{id}/nav-history |
| `order-service` | 单只基金下单、多资产下单、订单取消 | POST /orders, DELETE /orders/{id} |
| `portfolio-service` | 持仓计算、盈亏计算、总市值 | GET /portfolio/me, GET /portfolio/me/holdings |
| `plan-service` | 月定投计划管理、计划终止 | POST/GET/DELETE /plans |
| `notification-service` | 邮件通知（订单确认、月扣款提醒） | 内部事件消费 |
| `scheduler-service` | 月定投自动执行、NAV数据定时更新 | 内部定时任务 |
| `api-gateway` | 路由、JWT验证、限流 | 统一入口 :8080 |

---

## 三、开发环境与工具链准备（第0周，约2天）

### 3.1 必装工具清单

```bash
# ① Java 25（使用 SDKMAN 管理版本）
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install java 25-open
sdk use java 25-open
java --version  # 确认 Java 25

# ② Maven 3.9+
sdk install maven 3.9.6
mvn --version

# ③ Docker Desktop（必须）
# 下载：https://www.docker.com/products/docker-desktop/
docker --version && docker-compose --version

# ④ AWS CLI v2 + 配置
brew install awscli  # macOS；Linux: 参考AWS官网
aws configure
# Access Key ID: [你的 IAM 用户 key]
# Secret Access Key: [你的 secret]
# Default region name: ap-east-1   ← 香港区，离目标用户最近
# Default output format: json

# ⑤ eksctl（EKS 集群管理）
brew tap weaveworks/tap && brew install weaveworks/tap/eksctl
eksctl version

# ⑥ kubectl
brew install kubectl
kubectl version --client

# ⑦ Terraform 1.9+
brew tap hashicorp/tap && brew install hashicorp/tap/terraform
terraform --version

# ⑧ Helm 3（Kubernetes 包管理）
brew install helm
helm version

# ⑨ Node.js 20 LTS（前端）
brew install node@20
node --version && npm --version

# ⑩ IDE: IntelliJ IDEA 2024.x（推荐），安装插件：
#   - Lombok, Spring Boot Dashboard, Docker, Kubernetes, SonarLint
```

### 3.2 GitHub 仓库结构（最终目标）

```
flexinvest/
├── README.md                    # ★ 架构图 + 演示链接，简历必看
├── docs/
│   ├── architecture.png         # 系统架构图（用于简历）
│   └── api-spec/                # OpenAPI YAML 文档
├── infrastructure/              # Terraform IaC（基础设施即代码）
│   ├── modules/
│   │   ├── vpc/
│   │   ├── eks/
│   │   ├── rds/
│   │   └── elasticache/
│   ├── environments/
│   │   └── prod/
│   └── variables.tf
├── services/                    # 所有后端微服务
│   ├── pom.xml                  # 父 POM（统一版本管理）
│   ├── api-gateway/
│   ├── user-service/
│   ├── fund-service/
│   ├── order-service/
│   ├── portfolio-service/
│   ├── plan-service/
│   ├── notification-service/
│   └── scheduler-service/
├── frontend/                    # React SPA
│   ├── src/
│   │   ├── pages/               # 页面组件（对应产品截图每个页面）
│   │   ├── components/          # 复用组件（FundCard, RiskGauge, etc.）
│   │   ├── store/               # Zustand 状态管理
│   │   ├── api/                 # Axios 请求封装
│   │   └── styles/              # Tailwind + 全局样式
│   └── package.json
├── k8s/                         # Kubernetes 部署清单
│   ├── base/                    # 基础配置
│   └── overlays/prod/           # 生产环境覆盖
├── .github/
│   └── workflows/
│       ├── ci.yml               # PR 触发：build + test + lint
│       └── cd.yml               # main 合并触发：build image + deploy to EKS
└── docker-compose.yml           # 本地全栈联调
```

---

## 四、AWS 基础设施搭建（第1~2周）

### 4.1 成本估算（简历展示项目，省钱方案）

| 资源 | 规格 | 月费用(约) |
|------|------|-----------|
| EKS 集群控制平面 | 1 cluster | $73 |
| EC2 节点 | 3 × t3.medium (spot实例) | ~$30 |
| RDS PostgreSQL | db.t3.micro, 20GB | ~$15 |
| ElastiCache Redis | cache.t3.micro | ~$12 |
| ALB | 1个 | ~$16 |
| Route 53 | 1 hosted zone | $0.5 |
| ACM | 免费 | $0 |
| NAT Gateway | 1个 | ~$32 |
| **总计** | | **~$178/月** |

> 💡 **省钱建议**：演示期间使用 Spot 实例；不演示时缩容节点到0；RDS 可在不用时停止。

### 4.2 VPC 网络（Terraform）

```hcl
# infrastructure/modules/vpc/main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"

  name = "flexinvest-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-east-1a", "ap-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = true   # 省钱：只用一个 NAT
  enable_dns_hostnames   = true

  # EKS 需要这些 tag
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/flexinvest-cluster" = "owned"
  }
  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
    "kubernetes.io/cluster/flexinvest-cluster" = "owned"
  }

  tags = { Project = "flexinvest", Env = "prod" }
}
```

### 4.3 EKS 集群（Terraform）

```hcl
# infrastructure/modules/eks/main.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.13.0"

  cluster_name    = "flexinvest-cluster"
  cluster_version = "1.30"

  vpc_id     = var.vpc_id
  subnet_ids = var.private_subnet_ids

  # 允许 kubectl 从本地访问（开发用，生产应通过 Bastion）
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    general = {
      instance_types = ["t3.medium"]
      capacity_type  = "SPOT"    # Spot 实例省70%费用
      min_size       = 0
      max_size       = 5
      desired_size   = 3
    }
  }

  # 启用 IRSA（Pod 使用 IAM Role）
  enable_irsa = true

  tags = { Project = "flexinvest" }
}
```

### 4.4 RDS PostgreSQL（Terraform）

```hcl
# infrastructure/modules/rds/main.tf
module "db" {
  source  = "terraform-aws-modules/rds/aws"
  version = "6.6.0"

  identifier = "flexinvest-db"
  engine         = "postgres"
  engine_version = "16.3"
  instance_class = "db.t3.micro"

  allocated_storage = 20
  db_name   = "flexinvest"
  username  = "flexadmin"
  port      = "5432"

  # 密码存 AWS Secrets Manager，不写死！
  manage_master_user_password = true

  # 私有子网，只允许 EKS 安全组访问
  db_subnet_group_name   = module.vpc.database_subnet_group
  vpc_security_group_ids = [aws_security_group.rds_sg.id]

  multi_az               = false   # 演示项目，省钱
  deletion_protection    = false
  skip_final_snapshot    = true

  tags = { Project = "flexinvest" }
}
```

### 4.5 部署执行顺序

```bash
# 步骤 1：初始化并部署 VPC
cd infrastructure/modules/vpc
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# 步骤 2：部署 EKS（依赖 VPC 输出）
cd ../eks
terraform init
terraform apply

# 步骤 3：配置 kubectl
aws eks update-kubeconfig \
  --region ap-east-1 \
  --name flexinvest-cluster
kubectl get nodes   # 应该看到 3 个 Ready 的节点

# 步骤 4：部署 RDS
cd ../rds
terraform apply

# 步骤 5：安装 EKS 插件
# 5a. AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts && helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=flexinvest-cluster

# 5b. Nginx Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer

# 5c. cert-manager（自动 HTTPS）
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# 获取 Nginx Ingress 的外部 IP
kubectl get svc -n ingress-nginx
# 将这个 IP 配到你域名的 A 记录
```

---

## 五、后端微服务开发（第2~7周）

### 5.1 Maven 父 POM（所有服务继承）

```xml
<!-- services/pom.xml -->
<project>
  <groupId>com.flexinvest</groupId>
  <artifactId>flexinvest-parent</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.2</version>
  </parent>

  <properties>
    <java.version>25</java.version>
    <spring-cloud.version>2023.0.3</spring-cloud.version>
    <mapstruct.version>1.6.0</mapstruct.version>
    <lombok.version>1.18.34</lombok.version>
  </properties>

  <modules>
    <module>api-gateway</module>
    <module>user-service</module>
    <module>fund-service</module>
    <module>order-service</module>
    <module>portfolio-service</module>
    <module>plan-service</module>
    <module>notification-service</module>
    <module>scheduler-service</module>
  </modules>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${spring-cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- 所有服务公共依赖 -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.flywaydb</groupId>
      <artifactId>flyway-database-postgresql</artifactId>
    </dependency>
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct</artifactId>
      <version>${mapstruct.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springdoc</groupId>
      <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
      <version>2.6.0</version>
    </dependency>
    <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <!-- Test -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

---

### 5.2 user-service（第2周）

**数据库 Schema（Flyway 迁移文件）：**

```sql
-- resources/db/migration/V1__create_users.sql
CREATE TABLE users (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email        VARCHAR(255) UNIQUE NOT NULL,
    password     VARCHAR(255) NOT NULL,       -- BCrypt 加密
    full_name    VARCHAR(255) NOT NULL,
    risk_level   SMALLINT,                    -- NULL=未做问卷; 1~5
    -- 1=Conservative, 2=Moderate, 3=Balanced, 4=Adventurous, 5=Speculative
    status       VARCHAR(20) DEFAULT 'ACTIVE',
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE refresh_tokens (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  VARCHAR(255) UNIQUE NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 风险问卷答题记录
CREATE TABLE risk_assessments (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID NOT NULL REFERENCES users(id),
    answers       JSONB NOT NULL,    -- {"q1": "A", "q2": "C", ...}
    total_score   INTEGER NOT NULL,
    risk_level    SMALLINT NOT NULL,
    assessed_at   TIMESTAMPTZ DEFAULT NOW()
);
```

**核心代码结构：**

```
user-service/src/main/java/com/flexinvest/user/
├── UserServiceApplication.java
├── config/
│   ├── SecurityConfig.java          # Spring Security 6 + JWT 过滤器
│   └── JwtProperties.java           # @ConfigurationProperties
├── controller/
│   ├── AuthController.java          # /auth/register, /auth/login, /auth/refresh
│   ├── UserController.java          # /users/me, /users/me/risk-level
│   └── RiskAssessmentController.java # /risk/questions, /risk/submit
├── service/
│   ├── AuthService.java
│   ├── JwtService.java
│   ├── UserService.java
│   └── RiskAssessmentService.java
├── repository/
│   ├── UserRepository.java
│   ├── RefreshTokenRepository.java
│   └── RiskAssessmentRepository.java
├── entity/
│   ├── User.java
│   ├── RefreshToken.java
│   └── RiskAssessment.java
└── dto/
    ├── RegisterRequest.java
    ├── LoginRequest.java
    ├── AuthResponse.java           # access_token, refresh_token, expires_in
    ├── UserProfileResponse.java    # id, email, fullName, riskLevel
    └── RiskSubmitRequest.java      # answers: Map<String, String>
```

**SecurityConfig.java 关键配置：**

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/auth/**",
                    "/actuator/health",
                    "/v3/api-docs/**",
                    "/swagger-ui/**"
                ).permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

**风险评级逻辑（基于文档的5级体系）：**

```java
@Service
public class RiskAssessmentService {

    // 问卷共6题，每题1~5分，总分6~30
    public RiskLevel calculateRiskLevel(int totalScore) {
        if (totalScore <= 9)  return RiskLevel.CONSERVATIVE;    // 1
        if (totalScore <= 15) return RiskLevel.MODERATE;         // 2
        if (totalScore <= 20) return RiskLevel.BALANCED;         // 3
        if (totalScore <= 25) return RiskLevel.ADVENTUROUS;      // 4
        return RiskLevel.SPECULATIVE;                             // 5
    }
}

// 风险等级枚举（对应产品文档标签）
public enum RiskLevel {
    CONSERVATIVE(1, "Conservative"),
    MODERATE(2, "Moderate"),
    BALANCED(3, "Balanced"),
    ADVENTUROUS(4, "Adventurous"),
    SPECULATIVE(5, "Speculative");
    // ...
}
```

**API 端点清单：**
```
POST /auth/register        → 注册（返回 tokens）
POST /auth/login           → 登录
POST /auth/refresh         → 刷新 access_token
POST /auth/logout          → 作废 refresh_token
GET  /users/me             → 当前用户资料（含 riskLevel）
PUT  /users/me             → 更新资料
GET  /risk/questions       → 获取风险问卷题目
POST /risk/submit          → 提交答案 → 返回风险等级 + 更新 user.riskLevel
GET  /risk/assessment/me   → 我的最新风险评估结果
```

---

### 5.3 fund-service（第3周）

**数据库 Schema：**

```sql
-- V1__create_funds.sql
CREATE TABLE funds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(30) UNIQUE NOT NULL,  -- 如 "U50009"
    isin            VARCHAR(20),                  -- 如 "CLASS HC-HKD-ACC"
    name            VARCHAR(300) NOT NULL,
    fund_type       VARCHAR(30) NOT NULL,
    -- MONEY_MARKET | BOND_INDEX | EQUITY_INDEX | MULTI_ASSET
    risk_level      SMALLINT NOT NULL,            -- 0~5
    currency        VARCHAR(5) DEFAULT 'HKD',
    current_nav     DECIMAL(15, 4),
    nav_date        DATE,
    annual_mgmt_fee DECIMAL(6, 4),               -- 如 0.0031 代表 0.31%
    min_investment  DECIMAL(12, 2) DEFAULT 100.00,
    benchmark_index VARCHAR(300),                 -- 追踪指数名称
    market_focus    VARCHAR(200),                 -- 如 "US domestic market"
    description     TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- NAV 历史（用于绘制 3M/6M/1Y/3Y/5Y 图表）
CREATE TABLE fund_nav_history (
    id       BIGSERIAL PRIMARY KEY,
    fund_id  UUID NOT NULL REFERENCES funds(id),
    nav      DECIMAL(15, 4) NOT NULL,
    nav_date DATE NOT NULL,
    UNIQUE(fund_id, nav_date)
);
CREATE INDEX idx_nav_history_fund_date ON fund_nav_history(fund_id, nav_date DESC);

-- 基金资产配置（饼图数据）
CREATE TABLE fund_asset_allocations (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id      UUID NOT NULL REFERENCES funds(id),
    asset_class  VARCHAR(50) NOT NULL,   -- Stocks / Bonds / Cash / Others / Real Estate
    percentage   DECIMAL(6, 2) NOT NULL,
    as_of_date   DATE NOT NULL
);

-- 指数基金的 Top 10 持仓（对应截图 p12 的 Holdings Tab）
CREATE TABLE fund_top_holdings (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id      UUID NOT NULL REFERENCES funds(id),
    holding_name VARCHAR(200) NOT NULL,   -- 如 "Apple Inc"
    weight       DECIMAL(6, 2) NOT NULL,  -- 如 7.03
    as_of_date   DATE NOT NULL,
    sequence     SMALLINT NOT NULL        -- 排名
);

-- 地理分布（Geographical Tab）
CREATE TABLE fund_geo_allocations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id     UUID NOT NULL REFERENCES funds(id),
    region      VARCHAR(100) NOT NULL,
    percentage  DECIMAL(6, 2) NOT NULL,
    as_of_date  DATE NOT NULL
);

-- 行业分布（Sectors Tab）
CREATE TABLE fund_sector_allocations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id     UUID NOT NULL REFERENCES funds(id),
    sector      VARCHAR(100) NOT NULL,
    percentage  DECIMAL(6, 2) NOT NULL,
    as_of_date  DATE NOT NULL
);
```

**种子数据（必须有真实感的测试数据）：**

```sql
-- 基于产品截图中出现的实际基金名称
INSERT INTO funds (code, isin, name, fund_type, risk_level, annual_mgmt_fee, benchmark_index, market_focus) VALUES
-- 货币市场基金 (风险等级1)
('U50001', 'CLASS D-ACC', 'HSBC Global Money Funds - Hong Kong Dollar', 'MONEY_MARKET', 1, 0.0031, NULL, 'Hong Kong Money Market'),

-- 债券指数基金 (风险等级1~2)
('U50003', 'CLASS HC-HKD-ACC', 'HSBC Global Corporate Bond Index Fund', 'BOND_INDEX', 2, 0.0031, 'Bloomberg Global Corporate Bond Index', 'Global Investment Grade Corporates'),
('U50002', 'CLASS HC-HKD-ACC', 'HSBC Global Aggregate Bond Index Fund', 'BOND_INDEX', 1, 0.0025, 'Bloomberg Global Aggregate Bond Index', 'Global Bonds'),

-- 股票指数基金 (风险等级4)
('U50009', 'CLASS HC-HKD-ACC', 'HSBC US Equity Index Fund', 'EQUITY_INDEX', 4, 0.0031, 'S&P 500 Net Total Return Index', 'US domestic market - NYSE and NASDAQ top 500'),
('U50004', 'CLASS HC-HKD-ACC', 'HSBC Global Equity Index Fund', 'EQUITY_INDEX', 4, 0.0040, 'MSCI World Index', 'Global developed markets'),
('U50008', 'CLASS HC-HKD-ACC', 'Hang Seng Index Fund', 'EQUITY_INDEX', 5, 0.0050, 'Hang Seng Index', 'Hong Kong'),

-- 多资产组合 (风险等级1~5，对应5个Tab)
('U50015', 'CLASS BC-HKD-ACC', 'HSBC Portfolios - World Selection 1', 'MULTI_ASSET', 1, 0.0060, NULL, 'Conservative multi-asset'),
('U50016', 'CLASS BC-HKD-ACC', 'HSBC Portfolios - World Selection 2', 'MULTI_ASSET', 2, 0.0060, NULL, 'Moderately conservative multi-asset'),
('U50017', 'CLASS BC-HKD-ACC', 'HSBC Portfolios - World Selection 3', 'MULTI_ASSET', 3, 0.0060, NULL, 'Balanced multi-asset'),
('U50018', 'CLASS BC-HKD-ACC', 'HSBC Portfolios - World Selection 4', 'MULTI_ASSET', 4, 0.0060, NULL, 'Medium to high risk multi-asset'),
('U50019', 'CLASS BC-HKD-ACC', 'HSBC Portfolios - World Selection 5', 'MULTI_ASSET', 5, 0.0060, NULL, 'High risk multi-asset');
```

**关键 API 端点：**
```
GET  /funds                           → 基金列表（?type=EQUITY_INDEX&riskLevel=4&sortBy=RISK_LEVEL）
GET  /funds/{id}                      → 基金详情（含资产配置+费用）
GET  /funds/{id}/nav-history          → NAV历史（?period=3M|6M|1Y|3Y|5Y）
GET  /funds/{id}/asset-allocation     → 资产配置（饼图数据）
GET  /funds/{id}/top-holdings         → Top10持仓（Holdings Tab）
GET  /funds/{id}/geo-allocation       → 地理分布（Geographical Tab）
GET  /funds/{id}/sector-allocation    → 行业分布（Sectors Tab）
GET  /funds/multi-asset               → 5只多资产组合（对应5个风险等级Tab）
GET  /funds/reference-asset-mix       → 参考资产配置（?riskLevel=4，Build Portfolio用）
```

---

### 5.4 order-service（第4周）

**数据库 Schema：**

```sql
-- V1__create_orders.sql
CREATE TABLE orders (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference_number VARCHAR(30) UNIQUE NOT NULL,  -- 如 P-778974, 2022090xx
    user_id          UUID NOT NULL,
    fund_id          UUID NOT NULL,
    order_type       VARCHAR(20) NOT NULL,  -- ONE_TIME | MONTHLY_PLAN
    investment_type  VARCHAR(20) NOT NULL,  -- BUY | SELL
    amount           DECIMAL(15, 2),        -- 投资金额（HKD）
    nav_at_order     DECIMAL(15, 4),        -- 下单时NAV
    executed_units   DECIMAL(18, 6),        -- 成交单位数（金额/NAV）
    investment_account VARCHAR(100),        -- 如 "HSBC One Investment Services"
    settlement_account VARCHAR(100),        -- 如 "Default settlement account"
    status           VARCHAR(20) DEFAULT 'PENDING',
    -- PENDING | PROCESSING | COMPLETED | CANCELLED | FAILED
    order_date       DATE NOT NULL,
    settlement_date  DATE,                  -- T+2 (跳过周末/节假日)
    plan_id          UUID,                  -- 若来自月定投计划
    created_at       TIMESTAMPTZ DEFAULT NOW(),
    completed_at     TIMESTAMPTZ
);

CREATE INDEX idx_orders_user_status ON orders(user_id, status);
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date DESC);
```

**订单参考号生成（复现产品中的格式）：**

```java
@Service
public class OrderReferenceGenerator {

    // 一次性订单：P-778974（P + 6位随机数）
    // 月定投订单：20220908055008（年月日时分秒+随机）
    public String generate(OrderType type) {
        if (type == OrderType.ONE_TIME) {
            return "P-" + String.format("%06d",
                ThreadLocalRandom.current().nextInt(100000, 999999));
        } else {
            return LocalDateTime.now()
                .format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"))
                + String.format("%03d",
                    ThreadLocalRandom.current().nextInt(0, 999));
        }
    }
}
```

**核心业务逻辑：**

```java
@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final FundServiceClient fundClient;   // OpenFeign 调用
    private final ApplicationEventPublisher eventPublisher;

    public OrderResponse placeOrder(UUID userId, PlaceOrderRequest req) {
        // 1. 获取基金信息（NAV, 风险等级等）
        FundDto fund = fundClient.getFundById(req.getFundId());

        // 2. 校验最低金额
        BigDecimal minAmount = req.isPortfolioOrder()
            ? new BigDecimal("500")    // 自建组合最低500HKD
            : new BigDecimal("100");   // 单只基金最低100HKD
        if (req.getAmount().compareTo(minAmount) < 0) {
            throw new BusinessException("Minimum investment amount is " + minAmount + " HKD");
        }

        // 3. 计算成交单位数
        BigDecimal units = req.getAmount()
            .divide(fund.getCurrentNav(), 6, RoundingMode.DOWN);

        // 4. 计算结算日（T+2，跳过周末和节假日）
        LocalDate settlementDate = calculateSettlementDate(LocalDate.now(), 2);

        // 5. 创建订单
        Order order = Order.builder()
            .referenceNumber(referenceGenerator.generate(req.getOrderType()))
            .userId(userId)
            .fundId(req.getFundId())
            .orderType(req.getOrderType())
            .investmentType(InvestmentType.BUY)
            .amount(req.getAmount())
            .navAtOrder(fund.getCurrentNav())
            .executedUnits(units)
            .investmentAccount(req.getInvestmentAccount())
            .settlementAccount(req.getSettlementAccount())
            .status(OrderStatus.PENDING)
            .orderDate(LocalDate.now())
            .settlementDate(settlementDate)
            .planId(req.getPlanId())
            .build();

        Order saved = orderRepository.save(order);

        // 6. 发布事件（portfolio-service 订阅，更新持仓）
        eventPublisher.publishEvent(new OrderCreatedEvent(saved));

        return orderMapper.toResponse(saved);
    }

    public void cancelOrder(UUID userId, UUID orderId) {
        Order order = orderRepository.findByIdAndUserId(orderId, userId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));

        if (order.getStatus() != OrderStatus.PENDING) {
            throw new BusinessException("Only PENDING orders can be cancelled");
        }

        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);

        eventPublisher.publishEvent(new OrderCancelledEvent(order));
    }

    // 周末/节假日跳过计算（简化版，可接入节假日 API）
    private LocalDate calculateSettlementDate(LocalDate from, int businessDays) {
        LocalDate date = from;
        int count = 0;
        while (count < businessDays) {
            date = date.plusDays(1);
            if (date.getDayOfWeek() != DayOfWeek.SATURDAY
                && date.getDayOfWeek() != DayOfWeek.SUNDAY) {
                count++;
            }
        }
        return date;
    }
}
```

**API 端点清单：**
```
POST /orders                  → 下单（单只基金：一次性/月定投）
GET  /orders                  → 订单列表（?status=PENDING&page=0&size=20）
GET  /orders/{id}             → 订单详情
DELETE /orders/{id}           → 取消待处理订单（只有PENDING可取消）

POST /orders/portfolio        → 自建组合下单（批量，返回多个订单）
```

---

### 5.5 plan-service（第4~5周）

**数据库 Schema：**

```sql
CREATE TABLE investment_plans (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id              UUID NOT NULL,
    fund_id              UUID NOT NULL,
    reference_number     VARCHAR(30) UNIQUE NOT NULL,  -- 月定投计划参考号
    monthly_amount       DECIMAL(15, 2) NOT NULL,
    next_contribution_date DATE NOT NULL,
    investment_account   VARCHAR(100),
    settlement_account   VARCHAR(100),
    status               VARCHAR(20) DEFAULT 'ACTIVE',  -- ACTIVE | PAUSED | TERMINATED
    completed_orders     INTEGER DEFAULT 0,
    total_invested       DECIMAL(15, 2) DEFAULT 0,
    plan_creation_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    terminated_at        TIMESTAMPTZ,
    created_at           TIMESTAMPTZ DEFAULT NOW()
);
```

**API 端点：**
```
POST   /plans                  → 创建月定投计划（关联订单）
GET    /plans                  → 我的活跃计划列表
GET    /plans/{id}             → 计划详情（含参考详情、下次扣款日、已完成笔数）
DELETE /plans/{id}             → 终止计划（4步确认后调用）
```

---

### 5.6 portfolio-service（第5周）

**数据库 Schema：**

```sql
CREATE TABLE holdings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    fund_id         UUID NOT NULL,
    total_units     DECIMAL(18, 6) DEFAULT 0,
    avg_cost_nav    DECIMAL(15, 4),
    total_invested  DECIMAL(15, 2) DEFAULT 0,
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, fund_id)
);
```

**持仓计算核心：**

```java
@Service
@RequiredArgsConstructor
public class PortfolioService {

    private final HoldingRepository holdingRepo;
    private final FundServiceClient fundClient;

    // 对应截图 p22 的 "My holdings" 页面数据
    public PortfolioSummaryResponse getSummary(UUID userId) {
        List<Holding> holdings = holdingRepo.findByUserIdAndTotalUnitsGreaterThan(
            userId, BigDecimal.ZERO);

        if (holdings.isEmpty()) {
            return PortfolioSummaryResponse.empty();
        }

        // 批量查询当前 NAV
        List<UUID> fundIds = holdings.stream().map(Holding::getFundId).toList();
        Map<UUID, BigDecimal> navMap = fundClient.getBatchCurrentNav(fundIds);

        BigDecimal totalMarketValue = BigDecimal.ZERO;
        BigDecimal totalCost = BigDecimal.ZERO;
        List<HoldingDetail> details = new ArrayList<>();

        for (Holding h : holdings) {
            BigDecimal currentNav = navMap.getOrDefault(h.getFundId(), BigDecimal.ZERO);
            BigDecimal marketValue = h.getTotalUnits().multiply(currentNav)
                .setScale(2, RoundingMode.HALF_UP);
            BigDecimal unrealisedGL = marketValue.subtract(h.getTotalInvested());
            BigDecimal unrealisedGLPct = h.getTotalInvested().compareTo(BigDecimal.ZERO) > 0
                ? unrealisedGL.divide(h.getTotalInvested(), 4, RoundingMode.HALF_UP)
                    .multiply(new BigDecimal("100"))
                : BigDecimal.ZERO;

            totalMarketValue = totalMarketValue.add(marketValue);
            totalCost = totalCost.add(h.getTotalInvested());

            details.add(HoldingDetail.builder()
                .fundId(h.getFundId())
                .totalUnits(h.getTotalUnits())
                .currentNav(currentNav)
                .marketValue(marketValue)
                .totalInvested(h.getTotalInvested())
                .unrealisedGainLoss(unrealisedGL)       // 未实现盈亏（HKD）
                .unrealisedGainLossPct(unrealisedGLPct) // 未实现盈亏（%）
                .build());
        }

        return PortfolioSummaryResponse.builder()
            .totalMarketValue(totalMarketValue)     // 对应截图的 "Total market value"
            .totalCost(totalCost)
            .totalUnrealisedGL(totalMarketValue.subtract(totalCost))
            .holdings(details)
            .build();
    }

    // 监听订单创建事件，更新持仓（内部事件）
    @EventListener
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        Holding holding = holdingRepo
            .findByUserIdAndFundId(event.getUserId(), event.getFundId())
            .orElseGet(() -> Holding.builder()
                .userId(event.getUserId())
                .fundId(event.getFundId())
                .totalUnits(BigDecimal.ZERO)
                .totalInvested(BigDecimal.ZERO)
                .build());

        holding.setTotalUnits(holding.getTotalUnits().add(event.getExecutedUnits()));
        holding.setTotalInvested(holding.getTotalInvested().add(event.getAmount()));

        // 重算平均成本价
        if (holding.getTotalUnits().compareTo(BigDecimal.ZERO) > 0) {
            holding.setAvgCostNav(holding.getTotalInvested()
                .divide(holding.getTotalUnits(), 4, RoundingMode.HALF_UP));
        }

        holdingRepo.save(holding);
    }
}
```

**API 端点：**
```
GET /portfolio/me             → 持仓汇总（总市值、总盈亏、持仓列表）
GET /portfolio/me/holdings    → 持仓明细列表
```

---

### 5.7 scheduler-service（第6周）

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MonthlyInvestmentScheduler {

    private final PlanServiceClient planClient;
    private final OrderServiceClient orderClient;

    // 每天凌晨1点检查当天需要执行的月定投计划
    @Scheduled(cron = "0 0 1 * * *", zone = "Asia/Hong_Kong")
    public void executeMonthlyPlans() {
        LocalDate today = LocalDate.now(ZoneId.of("Asia/Hong_Kong"));

        // 如果今天是周末，不执行（等周一）
        if (today.getDayOfWeek() == DayOfWeek.SATURDAY
            || today.getDayOfWeek() == DayOfWeek.SUNDAY) {
            return;
        }

        List<ActivePlan> duePlans = planClient.getPlansDueToday(today);
        log.info("Processing {} monthly investment plans for {}", duePlans.size(), today);

        for (ActivePlan plan : duePlans) {
            try {
                orderClient.executeMonthlyPlan(plan.getId());
                log.info("Executed plan {} for user {}", plan.getId(), plan.getUserId());
            } catch (Exception e) {
                log.error("Failed to execute plan {}: {}", plan.getId(), e.getMessage());
                // 失败记录到 dead letter，人工介入
            }
        }
    }

    // 每天15:00 更新基金 NAV（模拟 T+1 NAV 更新）
    @Scheduled(cron = "0 0 15 * * MON-FRI", zone = "Asia/Hong_Kong")
    public void updateFundNav() {
        // 调用 fund-service 更新 NAV（实际项目对接彭博/路透数据源）
        // 本项目用随机波动模拟
        log.info("Updating fund NAV prices...");
    }
}
```

---

### 5.8 api-gateway（第6周）

使用 Spring Cloud Gateway，统一处理：

```yaml
# api-gateway/resources/application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://user-service:8081
          predicates:
            - Path=/api/auth/**, /api/users/**, /api/risk/**
        - id: fund-service
          uri: http://fund-service:8082
          predicates:
            - Path=/api/funds/**
        - id: order-service
          uri: http://order-service:8083
          predicates:
            - Path=/api/orders/**
        - id: portfolio-service
          uri: http://portfolio-service:8084
          predicates:
            - Path=/api/portfolio/**
        - id: plan-service
          uri: http://plan-service:8085
          predicates:
            - Path=/api/plans/**
      default-filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 20
            redis-rate-limiter.burstCapacity: 40
        - AddResponseHeader=X-App-Version, 1.0.0
```

---

## 六、前端开发（第5~8周，可与后端并行）

### 6.1 前端技术栈选型

| 技术 | 选型 | 理由 |
|------|------|------|
| 框架 | React 18 + TypeScript | 主流，简历加分 |
| 路由 | React Router v6 | 标准 |
| 状态管理 | Zustand | 轻量，比Redux简单 |
| HTTP | Axios + React Query | 缓存+加载状态管理 |
| UI 组件 | Tailwind CSS + shadcn/ui | 快速，可高度定制 |
| 图表 | Recharts | 绘制NAV折线图、资产配置饼图 |
| 图标 | Lucide React | 与产品截图风格接近 |
| 构建 | Vite | 快 |

### 6.2 页面路由规划（完全对应产品截图）

```
/                          → 主页（FlexInvest Dashboard）
  └── Total market value 入口
  └── Invest in individual funds
      ├── /funds/money-market             → 货币市场基金列表
      ├── /funds/bond-index               → 债券指数基金列表
      ├── /funds/equity-index             → 股票指数基金列表
      └── /funds/all                      → 全部基金列表（支持排序/过滤）
  └── Invest in portfolios
      ├── /portfolio/multi-asset          → 多资产组合（5个Tab）
      └── /portfolio/build                → 自建组合（仅风险4/5可见）
/funds/:id                 → 基金详情页（图表+持仓+风险等级+费用）
/order                     → 下单页（月定投/一次性）
/order/review              → Review 页
/order/confirm             → 确认+T&C 页
/order/success             → 订单成功页（显示参考号）
/holdings                  → 我的持仓（My Holdings）
/transactions              → 我的交易（My Transactions）
/plans                     → 我的投资计划（My Investment Plans）
/plans/:id                 → 计划详情（可终止）
/risk                      → 风险问卷
/auth/login                → 登录
/auth/register             → 注册
```

### 6.3 关键组件设计

**RiskGauge 组件（产品截图 p7、p12 的风险等级标尺）：**

```tsx
// components/RiskGauge.tsx
interface RiskGaugeProps {
  productRiskLevel: number;  // 产品风险等级（橙色倒三角 ▼）
  userRiskLevel: number;     // 用户风险承受（绿色倒三角 ▲）
}

export const RiskGauge: React.FC<RiskGaugeProps> = ({
  productRiskLevel, userRiskLevel
}) => {
  // 6格色块：0(灰) 1(深蓝) 2(蓝) 3(黄) 4(橙) 5(红)
  const colors = ['#9CA3AF', '#1E3A5F', '#3B82F6', '#EAB308', '#F97316', '#EF4444'];
  const labels = ['0', '1', '2', '3', '4', '5'];

  return (
    <div className="risk-gauge">
      <div className="flex gap-1">
        {colors.map((color, i) => (
          <div key={i} className="relative flex-1 h-6 rounded-sm"
               style={{ backgroundColor: color }}>
            {/* 产品风险等级指示器（倒三角朝下） */}
            {i === productRiskLevel && (
              <div className="absolute -top-4 left-1/2 transform -translate-x-1/2
                              text-black text-xs">▼</div>
            )}
            {/* 用户风险等级指示器（倒三角朝上） */}
            {i === userRiskLevel && (
              <div className="absolute -bottom-4 left-1/2 transform -translate-x-1/2
                              text-green-600 text-xs">▲</div>
            )}
          </div>
        ))}
      </div>
      <div className="flex justify-between mt-6 text-xs text-gray-500">
        <span>Product risk level</span>
        <span>Your risk tolerance</span>
      </div>
      {productRiskLevel <= userRiskLevel ? (
        <p className="text-green-600 text-sm mt-2">
          ✓ This fund is within your risk tolerance level.
        </p>
      ) : (
        <p className="text-orange-500 text-sm mt-2">
          ⚠ This fund exceeds your risk tolerance level.
        </p>
      )}
    </div>
  );
};
```

**PortfolioAllocation 组件（Build a Portfolio 的分配比例输入）：**

```tsx
// 对应截图 p18：各基金分配 % 必须合计 100%
export const FundAllocationForm: React.FC = () => {
  const { selectedFunds } = usePortfolioStore();
  const [allocations, setAllocations] = useState<Record<string, number>>(
    Object.fromEntries(selectedFunds.map(f => [f.id, 0]))
  );

  const total = Object.values(allocations).reduce((sum, v) => sum + v, 0);
  const isValid = Math.abs(total - 100) < 0.01; // 允许 0.01% 浮点误差

  const handleChange = (fundId: string, value: number) => {
    setAllocations(prev => ({ ...prev, [fundId]: value }));
  };

  return (
    <div>
      {selectedFunds.map(fund => (
        <div key={fund.id} className="flex items-center justify-between py-3 border-b">
          <div>
            <p className="font-medium text-sm">{fund.name}</p>
            <p className="text-xs text-gray-500">Risk level {fund.riskLevel}</p>
          </div>
          <div className="flex items-center gap-2">
            <span className="text-sm text-gray-600">Fund allocation</span>
            <input
              type="number"
              min={0}
              max={100}
              value={allocations[fund.id] || ''}
              onChange={e => handleChange(fund.id, Number(e.target.value))}
              className="w-16 text-right border rounded px-2 py-1"
            />
            <span className="text-sm">%</span>
          </div>
        </div>
      ))}

      <div className={`mt-4 text-right font-semibold ${isValid ? 'text-green-600' : 'text-red-500'}`}>
        {total}% of 100% allocated in selected funds
      </div>

      <button
        disabled={!isValid}
        className={`w-full mt-4 py-3 rounded-lg font-medium ${
          isValid ? 'bg-red-600 text-white' : 'bg-gray-200 text-gray-400 cursor-not-allowed'
        }`}
      >
        Continue
      </button>
    </div>
  );
};
```

**NAV 图表组件（对应截图 p7/p12 的 Performance Chart）：**

```tsx
// components/NavChart.tsx - 支持 3M/6M/1Y/3Y/5Y 切换
import { LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer } from 'recharts';

const periods = ['3M', '6M', '1Y', '3Y', '5Y'];

export const NavChart: React.FC<{ fundId: string }> = ({ fundId }) => {
  const [period, setPeriod] = useState<string>('3M');
  const { data } = useQuery(['nav-history', fundId, period],
    () => fundApi.getNavHistory(fundId, period));

  // 将NAV转换为百分比变化（从基准日开始）
  const chartData = useMemo(() => {
    if (!data?.length) return [];
    const baseNav = data[0].nav;
    return data.map(d => ({
      date: d.navDate,
      pct: ((d.nav - baseNav) / baseNav * 100).toFixed(2)
    }));
  }, [data]);

  return (
    <div>
      <div className="flex gap-4 mb-4">
        {periods.map(p => (
          <button key={p}
            onClick={() => setPeriod(p)}
            className={`text-sm font-medium px-2 py-1 ${
              period === p ? 'text-red-600 border-b-2 border-red-600' : 'text-gray-500'
            }`}>{p}</button>
        ))}
      </div>
      <ResponsiveContainer width="100%" height={200}>
        <LineChart data={chartData}>
          <XAxis dataKey="date" tick={{ fontSize: 10 }} />
          <YAxis tick={{ fontSize: 10 }} tickFormatter={v => `${v}%`} />
          <Tooltip formatter={(v) => [`${v}%`, 'Return']} />
          <Line type="monotone" dataKey="pct" stroke="#3B82F6" dot={false} strokeWidth={1.5} />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
};
```

---

## 七、Kubernetes 部署清单（第7周）

### 7.1 每个微服务的标准 K8s 清单

```yaml
# k8s/base/user-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8081"
    spec:
      containers:
      - name: user-service
        image: <YOUR_ECR_URI>/user-service:latest
        ports:
        - containerPort: 8081
        env:
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8081
          initialDelaySeconds: 20
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 8081
    targetPort: 8081
```

### 7.2 Ingress（HTTPS 路由）

```yaml
# k8s/base/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flexinvest-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  tls:
  - hosts:
    - flexinvest.yourdomain.com
    secretName: flexinvest-tls
  rules:
  - host: flexinvest.yourdomain.com
    http:
      paths:
      # 前端静态资源
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      # 所有 API 请求走网关
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 8080
```

### 7.3 Secrets 管理

```bash
# 使用 AWS Secrets Manager + External Secrets Operator
# 不要把明文密码写入 YAML 文件！

# 安装 External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace

# 创建 SecretStore（指向 AWS Secrets Manager）
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-east-1
EOF
```

---

## 八、CI/CD 流水线（GitHub Actions）

### 8.1 CI 流程（PR 触发）

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up Java 25
      uses: actions/setup-java@v4
      with:
        java-version: '25'
        distribution: 'temurin'
        cache: 'maven'

    - name: Build and Test
      run: mvn -B clean verify --file services/pom.xml
      # 包含：编译 + 单元测试 + 集成测试

    - name: SonarQube Analysis
      run: mvn sonar:sonar -Dsonar.projectKey=flexinvest

    - name: Frontend Build
      working-directory: frontend
      run: |
        npm ci
        npm run type-check
        npm run lint
        npm run build
```

### 8.2 CD 流程（合并到 main 触发）

```yaml
# .github/workflows/cd.yml
name: CD
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-east-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build and Push Docker Images
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        SERVICES="user-service fund-service order-service portfolio-service plan-service"
        for SERVICE in $SERVICES; do
          docker build -t $ECR_REGISTRY/$SERVICE:$IMAGE_TAG ./services/$SERVICE
          docker push $ECR_REGISTRY/$SERVICE:$IMAGE_TAG
        done

        # 前端
        docker build -t $ECR_REGISTRY/frontend:$IMAGE_TAG ./frontend
        docker push $ECR_REGISTRY/frontend:$IMAGE_TAG

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --region ap-east-1 --name flexinvest-cluster

    - name: Deploy to EKS (Kustomize)
      env:
        IMAGE_TAG: ${{ github.sha }}
      run: |
        cd k8s/overlays/prod
        kustomize edit set image "*:IMAGE_TAG=*:$IMAGE_TAG"
        kubectl apply -k .
        kubectl rollout status deployment/user-service -n flexinvest
        kubectl rollout status deployment/fund-service -n flexinvest
        # ... 其他服务
```

---

## 九、整体开发时间表（16周计划）

| 周次 | 工作内容 | 里程碑 |
|------|----------|--------|
| 第0周 | 工具安装、仓库创建、开发规范文档、架构设计 | ✅ 项目启动 |
| 第1~2周 | AWS 基础设施 (VPC/EKS/RDS/Redis)，Terraform，K8s 基础组件 | ✅ 云环境就绪 |
| 第2~3周 | user-service（注册/登录/JWT/风险问卷） | ✅ 认证模块 |
| 第3~4周 | fund-service（基金目录/NAV历史/资产配置/Top10/种子数据） | ✅ 基金数据 |
| 第4周 | order-service（下单/取消，单只基金和组合两种路径） | ✅ 交易核心 |
| 第5周 | plan-service（月定投计划管理）+ portfolio-service（持仓计算） | ✅ 持仓管理 |
| 第5~6周 | 前端：认证页 + FlexInvest 主页 + 基金列表页（带排序/过滤） | ✅ 前端启动 |
| 第6周 | 前端：基金详情页（图表+风险标尺+资产配置）+ 下单流程(4步) | ✅ 核心流程 |
| 第7周 | 前端：Build a Portfolio 流程(5步) + My Holdings + Transactions | ✅ 全流程打通 |
| 第8周 | scheduler-service + notification-service（邮件） | ✅ 自动化 |
| 第9周 | K8s 生产清单完善 + CI/CD 流水线 | ✅ DevOps 就绪 |
| 第10周 | 端到端测试 + 性能测试 + 安全扫描 | ✅ 测试完成 |
| 第11周 | 生产部署 + 监控配置(Prometheus/Grafana) + 域名/HTTPS | ✅ 上线 |
| 第12周 | 演示账号 + 种子数据完善 + README 架构图撰写 | ✅ **简历就绪** |
| 第13~16周 | 功能优化、性能调优、文档完善 | 持续迭代 |

---

## 十、简历展示关键要素

### 10.1 README 必须包含的内容

```markdown
## FlexInvest - 智能基金投资平台（Java 25 / Spring Boot / EKS）

**Live Demo**: https://flexinvest.yourdomain.com  
**Demo Account**: demo@flexinvest.com / Demo1234!

### 技术亮点
- 微服务架构（8个服务），部署于 AWS EKS（Kubernetes 1.30）
- Java 25 + Spring Boot 3.3，服务间 OpenFeign 通信
- JWT 无状态认证 + Spring Security 6
- Terraform IaC 管理全部云资源（VPC/EKS/RDS/ElastiCache）
- GitHub Actions CI/CD，Push-to-Deploy 全自动化
- PostgreSQL + Redis，Flyway 数据库版本管理
- 完整投资流程：风险问卷 → 基金选择 → 下单 → 持仓管理
```

### 10.2 面试中要能说出口的架构决策

| 决策 | 选择 | 为什么（面试时这样回答）|
|------|------|------------------------|
| 微服务 vs 单体 | 微服务 | 独立部署、故障隔离；order-service 和 portfolio-service 负载不同 |
| 服务通信 | 同步(OpenFeign) + 异步(ApplicationEvent) | 查询用同步保证实时性；持仓更新用异步解耦 |
| 认证方案 | JWT | 无状态，EKS 横向扩展时无需共享 Session |
| 数据库 | PostgreSQL | JSONB 存问卷答案；财务数据需要 ACID 事务 |
| 缓存 | Redis | 基金NAV每日更新，缓存后减少 DB 压力 |
| 基础设施 | Terraform | IaC 可重现，版本可控，符合生产实践 |
| 前端状态 | Zustand + React Query | 服务器状态和客户端状态分离管理 |

---

## 十一、下一步行动清单

**现在立即开始（今天）：**
1. `mkdir flexinvest && cd flexinvest && git init`
2. 注册 AWS 账号，创建 IAM 用户，记录 Access Key
3. 安装本文第三章所有工具，逐一验证版本

**本周内完成：**
4. 搭建 Terraform 结构，先跑通 VPC + EKS 部署
5. 创建 Maven 父 POM，搭建 user-service 骨架
6. 本地 Docker Compose 能跑起来 user-service + PostgreSQL

**每完成一个里程碑：**
7. 写 commit 信息时使用规范格式：`feat(order): add monthly plan creation`
8. 在 README 的 changelog 里记录，面试时可以展示进度

**有任何卡住的地方：**
直接把具体的报错/代码贴给我，我以架构师视角帮你 debug 和优化。
```
