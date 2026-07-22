List câu hỏi phỏng vấn

1. Your "@Transactional" method is executing, but the transaction isn't rolling back. Why?

2. An API is suddenly taking 8 seconds in production, but only 300ms locally. Where do you start debugging?

3. "@Async" is enabled, but the method still executes synchronously. What could be wrong?

4. Your application starts creating duplicate records even though you're checking if they already exist. How would you fix it?

5. A scheduled job runs only once locally but executes multiple times after deploying multiple instances. Why?

6. HikariCP starts throwing "Connection is not available" errors during peak traffic. What would you investigate?

7. Your REST API returns 200 OK, but the database update never happens. How is that possible?

8. Cache improves response time, but users start seeing outdated data. How would you solve it?

9. One slow external API causes your entire application to become slow. Which resilience patterns would you apply?

10. Everything works in staging, but the application fails immediately after deploying to production. What are the first things you would verify?



Core Java & OOPS Concepts
1. What is Polymorphism? Explain with examples.
2. What is Encapsulation?
3. What is Method Overloading?
4. What is the use of the `final` keyword?
5. How do you create an immutable class in Java?
6. Why is String immutable in Java?
7. What is Synchronization in Java?

🔹 Collections Deep Dive
8. Can we add/modify elements in a HashMap while iterating over it?
9. What is Fail-Fast behavior in HashMap? What exception is thrown if we modify a HashMap during iteration?
10. Explain the internal working of HashMap.
11. What is the equals() and hashCode() contract in Java?

🔹 Java 21 & Modern Java
12. What are the new features introduced in Java 21?
13. Coding: Create a Record class in Java 21.
14. What is a @FunctionalInterface?
15. Coding: Write a Lambda expression to find the sum of digits of a number.
16. What are the new features introduced in Java 17?
17. Coding: Create a Record class in Java 17.

🔹 Exception Handling
18. What are the different types of Exceptions in Java?
19. Coding: Create a custom Exception class.
20. Coding: Write a program to demonstrate Exception Handling (try-catch-finally with a custom scenario).

🔹 Miscellaneous Core Java
21. What is the use of the `super` keyword?
22. What is the difference between Comparator and Comparable?
23. What are the methods present in the Object class?

🔹 Spring Framework
24. What are the commonly used Spring annotations?
25. What is the use of the @Query annotation in Spring Data JPA?
26. How do you connect two different databases in a single Spring Boot application?

🔹 Coding/Logical Question
27. Given the number 122345, print the sum of digits that are even. Like we have to extract all the even numbers and sum those numbers.


Tại sao phải override cả equals() và hashCode()?
HashMap xử lý collision như thế nào? Khi nào chuyển từ LinkedList sang Red-Black Tree?
HashMap khác ConcurrentHashMap ở điểm nào?
ArrayList vs LinkedList: khi nào dùng mỗi loại?
volatile khác synchronized như thế nào?
wait() khác sleep() như thế nào?
Callable vs Runnable.
Thread Pool là gì? Tại sao không nên tạo thread bằng new Thread() liên tục?
Stream API có lazy evaluation là gì?
Optional dùng khi nào, có nên dùng làm field trong Entity không?