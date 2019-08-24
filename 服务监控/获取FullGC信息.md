# 获取Full GC信息

## 背景
Java API 并没有提供Full GC的方法，因为标准的虚拟机规范并不包含Full GC，并且垃圾回收的机制不一样，比如G1和CMS差别就很大。

## 如何查看 Full GC信息
标准做法就是通过jstat命令来查看，但是jstat是JDK tools.jar提供的，并且在Linux下，如果没有足够的权限，是无法查看的，通过在程序里调用jstat命令来解析Full GC信息，是行不通的。

可以想到的一个方法是通过GC日志来获取GC信息，需要在JVM启动参数里添加-Xloggc:配置项。

## GC日志的误区
我们查看gc.log文件，通常理解为包含Full GC关键字的为Full GC信息，其他的为Minor GC信息，实际上gc.log远远不止这些信息，比这个负载很多，而且不同的回收算法，打印的日志也不一样，不同的虚拟机日志也不一样。

**日志里没有包含Full GC关键词，不代表没有发生Full GC。**

## GC 日志里哪些是Full GC?
* 明确包含了Full GC关键字的
* 回收日志包含了多个分代信息，而不仅仅是一个分代信息的

## 如何方便的获取Full GC信息
GC View 提供了GC日志解析功能，非常强大，也可以直接通过它的API获取，支持多种虚拟机和回收算法

下载地址：https://github.com/chewiebug/GCViewer

## Full GC信息获取演示代码
```
package com.tagtraum.perf.gcviewer;

import java.io.File;
import java.io.FileFilter;
import java.lang.management.ManagementFactory;
import java.lang.ref.SoftReference;
import java.lang.ref.WeakReference;
import java.time.ZonedDateTime;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

import com.tagtraum.perf.gcviewer.imp.DataReaderException;
import com.tagtraum.perf.gcviewer.imp.DataReaderFacade;
import com.tagtraum.perf.gcviewer.model.AbstractGCEvent;
import com.tagtraum.perf.gcviewer.model.GCEvent;
import com.tagtraum.perf.gcviewer.model.GCModel;
import com.tagtraum.perf.gcviewer.model.GCResource;
import com.tagtraum.perf.gcviewer.model.GcResourceFile;

public class GCTest {

    public GCTest() {
        int[] arrays = new int[1000000];
    }

    public static void main(String[] args) {
        File logFile = getLogFile();
        if (logFile != null) {
            Thread thread = new Thread() {
                @Override
                public void run() {
                    while (true) {
                        ZonedDateTime now = ZonedDateTime.now();
                        ZonedDateTime max = now;
                        try {
                            Thread.sleep(5000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        File log = getLogFile();
                        if (log == null) {
                            continue;
                        }
                        GCResource gcResource = new GcResourceFile(log);
                        try {
                            DataReaderFacade dataReaderFacade = new DataReaderFacade();
                            GCModel model = dataReaderFacade.loadModel(gcResource);
                            int time = 0;
                            double pausedTime = 0;

                            Iterator<GCEvent> fullGCEvents = model.getFullGCEvents();
                            while (fullGCEvents.hasNext()) {
                                GCEvent event = fullGCEvents.next();
                                if (event.getDatestamp().isAfter(now)) {
                                    time++;
                                    pausedTime += event.getPause();
                                }
                                if (event.getDatestamp().isAfter(max)) {
                                    max = event.getDatestamp();
                                }
                            }
                            System.out.println("FullGC time = " + time);
                            System.out.println("FullGC paused time = " + String.format("%.3f", pausedTime));

                            time = 0;
                            pausedTime = 0;
                            Iterator<GCEvent> minorGCEvents = model.getGCEvents();
                            while (minorGCEvents.hasNext()) {
                                GCEvent event = minorGCEvents.next();
                                if (event.getDatestamp().isAfter(now)) {
                                    time++;
                                    pausedTime += event.getPause();
                                }
                                if (event.getDatestamp().isAfter(max)) {
                                    max = event.getDatestamp();
                                }
                            }
                            System.out.println("MinorGC time = " + time);
                            System.out.println("MinorGC paused time = " + String.format("%.3f", pausedTime));

                            time = 0;
                            pausedTime = 0;
                            Iterator<AbstractGCEvent<?>> vmEvents = model.getVmOperationsEvents();
                            while (vmEvents.hasNext()) {
                                AbstractGCEvent<?> event = vmEvents.next();
                                if (event.getDatestamp().isAfter(now)) {
                                    time++;
                                    pausedTime += event.getPause();
                                }
                                if (event.getDatestamp().isAfter(max)) {
                                    max = event.getDatestamp();
                                }
                            }
                            System.out.println("VM operation overhead time = " + time);
                            System.out.println("VM operation overhead time = " + String.format("%.3f", pausedTime));

                            now = max;

                        } catch (DataReaderException e) {
                            e.printStackTrace();
                        }
                    }
                }
            };
            thread.start();
        } else {
            System.err.println("无法获取GC日志，请在JVM启动参数添加-Xloggc:配置项");
            System.exit(1);
        }
        List items = new ArrayList();
        while (true) {
            items.add(new WeakReference(new int[1000000]));
            items.add(new SoftReference(new int[1000000]));
        }
    }

    private static File getLogFile() {
        List<String> inputArgs = ManagementFactory.getRuntimeMXBean().getInputArguments();
        File logFile = null;
        for (int i = 0; i < inputArgs.size(); i++) {
            String arg = inputArgs.get(i);
            if (arg.startsWith("-Xloggc:")) {
                final String file = arg.replace("-Xloggc:", "").trim();
                logFile = new File(file);
                final String configFileName = logFile.getName();
                if (!logFile.exists()) {
                    File[] files = logFile.getParentFile().listFiles(new FileFilter() {

                        @Override
                        public boolean accept(File temp) {
                            String fileName = temp.getName();
                            if (fileName.startsWith(configFileName) && fileName.endsWith("current")) {
                                return true;
                            }
                            return false;
                        }
                    });
                    if (files != null && files.length > 0) {
                        logFile = files[0];
                    }
                }
            }
        }
        return logFile;
    }

}
```

