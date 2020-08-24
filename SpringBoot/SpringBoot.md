```java
@SpringBootApplication
	--- @SpringBootConfiguration
        --- @Configuration
        	--- @Component
    --- @EnableAutoConfiguration
        --- @AutoConfigurationPackage
        	--- @Import(AutoConfigurationPackages.Registrar.class)
        --- @Import(AutoConfigurationImportSelector.class)
        	
    @ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
            @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
```



