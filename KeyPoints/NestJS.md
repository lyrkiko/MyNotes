@Controller() 和 @Injectable() 是 NestJS 最核心的两个装饰器，分别对应“入口层”和“服务层”。

@Controller()

- 作用：把一个类标记成“控制器”，专门负责接收请求、匹配路由、拿参数、返回响应。
- 你可以把它理解成 Web 层入口。
- 典型配合是 @Get()、@Post()、@All()、@Body()、@Req()、@Res() 一起用。

@Injectable()

- 作用：把一个类标记成“可被 Nest 依赖注入容器管理的 provider”。
- 你可以把它理解成“业务服务/可复用组件”。
- 被 @Injectable() 标记后，Nest 才能在别的类构造函数里自动注入它。

Summary：
- @Controller()：负责“接请求”
- @Injectable()：负责“干业务”