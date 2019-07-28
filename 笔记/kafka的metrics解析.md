+ 不少文章说kafka使用yammers框架来实现性能监控.这么说其实没有问题,因为kafka确实通过yammers向外暴露了接口,可以通过jmx或者grahite来监视各个性能参数.但是kafka内的性能监控比如producer,consumer的配额限制,并不是通过yammer实现的.而是通过自己的一套metrics框架来实现的.kafka的metrics并不是kafka主要逻辑的一部分。

+ 事实上,kafka有两个metrics包,在看源码的时候很容易混淆package kafka.metrics以及package org.apache.kafka.common.metrics。可以看到这两个包的包名都是metrics,但是他们负责的任务并不相同,而且两个包中的类并没有任何的互相引用关系，是两个完全独立的包。kafka.mtrics这个包,主要调用yammer的Api,并进行封装,提供给client监测kafka的各个性能参数；而org.apache.kafka.common.metrics这个包并不是面向client提供服务的,他是为了给kafka中的其他组件,比如replicaManager,PartitionManager,QuatoManager提供调用,让这些Manager了解kafka现在的运行状况,以便作出相应决策的。

+ 首先metrics第一次被初始化,在kafkaServer的startup()方法中初始化了一个Metrics,并将这个实例传到quotaManagers的构造函数中,这里简单介绍一下quotaManagers.这是kafka中用来限制kafka,producer的传输速度的,比如在config文件下设置producer不能以超过5MB/S的速度传输数据,那么这个限制就是通过quotaManager来实现的。
```scale
    metrics = new Metrics(metricConfig, reporters, kafkaMetricsTime, true)
    quotaManagers = QuotaFactory.instantiate(config, metrics, time)
```

回到metrics上,跟进代码。
```scale
    public class Metrics implements Closeable {
     ....
     ....
        private final ConcurrentMap<MetricName, KafkaMetric> metrics;
        private final ConcurrentMap<String, Sensor> sensors;
    }
```
metrics与sensors这两个concurrentMap是Metrics中两个重要的成员属性.那么什么是KafkaMetric,什么是Sensor呢?
### 首先分析KafkaMetric
KafkaMetric实现了Metric接口,可以看到它的核心方法value()返回要监控的参数的值。
```java
    public interface Metric {
        public MetricName metricName();
        public double value();
    
    }
```
那么KafkaMetric又是如何实现value()方法的呢?
```java
    public double value() {
        synchronized (this.lock) {
            return value(time.milliseconds());
        }
    }
    
    double value(long timeMs) {
        return this.measurable.measure(config, timeMs);
    }
```
原来value()是通过kafkaMetric中的另一个成员属性measurable完成。

```java
public interface Measurable {
    public double measure(MetricConfig config, long now);
}
```    

来看看一个Measrable的简单实现,在sender.java类中。
```java
     metrics.addMetric(m, new Measurable() {
         public double measure(MetricConfig config, long now) {
             return (now - metadata.lastSuccessfulUpdate()) / 1000.0;
         }
     });
```
可以看到measure的实现就是简单地返回要返回的值,因为是直接在目标类中定义的,所以可以直接获得相应变量的引用。
#### Sensor    
```scale
    private final ConcurrentMap<String, Sensor> sensors;
```               

以下是Sensor类的源码。
```java 
    public final class Sensor {
        //一个kafka就只有一个Metrics实例,这个registry就是对这个Metrics的引用
        private final Metrics registry;
        private final String name;
        private final Sensor[] parents;
        private final List<Stat> stats;
        private final List<KafkaMetric> metrics;
    }
``` 
这一段的注释很有意义,从注释中可以看到Sensor的作用不同KafkaMetric. KafkaMetric仅仅是返回某一个参数的值,而Sensor有基于某一参数时间序列进行统计的功能,比如平均值,最大值,最小值.那这些统计又是如何实现的呢?答案是List<Stat> stats这个属性成员。                            

