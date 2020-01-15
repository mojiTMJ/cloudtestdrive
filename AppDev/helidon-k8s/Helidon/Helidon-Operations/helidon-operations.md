[Go to Helidon for Cloud Native Page](../Helidon-labs.md)

![](../../../../common/images/customer.logo2.png)

# Migration of Monolith to Cloud Native

## A. Helidon for Cloud Native

## 4. Helidon and operations

One thing that many developers forget is that once they have finished writing code that it still has to run and be maintained. With the introduction of DevOps a lot of developers suddenly found they were the ones being woken up in the middle of the night to fix problems in their code. That changes the perception a bit and not many developers are acutely aware that they will have ongoing involvement in the code well after the time it compiles cleanly and passed the text suite.

To help maintain and operate systems after they have been released a lot of information is needed, especially in situations where a bug may be on one service, but not show up until the resulting data has passed through several other microservcies. 

Equally performance information is key to understanding how well the services are running, and in the event of a performance problem which specific microservice it it that's got the problem!

Fortunately for us and other developers Helidon has support for tools and and producing data that will help diagnose problems, and determine if there is a problem in the first place.

### Tracing
We now managed to achieve the situation where we have a set of microservices that cooperate to perform specific function. However we don't know exactly how they are operating in reality, we do of course know how they operate in terms of our design!

Tracing in a microservices environment allows us to see the flow of a request across all of the microservices involved, not just the sequence of method calls in a particular service. 

Helidon has built-in support for tracing. There are a few actions that we need to take to activate this.

Firstly we need to deploy a tracing engine, Helidon supports several tracing engines, but for this lab we will use the Zipkin engine. For now we will use docker to run Zipkin. In the Kubeneres labs we will see how we can run Zipkin in Kubernetes.

In the VM you have docker installed and running, so to start zipkin just open a terminal and run the following command to start Zipkin in a container

```
$ docker run -d -p 9411:9411 --name zipkin --rm openzipkin/zipkin 
Starting zipkin docker image in detached mode
d12b253c50b7793ca8e3eb64658efead336fa3880d3df040f12152b57347f067
```

You won't see any more output from this, just the container id, but if you go to http://localhost:9411/zipkin/ you will see the front end and can confirm it's running.

![zipkin-initial](images/zipkin-initial.png)

You normally would need to add the zipkin packages into the pom.xml file, but that's already been done for you. Helidon automatically recognizes the presence of the zipkin files in the it's runtime environment

