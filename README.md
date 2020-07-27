Spring Boot提倡约定大于配置。要开始工作，只需在项目中添加flowable-spring-boot-starter或flowable-spring-boot-starter-rest依赖。
如果不需要引入所有的引擎，可以查看其它的Flowable starter。 如使用Maven：
<dependency>
    <groupId>org.flowable</groupId>
    <artifactId>flowable-spring-boot-starter</artifactId>
    <version>${flowable.version}</version>
</dependency>
就这么简单。这个依赖会自动向classpath添加正确的Flowable与Spring依赖。现在可以编写Spring Boot应用了：

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
Flowable需要数据库来存储数据。运行上面的代码会得到异常提示，指出需要在classpath中添加数据库驱动依赖。
更换数据源与连接池
上面也提到过，Spring Boot的约定大于配置。默认情况下，如果classpath中只有H2，就会创建内存数据库，并传递给Flowable流程引擎配置。

只要添加一个数据库驱动的依赖并提供数据库URL，就可以更换数据源。例如，要切换到MySQL数据库：
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/flowable-spring-boot?characterEncoding=UTF-8
spring.datasource.username=flowable
spring.datasource.password=flowable
从Maven依赖中移除H2，并在classpath中添加MySQL驱动：
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.45</version>
</dependency>
多次重启应用，会发现任务的数量增加了（H2内存数据库在关闭后会丢失，而MySQL不会）。

关于配置数据源的更多信息，可以在Spring Boot的参考手册中Configure a DataSource(配置数据源)章节查看。

REST 支持
通常会在嵌入的Flowable引擎之上，使用REST API（用于与公司的不同服务交互）。Spring Boot让这变得很容易。在classpath中添加下列依赖：

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>${spring.boot.version}</version>
</dependency>
创建一个新的Spring服务类，并创建两个方法：一个用于启动流程，另一个用于获得给定任务办理人的任务列表。在这里只是简单地包装了Flowable调用，但在实际使用场景中会比这复杂得多。
@Service
public class MyService {

    @Autowired
    private RuntimeService runtimeService;

    @Autowired
    private TaskService taskService;

    @Transactional
    public void startProcess() {
        runtimeService.startProcessInstanceByKey("oneTaskProcess");
    }

    @Transactional
    public List<Task> getTasks(String assignee) {
        return taskService.createTaskQuery().taskAssignee(assignee).list();
    }

}
现在可以用@RestController来注解类，以创建REST endpoint。在这里我们简单地调用上面定义的服务。
@RestController
public class MyRestController {

    @Autowired
    private MyService myService;

    @RequestMapping(value="/process", method= RequestMethod.POST)
    public void startProcessInstance() {
        myService.startProcess();
    }

    @RequestMapping(value="/tasks", method= RequestMethod.GET, produces=MediaType.APPLICATION_JSON_VALUE)
    public List<TaskRepresentation> getTasks(@RequestParam String assignee) {
        List<Task> tasks = myService.getTasks(assignee);
        List<TaskRepresentation> dtos = new ArrayList<TaskRepresentation>();
        for (Task task : tasks) {
            dtos.add(new TaskRepresentation(task.getId(), task.getName()));
        }
        return dtos;
    }

    static class TaskRepresentation {

        private String id;
        private String name;

        public TaskRepresentation(String id, String name) {
            this.id = id;
            this.name = name;
        }

        public String getId() {
            return id;
        }
        public void setId(String id) {
            this.id = id;
        }
        public String getName() {
            return name;
        }
        public void setName(String name) {
            this.name = name;
        }

    }

}
Spring Boot会自动扫描组件，并找到我们添加在应用类上的@Service与@RestController。再次运行应用，现在可以与REST API交互了。例如使用cURL：
curl http://localhost:8080/tasks?assignee=kermit
[]

curl -X POST  http://localhost:8080/process
curl http://localhost:8080/tasks?assignee=kermit
[{"id":"10004","name":"my task"}]
