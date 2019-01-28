 ## 使用java8的stream-api简化集合操作.md

---  
 ## 为什么需要 Stream

  Stream 作为 Java 8 的一大亮点，它与 java.io 包里的 InputStream 和 OutputStream 是完全不同的概念。它也不同于 SAX 对 XML 解析的 Stream，也不是 Amazon Kinesis 对大数据实时处理的 Stream。Java 8 中的 Stream 是对集合（Collection）对象功能的增强，它专注于对集合对象进行各种非常便利、高效的聚合操作（aggregate operation），或者大批量数据操作 (bulk data operation)。Stream API 借助于同样新出现的 Lambda 表达式,对常规集合操作提供了许多抽象,大大提高了java代码的编程效率和可读性。
---  
### stream-api处理List
```java
// 从一个最简单的例子开始...
// 遍历一个List<Student>,取出student所属的classId
// java8以前的写法
List<Student> students;
List<String> classIdList = new ArrayList<>();
for(Student t: students){
	classIdList.add(t.getClassId());
}
```
```java
//使用stream-api
List<String> classIdList = students.stream()
.map((Student t)->{
	return t.getClassId();
}).collect(Collectors.toList());
```
--- 
```java
//进一步简化书写
List<String> classIdList = students.stream()
.map((Student t)-> t.getClassId())
.collect(Collectors.toList());

//再进一步简化书写
List<String> classIdList = students.stream()
.map(Student::getClassId)
.collect(Collectors.toList());
```
--- 
```java
//假如需要对学生属性做一些过滤
List<String> classIdList = students.stream()
.filter((Student t)->t.getAge() > 12)
.map(Student::getClassId)
.collect(Collectors.toList());

//假如要对List进行转换,按班级(classId)对学生进行分组
Map<String,List<Student>> result = students.stream()
  .collect(Collectors.groupingBy((Student t)->t.getClassId()));
//.collect(Collectors.groupingBy(Student::getClassId);
```
--- 
### stream-api处理Map
```java
Map<String,List<String>> result = new HashMap<>();
//java8进行遍历
result.forEach((String classId,List<Student> students)->{
	//...
})

//旧的写法进行遍历
for(Map.Entry<String,List<String>>  element: result.entrySet()){
	String classId = element.getKey();
	List<Student> students = element.getValue();
	//...
}
```
---
```java
//并且还可以将不同的流操作链接起来
students.stream()
  .filter((Student t)->t.getAge() > 12)
  .collect(Collectors.groupingBy((Student t)->t.getClassId()))
  .forEach((String classId,List<Student> students)->{
	//...
});
```
---
### stream-api进行分组
```java
//假如要在List循环记录多个数组,如何操作
List<Student> highGradeStudents = new ArrayList<>();
List<Student> lowGradeStudents = new ArrayList<>();
List<Student> middleGradeStudent = new ArrayList<>();
for(Student t: students){
    if(t.getAge()>12){
		highGradeStudents.add(t);
    }else if(t.getAge()<9){
		lowGradeStudents.add(t);
    }else{
		middleGradeStudent.add(t);
    }
}
//使用java8 的 groupingBy进行分类
Map<String,List<Student>> res= students.stream()
.collect(
	Collectors.groupingBy((Student t)->{
		if(t.getAge()>12){
		    return "high";
		}else if(t.getAge()<9){
		    return "low";
		}else{
		    return "middle";
		}
	})
);
List<Student> highGradeStudents = res.get("high");
List<Student> lowGradeStudents =  res.get("low");
List<Student> middleGradeStudent = res.get("middle");
```
---
```java
//那么对于这种情况,包含两种维度的遍历,java8要如何实现呢
int countOfClass3_7 = 0;
for(Student t: students){
    if(t.getAge()>12){
		highGradeStudents.add(t);
    }
    if(t.getClassName()=="3年7班"){
		countOfClass3_7++;
    }
}
/*
答案是无法实现;实际上这种代码体现的是一种bad smell,违背了单一职责的要求
除非有明显的证据可以表明这种写法提升了多大的效率,否则对于几十个元素的遍历
展开一次for循环与两次for循环相差不过几毫秒,但是做到分开遍历将会使得代码清晰很多,也有利于维护;
*/
```
---
### 使用并行流加速遍历

```java
for(List<Student> student:classDao.getStudents()){
    //其中doRegisterOne方法执行了注册学生信息,缴费,排课等操作,比较耗时
    registerService.doRegisterOne(student);
}
//引入并行流进行多线程处理
classDao.getStudents().parallelStream()
.forEach((Student student)->{
	boolean res = registerService.doRegisterOne(student);
});
```
---
然而.这种写法其实是有安全隐患的,parallelStream不能指定特定线程池,并且是全局共享,也就是说假如有另一个
xx方法把线程用光了,此处代码的处理速度就会急遽下降,并且还很难排查;
---
```java
// java8实战推荐使用CompletableFuture,并指定线程池
List<CompletableFuture<boolean>> futures = classDao.getStudents().stream()
.map((Student student) ->  
        CompletableFuture.supplyAsync(  
		 //接收一个Supplier<boolean>
		 () -> registerService.doRegisterOne(student),  
		 //指定的线程池
		  StudentExecutor.instance()
		  )
	) 
 .collect(toList());
List<Boolean> resultList = futures.stream()  
  .map(CompletableFuture::join)  
  .filter(Objects::nonNull).collect(toList());
```
---

本文只是对java8做了入门级的介绍,也是对阅读<java8实战>这本书的一个小的总结,更多深入特性推荐阅读原书了解:)