Update the conf/storefront-config.yaml file and conf/stockmanager-conf.yaml files in both projects, uncomment the tracing lines to specify the relevant project name as the service and the host as "zipkin" (there is actually a default value already set in the microprofile-config.properties file, but we're going to override that here) 

If they are still running stop any existing storefront and stockmanager instances, then restart them.

Make a request, say reserving stock

```
$ curl -i -X POST -u jill:password -d '{"requestedItem":"Pencil", "requestedCount":7}' -H "Content-type:application/json" http://localhost:8080/store/reserveStock
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 6 Jan 2020 15:46:29 GMT
connection: keep-alive
content-length: 37

{"itemCount":143,"itemName":"Pencil"}
```

We've successfully reserved 7 pencils


Now go to the zipkin web page and click find traces, you'll see the list of traces

![zipkin-traces-list](images/zipkin-traces-list.png)

Here you can see that is took 236.638ms doign storefront related parts of the request and only 102.309ms doing the stock manager parts of the request.

If we click on the overall time (`239.468ms 27 spans` in this case, though your numbers will vary) zipkin will display the full details of our trace.

![zipkin-trace-details](images/zipkin-trace-details.png)

Importantly even though they are in separate microservices and the flow switches between them several times we can see the overall flow, what part of the service was called when and how long it took. This let's developers understand exactly how the initial request was processed and how long each step took.

I restarted both the storefront and stockmanager service, so the lazy initialization around the REST infrastructure and the database had not yet happened, then made the same request. This took a lot longer as the framework had to do it's on-demand initialization.

![zipkin-trace-details-slow-response](images/zipkin-trace-details-slow-response.png)

As you can see it took a long time for the stockmanager to perform it's initial action (it felt longer than the 3.496s it actually took !) but subsequent requests to the stockmanager were much faster. A developer may look at this trace and decide that it would be a good thing if the stockmanager took an action during it's initialization to setup the database connection then which woudl speed the first request up for users.

### Metrics
Tracking solutions like Zipkin can provide us with detail on how a single request is processed, but they are not going to be able to tell us how many requests were made, and what the distribution of requests per second is. This is the kind of thing that is needed by the operations team to understand how the microservice is being used, and where enhancements may be a good idea (especially where to focus development work for performance enhancements)

The pom.xml will need to be updated for the metrics, that's already been done for you here.

Open the com.oracle.labs/helidon.storefront.resources.StorefrontResources class. Add the `@Counted` annotation to it.

```
@Path("/store")
@RequestScoped
@Counted(monotonic = true)
@Authenticated
@Timeout(value = 15, unit = ChronoUnit.SECONDS)
@Log
@NoArgsConstructor
public class StorefrontResource {
```

The `monotonic=true` means that the counters will increment when the method is called, but will not decrement when it's exited. If you wanted to have a particular method report how many threads were currently in it (perhaps to determine when resource limits may be reached) you'd use `@Counted(monotonic=false)` which would decrement the counter when a thread left the method giving the number of threads in a method.

A note on names, the default name for a counter is based on the class and method, that can be long (as we'll see in the output soon) but you can override the default name and for counters on *methods* you can specify a name for the counter E.g. `@Counted(names="MyListCounter")` This makes the output easier to understand, but do please chose a sensible name. You can also specify a description of the counter and help text if you like (examples of system generated versions are below)

That's it, you don't need to do anything else, Helidon will automatically generate a set of counters for all of the requests it processes.

Restart the storefront service.

look at http://localhost:9080/metrics see that there are metrics in the list

```
$ curl -i -X GET http://localhost:9080/metrics
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
Date: Mon, 6 Jan 2020 16:43:11 GMT
connection: keep-alive
content-length: 22309

# TYPE base:classloader_current_loaded_class_count counter
# HELP base:classloader_current_loaded_class_count Displays the number of classes that are currently loaded in the Java virtual machine.
base:classloader_current_loaded_class_count 8181
# TYPE base:classloader_total_loaded_class_count counter
# HELP base:classloader_total_loaded_class_count Displays the total number of classes that have been loaded since the Java virtual machine has started execution.
base:classloader_total_loaded_class_count 8181
# TYPE base:classloader_total_unloaded_class_count counter
# HELP base:classloader_total_unloaded_class_count Displays the total number of classes unloaded since the Java virtual machine has started execution.
base:classloader_total_unloaded_class_count 0
# TYPE base:cpu_available_processors gauge
# HELP base:cpu_available_processors Displays the number of processors available to the Java virtual machine. This value may change during a particular invocation of the virtual machine.
base:cpu_available_processors 16
# TYPE base:cpu_system_load_average gauge
# HELP base:cpu_system_load_average Displays the system load average for the last minute. The system load average is the sum of the number of runnable entities queued to the available processors and the number of runnable entities running on the available processors averaged over a period of time. The way in which the load average is calculated is operating system specific but is typically a damped timedependent average. If the load average is not available, a negative value is displayed. This attribute is designed to provide a hint about the system load and may be queried frequently. The load average may be unavailable on some platforms where it is expensive to implement this method.
base:cpu_system_load_average 2.3701171875
# TYPE base:gc_g1_old_generation_count gauge
# HELP base:gc_g1_old_generation_count Displays the total number of collections that have occurred. This attribute lists -1 if the collection count is undefined for this collector.
base:gc_g1_old_generation_count 0
# TYPE base:gc_g1_old_generation_time_seconds gauge
# HELP base:gc_g1_old_generation_time_seconds Displays the approximate accumulated collection elapsed time in milliseconds. This attribute displays -1 if the collection elapsed time is undefined for this collector. The Java virtual machine implementation may use a high resolution timer to measure the elapsed time. This attribute may display the same value even if the collection count has been incremented if the collection elapsed time is very short.
base:gc_g1_old_generation_time_seconds 0.0
# TYPE base:gc_g1_young_generation_count gauge
# HELP base:gc_g1_young_generation_count Displays the total number of collections that have occurred. This attribute lists -1 if the collection count is undefined for this collector.
base:gc_g1_young_generation_count 4
# TYPE base:gc_g1_young_generation_time_seconds gauge
# HELP base:gc_g1_young_generation_time_seconds Displays the approximate accumulated collection elapsed time in milliseconds. This attribute displays -1 if the collection elapsed time is undefined for this collector. The Java virtual machine implementation may use a high resolution timer to measure the elapsed time. This attribute may display the same value even if the collection count has been incremented if the collection elapsed time is very short.
base:gc_g1_young_generation_time_seconds 0.029
# TYPE base:jvm_uptime_seconds gauge
# HELP base:jvm_uptime_seconds Displays the start time of the Java virtual machine in milliseconds. This attribute displays the approximate time when the Java virtual machine started.
base:jvm_uptime_seconds 15.329
# TYPE base:memory_committed_heap_bytes gauge
# HELP base:memory_committed_heap_bytes Displays the amount of memory in bytes that is committed for the Java virtual machine to use. This amount of memory is guaranteed for the Java virtual machine to use.
base:memory_committed_heap_bytes 1073741824
# TYPE base:memory_max_heap_bytes gauge
# HELP base:memory_max_heap_bytes Displays the maximum amount of heap memory in bytes that can be used for memory management. This attribute displays -1 if the maximum heap memory size is undefined. This amount of memory is not guaranteed to be available for memory management if it is greater than the amount of committed memory. The Java virtual machine may fail to allocate memory even if the amount of used memory does not exceed this maximum size.
base:memory_max_heap_bytes 17179869184
# TYPE base:memory_used_heap_bytes gauge
# HELP base:memory_used_heap_bytes Displays the amount of used heap memory in bytes.
base:memory_used_heap_bytes 109051904
# TYPE base:thread_count counter
# HELP base:thread_count Displays the current number of live threads including both daemon and nondaemon threads
base:thread_count 63
# TYPE base:thread_daemon_count counter
# HELP base:thread_daemon_count Displays the current number of live daemon threads.
base:thread_daemon_count 59
# TYPE base:thread_max_count counter
# HELP base:thread_max_count Displays the peak live thread count since the Java virtual machine started or peak was reset. This includes daemon and non-daemon threads.
base:thread_max_count 63
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource 0
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item 0
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock 0
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_failed_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_failed_total The number of times the method was called and, after all Fault Tolerance actions had been processed, threw a Throwable
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_failed_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_total The number of times the method was called
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_calls_not_timed_out_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_calls_not_timed_out_total The number of times the method completed without timing out
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_calls_not_timed_out_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_calls_timed_out_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_calls_timed_out_total The number of times the method timed out
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_calls_timed_out_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_mean_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_mean_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_max_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_max_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_min_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_min_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_stddev_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_stddev_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds summary
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds Histogram of execution times for the method
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds_count 0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds{quantile="0.5"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds{quantile="0.75"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds{quantile="0.95"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds{quantile="0.98"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds{quantile="0.99"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_timeout_execution_duration_seconds{quantile="0.999"} 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_fallback_calls_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_fallback_calls_total Number of times the fallback handler or method was called
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_fallback_calls_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_invocations_failed_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_invocations_failed_total The number of times the method was called and, after all Fault Tolerance actions had been processed, threw a Throwable
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_invocations_failed_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_invocations_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_invocations_total The number of times the method was called
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_invocations_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_calls_not_timed_out_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_calls_not_timed_out_total The number of times the method completed without timing out
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_calls_not_timed_out_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_calls_timed_out_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_calls_timed_out_total The number of times the method timed out
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_calls_timed_out_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_mean_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_mean_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_max_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_max_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_min_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_min_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_stddev_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_stddev_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds summary
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds Histogram of execution times for the method
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds_count 0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds{quantile="0.5"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds{quantile="0.75"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds{quantile="0.95"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds{quantile="0.98"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds{quantile="0.99"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock_timeout_execution_duration_seconds{quantile="0.999"} 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_fallback_calls_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_fallback_calls_total Number of times the fallback handler or method was called
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_fallback_calls_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_invocations_failed_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_invocations_failed_total The number of times the method was called and, after all Fault Tolerance actions had been processed, threw a Throwable
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_invocations_failed_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_invocations_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_invocations_total The number of times the method was called
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_invocations_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_calls_not_timed_out_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_calls_not_timed_out_total The number of times the method completed without timing out
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_calls_not_timed_out_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_calls_timed_out_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_calls_timed_out_total The number of times the method timed out
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_calls_timed_out_total 0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_mean_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_mean_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_max_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_max_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_min_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_min_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_stddev_seconds gauge
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_stddev_seconds 0.0
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds summary
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds Histogram of execution times for the method
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds_count 0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds{quantile="0.5"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds{quantile="0.75"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds{quantile="0.95"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds{quantile="0.98"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds{quantile="0.99"} 0.0
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item_timeout_execution_duration_seconds{quantile="0.999"} 0.0
# TYPE vendor:grpc_requests_count counter
# HELP vendor:grpc_requests_count Each gRPC request (regardless of the method) will increase this counter
vendor:grpc_requests_count 0
# TYPE vendor:grpc_requests_meter_total counter
# HELP vendor:grpc_requests_meter_total Each gRPC request will mark the meter to see overall throughput
vendor:grpc_requests_meter_total 0
# TYPE vendor:grpc_requests_meter_rate_per_second gauge
vendor:grpc_requests_meter_rate_per_second 0.0
# TYPE vendor:grpc_requests_meter_one_min_rate_per_second gauge
vendor:grpc_requests_meter_one_min_rate_per_second 0.0
# TYPE vendor:grpc_requests_meter_five_min_rate_per_second gauge
vendor:grpc_requests_meter_five_min_rate_per_second 0.0
# TYPE vendor:grpc_requests_meter_fifteen_min_rate_per_second gauge
vendor:grpc_requests_meter_fifteen_min_rate_per_second 0.0
# TYPE vendor:requests_count counter
# HELP vendor:requests_count Each request (regardless of HTTP method) will increase this counter
vendor:requests_count 0
# TYPE vendor:requests_meter_total counter
# HELP vendor:requests_meter_total Each request will mark the meter to see overall throughput
vendor:requests_meter_total 0
# TYPE vendor:requests_meter_rate_per_second gauge
vendor:requests_meter_rate_per_second 0.0
# TYPE vendor:requests_meter_one_min_rate_per_second gauge
vendor:requests_meter_one_min_rate_per_second 0.0
# TYPE vendor:requests_meter_five_min_rate_per_second gauge
vendor:requests_meter_five_min_rate_per_second 0.0
# TYPE vendor:requests_meter_fifteen_min_rate_per_second gauge
vendor:requests_meter_fifteen_min_rate_per_second 0.0
```

It'a **lot** of data, but it's broken up into sections.

The `base` data e.g. 

```
# TYPE base:classloader_current_loaded_class_count counter
# HELP base:classloader_current_loaded_class_count Displays the number of classes that are currently loaded in the Java virtual machine.
base:classloader_current_loaded_class_count 8181
```
is core data about the operation of the JVM, this is the kind of data you care about if you want to do things like customize the garbage collector (though I recommend never doign that as nowadays it's highly adaptive and will do a way better job at figuring out how it should behave than you'll ever do)

The requests under the `vendor` section relate to the Helidon frameworks operation

```
# TYPE vendor:requests_count counter
# HELP vendor:requests_count Each request (regardless of HTTP method) will increase this counter
vendor:requests_count 0
```

The data under the `application`` section relate to counters applied on the application code itself.

```
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock 0
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item 0
```

These ones tell us how often the ListAllStock and reserveStockItem methods have been called. 

Lastly you'll see there are quite a lot that start `application:ft_`

```
# TYPE application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_total counter
# HELP application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_total The number of times the method was called
application:ft_com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item_invocations_total 0
```

These are generated automatically because we've enabled fault tolerance, the ft_ counters keep track of how many dined the fallback has been called, if the fallback returned useful data or itself generated an exception and so on.

As we only just restarted the storefront it's not a surprise that these are all zero.


### Limiting the output

If you like you can limit the scope of the returned metrics by specifying the scope in the request

```
$ curl -i -X GET http://localhost:9080/metrics/application
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
Date: Mon, 6 Jan 2020 17:07:17 GMT
connection: keep-alive
content-length: 15576

# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource 5

<deleted text>
```

Of if you're only interested in a specific metric you can just retrieve that, provided it's been named.

### With real counter data

Let's make a couple of list stock requests, then look at the list_all_stock counter

```
$ curl -i -X GET -u jill:password http://localhost:8080/store/stocklevel
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 6 Jan 2020 16:58:21 GMT
connection: keep-alive
content-length: 148

[{"itemCount":5000,"itemName":"pin"},{"itemCount":136,"itemName":"Pencil"},{"itemCount":50,"itemName":"Eraser"},{"itemCount":100,"itemName":"Book"}]
```
(I did the curl above 5 times)

Now let's look at the metrics (I removed a bunch of unneeded output here to focus on the counters)

```
$ curl -i -X GET http://localhost:9080/metrics
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
Date: Mon, 6 Jan 2020 16:59:03 GMT
connection: keep-alive
content-length: 22467

# TYPE base:classloader_current_loaded_class_count counter
# HELP base:classloader_current_loaded_class_count Displays the number of classes that are currently loaded in the Java virtual machine.
base:classloader_current_loaded_class_count 9941

<deleted output>

# TYPE base:thread_max_count counter
# HELP base:thread_max_count Displays the peak live thread count since the Java virtual machine started or peak was reset. This includes daemon and non-daemon threads.
base:thread_max_count 83
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_storefront_resource 5
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_failed_list_stock_item 0
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_list_all_stock 5
# TYPE application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item counter
# HELP application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item 
application:com_oracle_labs_helidon_storefront_resources_storefront_resource_reserve_stock_item 0

<deleted output>

```

We can see that now 5 requests in total have been made to the storefront resource, and 5 requests to the listAllStock method, the others have had none. If we were lookgin for a place to optimize things then perhaps we might like to consider looking at that method first !


Why port 9080 ? Well you may recall that in the helidon core lab we defined the network as having two ports, one for the main application on port 8080 and another for admin functions on port 9080, we then specified that metrics (and health which we'll see later) were in the admin category so they are on the admin port. It's useful to splis these things so we don't risk the core function of the microservice getting mixed up with operation data.  

### Other types of metrics
There are other types of metrics, for examples times. in ther StorefrontResource class add a timer annotation `@Timed(name = "listAllStockTimer")` and a meter `@Metered(name = "listAllStockMeter", absolute = true)`  to the listAllStock method. The absolute=true on the meter means that the class name won't be prepended, it will just be called listAllStockMeter the timer as it doesn;t have absoluteTrue will have the class name added as a perfix.

```
	@GET
	@Path("/stocklevel")
	@Produces(MediaType.APPLICATION_JSON)
	@Fallback(fallbackMethod = "failedListStockItem")
	@Timed(name = "listAllStockTimer")
	@Metered(name = "listAllStockMeter", absolute = true)
	public Collection<ItemDetails> listAllStock() {
```

Now restart the storefront and make a few calls

```
$ curl -i -X GET -u jill:password http://localhost:8080/store/stocklevel
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 6 Jan 2020 17:19:49 GMT
connection: keep-alive
content-length: 148

[{"itemCount":5000,"itemName":"pin"},{"itemCount":136,"itemName":"Pencil"},{"itemCount":50,"itemName":"Eraser"},{"itemCount":100,"itemName":"Book"}]
```

(I made the curl request 5 time again)

Now let's get the details specific to our named meter by specifying it in the metrics data request

```
$ curl -i -X GET http://localhost:9080/metrics/application/listAllStockMeter
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
Date: Mon, 6 Jan 2020 17:20:41 GMT
connection: keep-alive
content-length: 726

# TYPE application:list_all_stock_meter_total counter
# HELP application:list_all_stock_meter_total 
application:list_all_stock_meter_total 5
# TYPE application:list_all_stock_meter_rate_per_second gauge
application:list_all_stock_meter_rate_per_second 0.08133631423819475
# TYPE application:list_all_stock_meter_one_min_rate_per_second gauge
application:list_all_stock_meter_one_min_rate_per_second 0.03716438621959536
# TYPE application:list_all_stock_meter_five_min_rate_per_second gauge
application:list_all_stock_meter_five_min_rate_per_second 0.014179223683357264
# TYPE application:list_all_stock_meter_fifteen_min_rate_per_second gauge
application:list_all_stock_meter_fifteen_min_rate_per_second 0.005264116322948982
```

### Combining counters, metrics, times and so on
You can have multiple annotations on your class / methods, but be careful that you don't get naming coliccions, if you do your program will likely fail to start.

By default any of `@Metric`, `@Timed`, `@Counted` etc. will use a name that's depending on the class / method name, it does **not** append the type of thing it's looking for. So if you had `@Counted` on the class and `@Timed` a class (or `@Counted` and `@Timed` on a particular method) then there would be a naming clash between the two of them. It's best to get into the habit of naming these, and putting the type in the name. Then you also get the additional benefit of being able to easily extract it using the metrics url like `http://localhost:9080/metrics/application/listAllStockMeter`




### End of the lab
You have finished this part of the lab, you can proceed to the next step of this lab:


[5. The Helidon support for Cloud Native Operations lab](../Helidon-cloud-native/helidon-cloud-native.md)





---


[Go to *Helidon for Cloud Native* overview Page](../Helidon-labs.md)