```java 
    public interface Stat {
        public void record(MetricConfig config, double value, long timeMs);
    }
```  
可以看到Stat是一个接口,其中有一个record方法可以记录一个采样数值,下面看一个例子,max这个功能如何用Stat来实现?
```java
    public final class Max extends SampledStat {
    
        public Max() {
            super(Double.NEGATIVE_INFINITY);
        }
    
        @Override
        protected void update(Sample sample, MetricConfig config, double value, long now) {
            sample.value = Math.max(sample.value, value);
        }
    
        @Override
        public double combine(List<Sample> samples, MetricConfig config, long now) {
            double max = Double.NEGATIVE_INFINITY;
            for (int i = 0; i < samples.size(); i++)
                max = Math.max(max, samples.get(i).value);
            return max;
        }
    }
```   
是不是很简单,update相当于冒一次泡,把当前的值与历史的最大值比较.combine相当于用一次完整的冒泡排序找出最大值,需要注意的是,max是继承SampleStat的,而SampleStat是Stat接口的实现类.那我们回到Sensor类上来。
 ```java
    public void record(double value, long timeMs) {
        this.lastRecordTime = timeMs;
        synchronized (this) {
            // increment all the stats
            for (int i = 0; i < this.stats.size(); i++)
                this.stats.get(i).record(config, value, timeMs);
            checkQuotas(timeMs);
        }
        for (int i = 0; i < parents.length; i++)
            parents[i].record(value, timeMs);
    }
```                      
record方法,每个注册于其中的stats提交值,同时如果自己有父sensor的话,向父sensor提交。
 ```java
    public void checkQuotas(long timeMs) {
        for (int i = 0; i < this.metrics.size(); i++) {
            KafkaMetric metric = this.metrics.get(i);
            MetricConfig config = metric.config();
            if (config != null) {
                Quota quota = config.quota();
                if (quota != null) {
                    double value = metric.value(timeMs);
                    if (!quota.acceptable(value)) {
                        throw new QuotaViolationException(
                            metric.metricName(),
                            value,
                            quota.bound());
                    }
                }
            }
        }
    }
``` 
checkQuotas,通过这里其实是遍历注册在sensor上的每一个KafkaMetric来检查他们的值有没有超过config文件中设置的配额.注意这里的QuotaVioLationException,是不是很熟悉.在QuatoManager中,如果有一个client的上传/下载速度超过指定配额.那么就会抛出这个警告。
```java
    try {
      clientSensors.quotaSensor.record(value)
      // trigger the callback immediately if quota is not violated
      callback(0)
    } catch {
      case qve: QuotaViolationException =>
        // Compute the delay
        val clientMetric = metrics.metrics().get(clientRateMetricName(clientQuotaEntity.sanitizedUser, clientQuotaEntity.clientId))
        throttleTimeMs = throttleTime(clientMetric, getQuotaMetricConfig(clientQuotaEntity.quota))
        clientSensors.throttleTimeSensor.record(throttleTimeMs)
        // If delayed, add the element to the delayQueue
        delayQueue.add(new ThrottledResponse(time, throttleTimeMs, callback))
        delayQueueSensor.record()
        logger.debug("Quota violated for sensor (%s). Delay time: (%d)".format(clientSensors.quotaSensor.name(), throttleTimeMs))
    }
``` 
这里就很好理解了,向clientSensor提交上传,下载的值,如果成功了,就掉用相应的callback,如果失败了catch的就是QuotaViolationException.
其实metrics的运行模型还是很简单的,让人感觉绕的就是,各种抽象,Metrics,KafkaMetrics,Sensor,Stat这些概念吧.

最后,Sensor会初始化一个线程专门用来清除长时间没有使用的线程.这个线程名为"SensorExpiryThread"
 ```java
class ExpireSensorTask implements Runnable {
    public void run() {
        for (Map.Entry<String, Sensor> sensorEntry : sensors.entrySet()) {
            synchronized (sensorEntry.getValue()) {
                if (sensorEntry.getValue().hasExpired()) {
                    log.debug("Removing expired sensor {}", sensorEntry.getKey());
                    removeSensor(sensorEntry.getKey());
                }
            }
        }
    }
}
``` 