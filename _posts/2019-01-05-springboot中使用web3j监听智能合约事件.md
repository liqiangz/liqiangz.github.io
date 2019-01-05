智能合约执行后有时需要在后台监听相应的事件。本文将介绍下如何在springboot中使用web3j库监听智能合约的事件
### 导入web3j的包
在 pom.xml 文件中添加如下依赖
```xml
<dependency>
            <groupId>org.web3j</groupId>
            <artifactId>core</artifactId>
            <version>3.3.1</version>
</dependency>
```   
### 将web3j对象放入spring容器中管理

新建 `ContractConfig.java` 文件，代码注释比较详细，参考注释
注意不能是单例模式，还有合约地址的格式也要注意。
```java
package io.liqiangz.config;

import io.liqiangz.contract.Trace;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;
import org.web3j.protocol.Web3j;
import org.web3j.protocol.admin.Admin;
import org.web3j.protocol.core.DefaultBlockParameter;
import org.web3j.protocol.core.DefaultBlockParameterName;
import org.web3j.protocol.core.Request;
import org.web3j.protocol.core.methods.request;
import org.web3j.protocol.core.methods.request.EthFilter;
import org.web3j.protocol.core.methods.response.AnmoBlockNumber;
import org.web3j.protocol.core.methods.response.EthBlockNumber;
import org.web3j.protocol.core.methods.response.Web3ClientVersion;
import org.web3j.protocol.http.HttpService;
import org.web3j.tx.ClientTransactionManager;
import org.web3j.tx.ManagedTransaction;
import org.web3j.tx.TransactionManager;

import java.io.IOException;
import java.math.BigInteger;

/**
 * 智能合約
 */
@Configuration
public class ContractConfig {

    //智能合约部署地址
    private String contractAddress = "0xaf0895260ef377ceea0086c99e3eff7999f742c9";

    @Bean
    @Scope("prototype")
    public Web3j web3j() {
        return Web3j.build(new HttpService("http://192.168.0.104:15450"));
    }

    @Bean
    @Autowired
    public Trace trace(Web3j web3j) throws IOException {
        Trace contract;
        try {
            Web3ClientVersion web3ClientVersion = web3j.web3ClientVersion().send();
            String clientVersion = web3ClientVersion.getWeb3ClientVersion();
            System.out.println("clientVersion" + clientVersion);
            // 以某个用户的身份调用合约
            TransactionManager transactionManager = new ClientTransactionManager(web3j, "0x24602722816b6cad0e143ce9fabf31f6026ec622"); 
            //加载智能合约
            contract = Trace.load(contractAddress, web3j, transactionManager, ManagedTransaction.GAS_PRICE, org.web3j.tx.Contract.GAS_LIMIT);
            return contract;
        } catch (Exception e) {
            e.printStackTrace();
            throw e;
            //throw new RRException("连接智能合约异常");
        }
    }

    @Bean
    //监听这里才用每次都生成一个新的对象，因为同时监听多个事件不能使用同一个实例
    @Scope("prototype")
    @Autowired
    public EthFilter ethFilter(Trace trace, Web3j  web3j) throws IOException {
        //获取启动时监听的区块
        Request<?, EthBlockNumber> request = web3j.ethBlockNumber();
        BigInteger fromblock = request.send().getBlockNumber();
        return new EthFilter(DefaultBlockParameter.valueOf(fromblock),
                //如果监听不到这里的地址可以把 0x 给去掉
                DefaultBlockParameterName.LATEST, trace.getContractAddress());
    }

}
```

### 在spring启动时启动监听

启动监听，注意监听一个就要注入一个监听器，不能共用一个监听器。

```java
/**
 * 服务监听器，继承ApplicationRunner，在spring启动时启动
 * @author liqiang
 */
@Component
public class ServiceRunner implements ApplicationRunner {
    /**
     * 日志记录
     */
    private Logger log = LoggerFactory.getLogger(ServiceRunner.class);
    

    @Autowired
    private Web3j web3j;

    //如果多个监听，必须要注入新的过滤器
    @Autowired
    private EthFilter uploadProAuth;

    @Override
    public void run(ApplicationArguments var1) throws Exception{
        uploadProAuth();
        this.log.info("This will be execute when the project was started!");
    }


    /**
     * 收到上链事件
     */
    public void uploadProAuth(){
        Event event = new Event("uploadProAuthEvent",
                Arrays.<TypeReference<?>>asList(new TypeReference<Address>() {}),
                Arrays.<TypeReference<?>>asList(new TypeReference<Bytes32>() {},
                        new TypeReference<Uint32>() {}));

        uploadProAuth.addSingleTopic(EventEncoder.encode(event));
        log.info("启动监听uploadProAuthEvent");
        web3j.ethLogObservable(uploadProAuth).subscribe(log -> {
            this.log.info("收到事件uploadProAuthEvent");
            EventValues eventValues = staticExtractEventParameters(event, log);
            Trace.UploadProAuthEventEventResponse typedResponse = new Trace.UploadProAuthEventEventResponse();
            typedResponse._fromaddr = (String) eventValues.getIndexedValues().get(0).getValue();
            typedResponse._product_id = (byte[]) eventValues.getNonIndexedValues().get(0).getValue();
            typedResponse._rand_number =  (BigInteger)eventValues.getNonIndexedValues().get(1).getValue();
        });

    }
}
```
