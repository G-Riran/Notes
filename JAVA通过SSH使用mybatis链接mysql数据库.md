工作中遇到一个问题：在本地需要链接阿里云环境的数据库，因为阿里云是预生产环境，与本地网络隔离导致不能直链，只能使用SSH的方式建立通道。在网上找了几篇大都是讲java如何建立ssh通道的，或者使用的是JDBC的方式连接数据库，一直不知道如果使用mybatis是否也可以，最近测试了一次，发现完全是可以的。现将整个的过程简单记录下。整个代码主要涉及两个类：MybatisUtil.java（通过mybatis连接mysql数据库）、SshChannel.java（SSH通道建立类）

代码如下：

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
 
import java.io.IOException;
import java.io.Reader;
 
/**
 * @author jinga
 * @date 2019/05/14 10:14
 * Description: 连接数据库
 */
 
public class MybatisUtil {
    private static ThreadLocal<SqlSession> threadLocal = new ThreadLocal<SqlSession>();
    private static ThreadLocal<SSHChannel> channelThreadLocal = new ThreadLocal<SSHChannel>();
    private static String PRO_ENV= "ali-staging";
 
    /**
     * 禁止外界通过new方法创建
     */
    private MybatisUtil() {
    }
 
    public static void main(String[] args) throws Exception {
        SqlSession sqlSession = getSqlsession("ali-staging");
        //执行sql语句，测试是否连接成功
        System.out.println(sqlSession.selectList("getConfigByConfig"));
        closeSqlSession();
    }
 
    /**
     * 获取SqlSession
     */
    public static SqlSession getSqlsession(String envId) throws IOException {
 
        /**
         * 加载MybatisConfig.xml配置文件
         * 配置文件中的连接数据库的地址需要改成“127.0.0.1”，端口需要改成“本地映射端口”
         */
        try {
            Reader reader = Resources.getResourceAsReader("MybatisConfig.xml");
            SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, envId);
 
            if (PRO_ENV.equals(envId)){
                SSHChannel sshChannel = channelThreadLocal.get();
                if (sshChannel == null) {
                    sshChannel = new SSHChannel();
                    //建立一个通道，将本地端口通过跳板机映射到远程服务器的指定端口
                    sshChannel.goSSH(本地映射端口, "跳板机IP",
                            跳板机端口, "跳板机用户名", "跳板机密码(如果没有写 null )",
                            "阿里云数据库IP", 阿里云数据库端口);
                    channelThreadLocal.set(sshChannel);
                }
            }
 
            SqlSession sqlSession = threadLocal.get();
            if (sqlSession == null) {
                sqlSession = factory.openSession();
                threadLocal.set(sqlSession);
            }
            return sqlSession;
 
        } catch (IOException e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        } finally {
 
        }
 
    }
 
    /**
     * 关闭SqlSession和sshChannel与当前线程分开
     */
    public static void closeSqlSession() {
        SqlSession sqlSession = threadLocal.get();
        if (sqlSession != null) {
            sqlSession.close();
            threadLocal.remove();
        }
        SSHChannel sshChannel = channelThreadLocal.get();
        if (sshChannel != null) {
            sshChannel.close();
            channelThreadLocal.remove();
        }
 
    }
}

——————————————————————————————————————————————————————————————————————————————————

import com.jcraft.jsch.Channel;
import com.jcraft.jsch.JSch;
import com.jcraft.jsch.Session;
 
public class SSHChannel {
    private Session session;
    private Channel channel;
    private String sshPrvKey = "~/.ssh/id_rsa";//SSH证书存放的目录
 
    /**
     *
     * @param localPort  本地host 建议mysql 3306 redis 6379
     * @param sshHost   ssh host
     * @param sshPort   ssh port
     * @param sshUserName   ssh 用户名
     * @param sshPassWord   ssh密码
     * @param remotoHost   远程机器地址
     * @param remotoPort	远程机器端口
     */
    public void goSSH(int localPort, String sshHost, int sshPort,
                      String sshUserName, String sshPassWord,
                      String remotoHost, int remotoPort) {
        try {
            JSch jsch = new JSch();
            //如果使用证书登入ssh的方式，如果使用密码可以不需要这行
            jsch.addIdentity(sshPrvKey);
            //登陆跳板机
            session = jsch.getSession(sshUserName, sshHost, sshPort);
            session.setPassword(sshPassWord);
            session.setConfig("StrictHostKeyChecking", "no");
            session.connect();
            //建立通道
            channel = session.openChannel("session");
            channel.connect();
            //通过ssh连接到mysql机器
            int assinged_port = session.setPortForwardingL(localPort, remotoHost, remotoPort);
            System.out.println("通道建立成功，映射端口号："+assinged_port);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
 
    /**
     * 关闭
     */
    public void close() {
        if (session != null && session.isConnected() ) {
            session.disconnect();
        }
 
        if (channel != null && session.isConnected() ) {
            channel.disconnect();
        }
    }
}