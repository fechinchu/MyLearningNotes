# SpringBoot自动配置

![image-20200803163850719](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200803163850719.png)

1. `@SpringBootContiguration`注解是一个组合注解,内部包含了`@Configuration`,`@Configuration`注解实际上就是`@Component`;
2. `@EnableAutoConfiguration`注解也是一个组合注解,内部包含了`@AutoConfigurationPackage`和`@Import(AutoConfigurationImportSelector.class)`,其中`@Import(AutoConfigurationImportSelector.class)`有方法如下:

![image-20200803170411160](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200803170411160.png)

![image-20200803170433865](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200803170433865.png)

``@AutoConfigurationPackage`中有`@Import(AutoConfigurationPackages.Registrar.class)`,

![image-20200803171525447](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200803171525447.png)

3. `@ComponentScan`

