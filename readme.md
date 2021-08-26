# file-sink模块
## 功能
通过TailDir Source读取日志文件，一条日志为一个Flume Event，通过File Roll Sink输出Event的Header和文本内容。

## 调试过程

+ **docker-compose.yml的环境变量FLUME_AGENT_NAME需要配置成flume.conf定义的flume agent名称**

  flume agent name错误，会提示：No configuration found for this host。阅读源码AbstractConfigurationProvider，可以得知就是从flume配置文件中查找Agent名称。

  

+ **docker-compose.yml的环境变量JAVA_OPTS不生效**

  测试Flume Monitor需要增加JAVA_OPTS。调试这个问题花了我1-2个小时，没找到原因，最后解决方法是在flume-env.sh中配置JAVA_OPTS。



+ **不使用Docker，直接在Linux安装Flume调试会更快**

  因为要调试Flume，所以使用Docker之后，还是要熟悉Flume的配置，并且要知道如何通过Docker的API传递这些配置，反而让过程变复杂。



## TailDir Source如何记录文件读取位置

### 基本流程

1. 读取日志文件，将一条条的日志记录封装成Event。
2. 将Event批量写入Channel，写入成功后修改文件读取位置，写入失败则等待一段时间后重试。
3. 定时将文件读取位置写入文件系统。
4. TailDir Source退出时将文件读取位置写入文件系统。



### 源码分析

```java
// 周期性调用process方法，检查文件是否追加日志
class TaildirSource {
    public Status process() {
        for (long inode : existingInodes) {
        TailFile tf = reader.getTailFiles().get(inode);
        tailFileProcess(tf, true);
      }
    }
}

// 关键步骤1和2
class TaildirSource {
	private boolean tailFileProcess(TailFile tf, boolean backoffWithoutNL) {
        while (true) { 
            reader.setCurrentFile(tf);
            // 读取日志文件，将一条条的日志记录封装成Event
            List<Event> events = reader.readEvents(batchSize, backoffWithoutNL);
            try {
                // 将Event批量写入Channel
                getChannelProcessor().processEventBatch(events);
                // 写入成功后修改文件读取位置
                reader.commit();
            } catch (ChannelException ex) {
                // 写入失败则等待一段时间后重试
                TimeUnit.MILLISECONDS.sleep(retryInterval);
            }
            if (condition...) {
                return true or false;
            }
        }
		
    }
}

// reader.commit()细节，写入成功后修改文件读取位置
class ReliableTaildirEventReader {
    public void commit() throws IOException {
        if (!committed && currentFile != null) {
            long pos = currentFile.getLineReadPos();
            currentFile.setPos(pos);
            currentFile.setLastUpdated(updateTime);
            committed = true;
        }
    }
}

// 关键步骤3：定时将文件读取位置写入文件系统
class TaildirSource {
    public synchronized void start() {
        // writePosInitDelay默认为5秒，writePosInterval默认为3秒
        positionWriter.scheduleWithFixedDelay(new PositionWriterRunnable(),
        	writePosInitDelay, writePosInterval, TimeUnit.MILLISECONDS);
    }
}

class PositionWriterRunnable implements Runnable {
    @Override
    public void run() {
        writePosition();
    }
}

private void writePosition() {
    // positionFilePath默认为~/.flume/taildir_position.json
    File file = new File(positionFilePath);
    FileWriter writer = new FileWriter(file);
    // 以Json格式写入文件系统
    String json = toPosInfoJson();
    writer.write(json);
}

// 关键步骤4：TailDir Source退出时将文件读取位置写入文件系统
class TaildirSource {
    @Override
    public synchronized void stop() {
    	writePosition();
    }
}
```



### 结论

定时持久化文件读取位置的机制，在Flume程序异常停止后，存在日志重复投递的问题，下游需要保证幂等写，事务写都不得行。

源文件名+源文件所在主机+文件偏移位置可以作为一条记录得唯一ID。

