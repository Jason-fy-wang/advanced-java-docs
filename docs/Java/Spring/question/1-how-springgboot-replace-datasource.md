---
tags:
  - springboot
  - datasource
  - replace
---
在springboot项目中, database的password 每隔两个小时会重新生成一次, 那么对应的项目中的datasource需要重新创建.  

那么如何动态重新创建datasource并把其注入到DAO 层呢?

创建一个 dataSourceWrapper 类, 并实现dataSource接口, 之后把此类出入到 DAO层中, 那么在周期性re-create dataSource时, 每次都更新此Wrapper类中的dataSource.


```java
import javax.sql.DataSource;  
import java.io.PrintWriter;  
import java.sql.Connection;  
import java.sql.SQLException;  
import java.sql.SQLFeatureNotSupportedException;  
import java.util.logging.Logger;  
  
public class DataSourceWrapper implements DataSource {  
  
    private DataSource dataSource;  
  
    public DataSourceWrapper(DataSource dataSource) {  
        this.dataSource = dataSource;  
    }  
    
    public DataSourceWrapper() {    
    }  

  
	public DataSource getDataSource() {  
	    return dataSource;  
	}  
	  
	public void setDataSource(DataSource dataSource) {  
	    this.dataSource = dataSource;  
	}

    @Override  
    public Connection getConnection() throws SQLException {  
        return dataSource.getConnection();  
    }  
  
    @Override  
    public Connection getConnection(String username, String password) throws SQLException {  
        return dataSource.getConnection(username,password);  
    }  
  
    @Override  
    public PrintWriter getLogWriter() throws SQLException {  
        return dataSource.getLogWriter();  
    }  
  
    @Override  
    public void setLogWriter(PrintWriter out) throws SQLException {  
        dataSource.setLogWriter(out);  
    }  
  
    @Override  
    public void setLoginTimeout(int seconds) throws SQLException {  
        this.dataSource.setLoginTimeout(seconds);  
    }  
  
    @Override  
    public int getLoginTimeout() throws SQLException {  
        return dataSource.getLoginTimeout();  
    }  
  
    @Override  
    public Logger getParentLogger() throws SQLFeatureNotSupportedException {  
        return dataSource.getParentLogger();  
    }  
  
    @Override  
    public <T> T unwrap(Class<T> iface) throws SQLException {  
        return dataSource.unwrap(iface);  
    }  
  
    @Override  
    public boolean isWrapperFor(Class<?> iface) throws SQLException {  
        return dataSource.isWrapperFor(iface);  
    }  
}

```


```java
@Configuration  
public class MybatisOrderConfig {  
	@Autowired
	private DataSourceCreator creator;
	
    @Bean(name="orderDataSource")  
    public DataSource dataSource(){  
        return creator.getDataSourceWrapper();  
    }  
}

```


```java
@Component
public class DataSourceCreator {  
    private DataSourceWrapper dataSourceWrapper = new DataSourceWrapper();  
	// to mock schedule create  dataSource
    public void scheduleCreateSource(){  
        ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);  
        executorService.schedule(new Runnable() {  
            @Override  
            public void run() {  
                // Create data source here  
                DataSource dataSource = DataSourceBuilder.create().driverClassName("com.mysql.jdbc.Driver")  
                        .url("jdbc:mysql://127.0.0.1:3306/test_orders?useSSL=false&useUnicode=true&characterEncoding=UTF-8")  
                        .username("root")  
                        .password("admin").build();  
                dataSourceWrapper.setDataSource(dataSource);  
            }  
        }, 0, java.util.concurrent.TimeUnit.SECONDS);  
    }  
  
  
  
    public DataSourceWrapper getDataSourceWrapper() {  
        return dataSourceWrapper;  
    }  
  
    public void setDataSourceWrapper(DataSourceWrapper dataSourceWrapper) {  
        this.dataSourceWrapper = dataSourceWrapper;  
    }  
}

```


