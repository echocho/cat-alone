# Different Kinds of Testing
This note distinguishes several common testings, how to conduct the tests in general, and when to use them.
1. benchmark testing
2. load testing
3. performance testing
4. concurrency testing
5. scalability testing
6. stress testing


## Benchmark Testing
It's normal part of a development life cycle. Unlike load testing which focuses more on the system performance as a whole, benchmark testing is lower level. For instance, with benchmark testing, we are usually interested in method runtime, database response time, or the number of requests that can be handled within some timeframe, such as 1 second, in typical use cases. Benchmark testing is performed to determine current behavior and performance of a system and is used for a baseline to decide if and to what extent the software/application is enhanced or improved.

### Reference: 
https://www.ibm.com/docs/ko/db2/10.5?topic=methodology-benchmark-testing
https://www.guru99.com/benchmark-testing.html


## Load Testing
It's the process of [putting simulated demand on software, an application or website in a way that tests or demonstrates it's behavior under various conditions](https://loadninja.com/load-testing/#:~:text=Load%20testing%20is%20the%20process,it's%20behavior%20under%20various%20conditions.). Usually, you a clear target or goal for your software or application, and you do load testing to see if the systems match your expectation.

## Performance Testing
This kind of testing is performed to determine how the stability, scalability, speed and responsiveness of an application/software holds up under a give workload. Common metrics are throughput and transaction per second (TPS). Performance testing is and should be iterative. It's common to find people do performance testing only in an afterthought. However, it's highly recommended to provide performance test result at each release, or even better, include performance testing in CI. It's useful in that we can check if engineers make mistake in their code to hamper the software performance, e.g. high time-complexity code, SQL n+1 problem. You'll also find it handy when you need to consider if a product requirement is reasonable, in that if it'll greatly slows down the application.

## Concurrency Testing
[TODO] 
tomcat connection pool max number, what concurrency comes max tsp, concurrency -response time
think about database pool size, tomcat connection pool size, etc
JVM tuning: most of them are to handle full GC - stop the world

## Stress/Torture Testing
We do stress/torture testing to understand how software/ application behaves under extreme level of stress, or pressure. One example is we have requests run for a long time, such as weeks and months and see if anything goes wrong, e.g. OOM. Another example is sending requests in higher concurrency than the max number a system can normally handle and see if and how transactions terminates and rollback.

### Reference
https://www.guru99.com/stress-testing-tutorial.html

## Scalability Testing
[TODO]
1 concurrency, tps 1, good 10 concurrency, tps 10, etc. (if 10, 1, means lock)


## Further Reading
https://octoperf.com/blog/2017/10/19/how-to-analyze-jmeter-results/#installation
