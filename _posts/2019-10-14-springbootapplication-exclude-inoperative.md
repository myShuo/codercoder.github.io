---
id: 801
title: 'SpringBootApplication启动排除DataSourceAutoConfiguration不生效???'
date: '2019-10-14T22:11:42+08:00'
author: Smile
layout: post
guid: 'http://codercoder.cn/?p=801'
permalink: /index.php/2019/10/springbootapplication-exclude-inoperative/
views:
    - '11158'
    - '11158'
    - '11158'
categories:
    - Tech
    - Fell-in-pit
tags:
    - Spring
    - Tech
    - Fell-in-pit
---

 项目引用了新版本mybatis-spring-boot-starter之后启动不起来，报错Cannot determine embedded database driver class for database type NONE，在网上搜索是需要在排除掉spring自身的org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration这个类就可以，不让其自动配置。  
 由于项目是采用spring boot框架，所以在@SpringBootApplication中exclude这个类即可：  
改之前代码：

```java
@EnableAutoConfiguration
@SpringBootApplication
@EnableDubbo(multipleConfig = true)
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).profiles("default").build(args).run(args);
    }
}

```

添加DataSourceAutoConfiguration排除之后：

```java
@EnableAutoConfiguration
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
@EnableDubbo(multipleConfig = true)
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).profiles("default").build(args).run(args);
    }
}

```

 本以为这样就成功了，改之后发现重新启动项目还是报一样的错误，就比较奇怪，网上都是这样解决的，自己这里却不行。  
 现在只能猜测是不是我上面这样配置没生效，根本没有排除掉DataSourceAutoConfiguration呢，然后去debug了下spring这块源码，看spring是如何找到启动注解上要排除的类的。  
 最终找到了这个类AnnotatedElementUtils#searchWithGetSemanticsInAnnotations方法，该方法是查找该类上某个注解的定义配置的。

```java
/**
 * This method is invoked by {@link #searchWithGetSemantics} to perform
 * the actual search within the supplied list of annotations.
 * <p>This method should be invoked first with locally declared annotations
 * and then subsequently with inherited annotations, thereby allowing
 * local annotations to take precedence over inherited annotations.
 * <p>The {@code metaDepth} parameter is explained in the
 * {@link Processor#process process()} method of the {@link Processor} API.
 * @param element the element that is annotated with the supplied
 * annotations, used for contextual logging; may be {@code null} if unknown
 * @param annotations the annotations to search in
 * @param annotationType the annotation type to find
 * @param annotationName the fully qualified class name of the annotation
 * type to find (as an alternative to {@code annotationType})
 * @param containerType the type of the container that holds repeatable
 * annotations, or {@code null} if the annotation is not repeatable
 * @param processor the processor to delegate to
 * @param visited the set of annotated elements that have already been visited
 * @param metaDepth the meta-depth of the annotation
 * @return the result of the processor (potentially {@code null})
 * @since 4.2
 */
    private static <T> T searchWithGetSemanticsInAnnotations(AnnotatedElement element,
            List<Annotation> annotations, Class<? extends Annotation> annotationType,
            String annotationName, Class<? extends Annotation> containerType,
            Processor<T> processor, Set<AnnotatedElement> visited, int metaDepth) {

        // Search in annotations
        // 这个for循环是在类上直接定义的注解上进行查找指定的注解
        for (Annotation annotation : annotations) {
            Class<? extends Annotation> currentAnnotationType = annotation.annotationType();
            if (!AnnotationUtils.isInJavaLangAnnotationPackage(currentAnnotationType)) {
                if (currentAnnotationType == annotationType ||
                        currentAnnotationType.getName().equals(annotationName) ||
                        processor.alwaysProcesses()) {
                    T result = processor.process(element, annotation, metaDepth);
                    if (result != null) {
                        if (processor.aggregates() && metaDepth == 0) {
                            processor.getAggregatedResults().add(result);
                        }
                        else {
                            return result;
                        }
                    }
                }
                // Repeatable annotations in container?
                else if (currentAnnotationType == containerType) {
                    for (Annotation contained : getRawAnnotationsFromContainer(element, annotation)) {
                        T result = processor.process(element, contained, metaDepth);
                        if (result != null) {
                            // No need to post-process since repeatable annotations within a
                            // container cannot be composed annotations.
                            processor.getAggregatedResults().add(result);
                        }
                    }
                }
            }
        }

        // Recursively search in meta-annotations
        // 如果根据类上直接定义的注解去找不到话，然后在遍历每一个注解，找寻其通过继承关系得到的注解
        for (Annotation annotation : annotations) {
            Class<? extends Annotation> currentAnnotationType = annotation.annotationType();
            if (!AnnotationUtils.isInJavaLangAnnotationPackage(currentAnnotationType)) {
                T result = searchWithGetSemantics(currentAnnotationType, annotationType,
                        annotationName, containerType, processor, visited, metaDepth + 1);
                if (result != null) {
                    processor.postProcess(element, annotation, result);
                    if (processor.aggregates() && metaDepth == 0) {
                        processor.getAggregatedResults().add(result);
                    }
                    else {
                        return result;
                    }
                }
            }
        }

        return null;
    }

```

 拿我上面项目例子来说明，上面的代码我配置了@EnableAutoConfiguration和@SpringBootApplication两个注解，@SpringBootApplication里的exclude实际上使用的是@EnableAutoConfiguration里的exclude，所以当spring查找EnableAutoConfiguration这个注解的配置时候，根据上面spring代码可以知道，如果本地配置了，会优先取本地配置的，如果本地没有配置，才会通过第二个for循环，也就是从@SpringBootApplication里面去取，然后我项目代码也配置了@EnableAutoConfiguration，所以优先取本地配置，即在@SpringBootApplication配置的不会生效。  
 知道了这个结论之后，项目正常启动，改动就很简单了：

```java
@EnableAutoConfiguration(exclude = {DataSourceAutoConfiguration.class})
@SpringBootApplication
@EnableDubbo(multipleConfig = true)
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).profiles("default").build(args).run(args);
    }
}

```

或者：**（推荐使用这种方式）**

```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
@EnableDubbo(multipleConfig = true)
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).profiles("default").build(args).run(args);
    }
}

```

***总结***  
– 通过查看上面spring代码方法注释也能得到结论，**这个方法应该首先调用本地声明的注释然后用继承的注释，从而允许本地注解优先于继承的注解。**  
– @SpringBootApplication是多个注解的组合，其中就已经包含了@EnableAutoConfiguration注解，所以引用了@SpringBootApplication注解之后不需要在手动注解@EnableAutoConfiguration