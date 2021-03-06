## 唐华嶓(bo)个人简历
### 个人信息

- 工作经验：IT从业12年（5年研发+项目/小团队管理、2年互联网创业、5年创业公司技术管理）
- 年龄：31（1989）
- 联系电话：18902490035 （微信同号）
- 邮箱：tanghuabo1989@163.com
- 教育信息：本科 湘南学院 信息与计算科学

### 自我评价

12年IT从业经验，7年团队及部门管理经验。有丰富的团队组建和管理经验。先后在两家互联网公司，组建技术团队、落地团队考核。有管理过20人+产品技术团队。

技术方面，有超过7年的服务化、分布式研发经验。有主持过日访问量近20亿+的广告平台研发、迭代。

### 工作经历

#### 2016~至今，技术合伙人，氪金信息科技（天津）有限公司（Codrim）

移动网盟广告公司，自建了广告平台，支撑公司广告业务。平台定位为网盟广告Saas平台，同FuseClick、Hasoffers等。后因网盟业务萎缩，对外开放搁置。期间还尝试了游戏，微信小程序/小游戏，App等方向。

属于空降加入该公司，加入之后，快速接手研发团队，保证业务的持续，并在随后逐步梳理和优化和团队及项目。

工作职责：

- 技术研发团队搭建、管理，包括招聘、考核、目标制定落地，规范等，公司研发团队最多时接近30人
- 多项目协调推进，之前一度同时有3个左右项目并行
- 参与部分核心编码，加速业务项目落地
- 以合伙人身份参与公司决策

成绩：

- 服务化改造/升级公司广告平台，从原有几百w的点击量/天，实际支撑到近20亿/每天
- 搭建、协调团队，建立小组的管理机制，顺利支撑公司在多个领域的业务探索和开展，包括：网盟、联运、SDK、程序化广告、小程序及小游戏等
- 建立并落地研发绩效考核，提高招聘的效率和团队战斗力

#### 2015~2016，技术经理，合摩信息科技有限公司（爱员工）

以高级研发身份加入，一个月后提升为技术经理，主管整个研发团队，当时CTO欠缺。

#### 2014~2015，互联网创业

移动互联网创业，内容是现在餐馆很常见的二维码点餐。失败主因是不盈利，用户还未形成使用习惯。

#### 2010~2014，软件工程师/项目经理，京信通信

初期研发，后兼任项目经理及小组负责人。负责的项目参与过公司cmmi3评级并通过。

#### 2009~2010，软件工程师，新太科技

Java软件开发工作

### 项目

#### 移动广告平台，氪金，2016~至今

公司核心的网盟广告平台，任职期间持续在迭代。入职时接手项目，已在线上运行，但为单体集中式部署，性能问题严重，经常服务中断。当时公司业务增长迅速，系统问题已经成为关键和迫切的问题。

处理过程：
- 梳理系统，模块拆分，将系统服务化（SpringBoot+Dubbo)
- 设计多期迭代计划，逐步、快速调整上线，减少风险，维持对外服务的正常
- 数据库，缓存，中间键等服务上云（阿里云），减少人力投入，并充分利用云服务的稳定和便捷的扩容
- 发布自动化，解决人工重复引起错误（Jenkins+git)
- 借助云服务，上线服务监控，进一步及时发现问题，提升系统可用

成果：
- 在3个月左右，成功重构上线，解决性能问题，并保持了线上业务的持续
- 使用小型机，多服务部署，在业务量未上升时，服务器成本下降30%左右
- 因支持水平及垂直扩容，顺利支撑了后续业务量的暴涨

更多的方案内容，参考：[广告平台重构设计](/posts/cs-repeate)

#### 统一小程序、游戏&App平台，氪金，2018~至今

前后开发了几十款小程序和App。为了加速后台的开发迭代速度，设计并开发了统一的服务后台。项目使用了SpringCloud作为服务化框架（主要因当时Dubbo长期暂停更新）。

在项目中主要碰到的问题是多需求、多任务同时开发，带来的人员管理和代码管理、发布的等问题。处理的一些措施：

- 按小组为单位，分配人员，任务和需求，每个小组指定负责人，负责人处理需求的排期和进度的收集上报；
- 调整代码分支管理方式，以迭代为单位从主分支建立开发，并使用统一的测试分支，部署测试环境；
- 待上线时，提交PR，经过代码审查无误之后，合并到主分支进行上线；
- 建立后台功能公共化规则，当功能重复使用到3次时，抽取公共功能；

成果：

- 同时支撑数十款小程序/游戏，在线运行
- 缩短开发时间，快速尝试，基本一款小程序，第一版本控制在一周左右时间上线测试；
- 其中一款答题类小程序，当时日自然新增接近1000，同时在线接近1w；

### 技术
- 母语Java，也比较多用Python，其它的前端、游戏、App开发语言都有实际使用
- 熟练 shell、linux命令行操作，部分时间会承担运维工作
- 服务化框架SpringCloud, Apache Dubbo 都有应用到生产环境
- 熟悉Hadoop相关分布式存储及统计处理
- Java开源常见中间键都熟练使用，如：Redis、Memcached、Kafka、Mongo、ES等
- 熟悉容器化技术：Docker, k8s