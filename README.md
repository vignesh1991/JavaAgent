# JavaAgent
```
package com.snc;

import java.io.IOException;
import java.lang.management.*;
import java.net.SecureCacheResponse;
import java.nio.file.Paths;
import java.util.Optional;
import java.util.Properties;
import java.util.logging.Logger;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import jdk.jfr.consumer.EventStream;
import jdk.jfr.Event;
import jdk.jfr.consumer.RecordedThread;
import jdk.jfr.consumer.*;

import javax.management.MBeanServerConnection;
import javax.management.Notification;
import javax.management.NotificationListener;
import javax.management.remote.JMXConnectionNotification;
import javax.management.remote.JMXConnector;
import javax.management.remote.JMXConnectorFactory;
import javax.management.remote.JMXServiceURL;

import static java.lang.Thread.State.RUNNABLE;

public class StreamExternalEventsWithAttachAPISample {
    static StreamExternalEventsWithAttachAPISample se;

    public static void main(String... args) throws Exception {

        Optional<VirtualMachineDescriptor> vmd =
                VirtualMachine.list().stream()
                        .filter(v -> v.displayName()
                                .equals("com.abc.mainclassName")) // main class name here /
                        .findFirst();

        if (vmd.isEmpty()) {
            throw new RuntimeException("app name here"); // put app name here 
        }

        VirtualMachine vm = VirtualMachine.attach(vmd.get());
        System.out.println(vm);
        System.out.println(vm.getAgentProperties());
        System.out.println(vm.getSystemProperties());
        System.out.println("provider " + vm.provider());

        if (vm != null) {
            se = new StreamExternalEventsWithAttachAPISample();
            String jmxUrl = se.getJmxUrl(vm);
            se.readDataWithJmx(jmxUrl);
        }

    }

    private void getStats(MemoryMXBean memoryMXBean, ThreadMXBean threadMXBean) {
    //   while(memoryMXBean != null) {
            // MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();
//            System.out.println(String.format("Initial memory: %.2f GB",
//                    (double) memoryMXBean.getHeapMemoryUsage().getInit() / 1073741824));
//            System.out.println(String.format("Used heap memory: %.2f GB",
//                    (double) memoryMXBean.getHeapMemoryUsage().getUsed() / 1073741824));
//            System.out.println(String.format("Max heap memory: %.2f GB",
//                    (double) memoryMXBean.getHeapMemoryUsage().getMax() / 1073741824));
//            System.out.println(String.format("Committed memory: %.2f GB",
//                    (double) memoryMXBean.getHeapMemoryUsage().getCommitted() / 1073741824));

            // ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();

            for (final Long threadID : threadMXBean.getAllThreadIds()) {
               ThreadInfo info = threadMXBean.getThreadInfo(threadID);
               while(info.getThreadName().contains("glide.scheduler.worker.1")){
                   System.out.println("Thread name: " + info.getThreadName() + "      " + info.getThreadState());
                 //   MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();
            System.out.println(String.format("Initial memory: %.2f GB",
                    (double) memoryMXBean.getHeapMemoryUsage().getInit() / 1073741824));
            System.out.println(String.format("Used heap memory: %.2f GB",
                    (double) memoryMXBean.getHeapMemoryUsage().getUsed() / 1073741824));
            System.out.println(String.format("Max heap memory: %.2f GB",
                    (double) memoryMXBean.getHeapMemoryUsage().getMax() / 1073741824));
            System.out.println(String.format("Committed memory: %.2f GB",
                    (double) memoryMXBean.getHeapMemoryUsage().getCommitted() / 1073741824));
               }

//                System.out.println("Thread State: " + info.getThreadState());
//                System.out.println(String.format("CPU time: %s ns",
//                        threadMXBean.getThreadCpuTime(threadID)));

            }

    }

    private String readSystemProperty(VirtualMachine virtualMachine, String propertyName) {
        String propertyValue = null;
        try {
            Properties systemProperties = virtualMachine.getSystemProperties();
            propertyValue = systemProperties.getProperty(propertyName);
        } catch (IOException e) {
            // logger.error("Reading system property failed", e);
        }
        return propertyValue;
    }

    private void detachSilently(VirtualMachine virtualMachine) {
        if (virtualMachine != null) {
            try {
                virtualMachine.detach();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void readDataWithJmx(String jmxUrl) {
        JvmMxBean mxBean = new JvmMxBean(jmxUrl);

        try {
            if (mxBean.connect()) {
                RuntimeMXBean runtimeMXBean = mxBean.getRuntimeMXBean();
                OperatingSystemMXBean operatingSystemMXBean = mxBean.getOperatingSystemMXBean();
                MemoryMXBean memoryMXBean = mxBean.getMemoryMXBean();
                ThreadMXBean threadMXBean = mxBean.getThreadMXBean();
                getStats(memoryMXBean, threadMXBean);

            }
        } finally {
            mxBean.disconnect();
        }
    }

    private String getJmxUrl(VirtualMachine virtualMachine) {
        String jmxUrl = readAgentProperty(virtualMachine, "com.sun.management.jmxremote.localConnectorAddress");
        if (jmxUrl == null) {
            loadMangementAgent(virtualMachine);
            jmxUrl = readAgentProperty(virtualMachine, "com.sun.management.jmxremote.localConnectorAddress");
        }

        return jmxUrl;
    }

    private void loadMangementAgent(VirtualMachine virtualMachine) {
        final String id = virtualMachine.id();
        String agent = null;
        Boolean loaded = false;
        try {
            String javaHome = readSystemProperty(virtualMachine, "java.home");
            agent = javaHome + "/lib/management-agent.jar";
            virtualMachine.loadAgent(agent);
            loaded = true;
        } catch (Exception e) {
            System.out.println("Loading agent failed for pid " + e);
        }
        System.out.println("Loading management agent ");
    }

    private String readAgentProperty(VirtualMachine virtualMachine, String propertyName) {
        String propertyValue = null;
        try {
            Properties agentProperties = virtualMachine.getAgentProperties();
            propertyValue = agentProperties.getProperty(propertyName);
        } catch (Exception e) {
            System.out.println("Reading agent property failed" + e);
        }
        return propertyValue;
    }


    class JvmMxBean implements NotificationListener {

        //  private Logger logger = LoggerFactory.getLogger(JvmMxBean.class);

        private final String jmxUrl;

        private MBeanServerConnection mBeanServerConnection;
        private JMXConnector connector;

        private RuntimeMXBean runtimeMXBean;
        private OperatingSystemMXBean operatingSystemMXBean;
        private MemoryMXBean memoryMXBean;
        private ThreadMXBean threadMXBean;

        public JvmMxBean(final String jmxUrl) {
            super();
            this.jmxUrl = jmxUrl;
        }

        public synchronized RuntimeMXBean getRuntimeMXBean() {
            if (mBeanServerConnection != null && runtimeMXBean == null) {
                runtimeMXBean = getMxBean(ManagementFactory.RUNTIME_MXBEAN_NAME, RuntimeMXBean.class);
            }
            return runtimeMXBean;
        }

        public synchronized OperatingSystemMXBean getOperatingSystemMXBean() {
            if (mBeanServerConnection != null && operatingSystemMXBean == null) {
                operatingSystemMXBean = getMxBean(ManagementFactory.OPERATING_SYSTEM_MXBEAN_NAME, OperatingSystemMXBean.class);
            }
            return operatingSystemMXBean;
        }

        public synchronized MemoryMXBean getMemoryMXBean() {
            if (mBeanServerConnection != null && memoryMXBean == null) {
                memoryMXBean = getMxBean(ManagementFactory.MEMORY_MXBEAN_NAME, MemoryMXBean.class);
            }
            return memoryMXBean;

        }

        public synchronized ThreadMXBean getThreadMXBean() {
            if (mBeanServerConnection != null && threadMXBean == null) {
                threadMXBean = getMxBean(ManagementFactory.THREAD_MXBEAN_NAME, ThreadMXBean.class);
            }
            return threadMXBean;

        }

        public boolean connect() {
            try {
                JMXServiceURL url = new JMXServiceURL(jmxUrl);
                connector = JMXConnectorFactory.newJMXConnector(url, null);
                connector.addConnectionNotificationListener(this, null, jmxUrl);
                connector.connect();
                mBeanServerConnection = connector.getMBeanServerConnection();

                return true;
            } catch (Exception e) {
                disconnect();
                throw new IllegalStateException("Couldn't connect to JVM with URL: " + jmxUrl, e);
            }
        }

        public boolean disconnect() {
            boolean result = false;
            try {
                if (connector != null) {
                    connector.removeConnectionNotificationListener(this);
                    connector.close();
                }
                result = true;
            } catch (Exception e) {

                throw new IllegalStateException("Closing the connection failed for JVM", e);
            } finally {
                mBeanServerConnection = null;
                connector = null;
            }
            return result;
        }

        @Override
        public void handleNotification(Notification notification, Object handback) {
            final JMXConnectionNotification noti = (JMXConnectionNotification) notification;
            if (!handback.equals(jmxUrl)) {
                return;
            }

            if (noti.getType().equals(JMXConnectionNotification.CLOSED)) {
                disconnect();
            } else if (noti.getType().equals(JMXConnectionNotification.FAILED)) {
                disconnect();
            } else if (noti.getType().equals(JMXConnectionNotification.NOTIFS_LOST)) {
                disconnect();
            }
        }

        private <MX> MX getMxBean(String mxBeanName, Class<MX> mxBeanInterfaceClass) {
            MX result = null;
            if (mBeanServerConnection != null) {
                try {
                    result = ManagementFactory.newPlatformMXBeanProxy(mBeanServerConnection, mxBeanName, mxBeanInterfaceClass);
                } catch (IOException ioe) {

                }
            }
            return result;
        }
    }

}
```
