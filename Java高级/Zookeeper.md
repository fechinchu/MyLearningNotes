# Zookeeper

# 1.Zookeeper简介

简介:Apache ZooKeeper是一种用于分布式应用程序的高性能协调服务,提供一种集中式信息存储服务;

特点:数据存在内存中,类型文件系统的树形结构(文件和目录),高吞吐量和低延迟,集群高可靠;

作用:基于Zookeeper可以实现分布式统一配置中心,服务注册中心,分布式锁等功能;

![image-20200609163047861](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200609163047861.png)

 # 2.CLI操作指令

![image-20200613111655594](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613111655594.png)

# 3.Zookeeper的核心概念

## 3.1.Session

![image-20200613163123024](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613163123024.png)

## 3.2.数据模型

* 层次名称空间

  * 类似unix文件系统,以`/`为根;
  * 区别:节点可以包含与之关联的数据以及子节点(既是文件也是文件夹);
  * 节点的路径总是表示为规范的,绝对的,斜杠分隔的路径;

* znode

  * 名称唯一,命名规范;
  * 节点类型:持久,顺序,临时,临时顺序;
  * 节点数据构成;

  ![image-20200613163545497](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613163545497.png)

* znode节点类型

  * 持久节点 `create /app1 xxx`;
  * 临时节点 `create -e /app2 xxx`;
  * 顺序节点
    * `create -s /app1/cp xxx`:名称为`cp0000000000`;
    * `create -s /app1/ aa`:名称为`0000000001`;
  * 临时顺序节点 `create -e -s /app1/ xxx`;

* znode数据构成

  * 节点数据:存储的协调数据(状态信息,配置,位置信息等);
  * 节点元数据:stat结构;

  ![image-20200613170416517](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613170416517.png)

  * 数据量上限:1M;

## 3.3.Zookeeper中的时间

![image-20200613173905433](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613173905433.png)

 ## 3.4.Watch监听机制

客户端可以在znodes上设置watch,监听znode的变化

![image-20200613174355562](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613174355562.png)

* data watch 监听数据变更;
* child watch 监听子节点变化;

**watch重要特性**:

* 一次性触发:watch触发后即被删除,要持续监控变化,则需要设置watch;
* 有序性:客户端先得到watch通知,后才会看到变化结果;

**watch注意事项:**

* watch是一次性触发器,如果你获得了一个watch事件,并且希望得到关于未来更改的通知,则必须设置另一个watch;
* 因为watch是一次性触发器,并且在获取事件和发送获取watch的请求之间存在延迟,所以不能可靠地得到节点发生的每个更改;
* 一个watch对象只会被特定的通知触发一次.如果一个watch对象同时注册了exists,getData,当节点被删除时,删除事件对exists,getData都有效,但只会调用watch一次;

## 3.5.ZooKeeper特性

1. 顺序一致性:保证客户端操作都是按顺序生效的;
2. 原子性:更新成功或失败,没有部分结果;
3. 单个系统印象:无论连接到哪个服务器,客户端都将看到相同的内容;
4. 可靠性:数据的变更不会丢失,除非被客户端覆盖修改;
5. 及时性:保证系统的客户端当时读取到的数据是最新的;

# 4.ZooKeeper典型应用场景

* 数据发布订阅(配置中心);
* 命名服务;
* Master选举;
* 集群管理;
* 分布式队列;
* 分布式锁;

## 4.1.ZooKeeper实现配置中心

为什么ZooKeeper能够实现配置中心

* znode能够存储数据;
* watch能够监听数据改变;

方案:

* 一个配置项一个znode;
* 一个配置文件一个znode;

~~~java
public class ConfigCenterDemo {

	// 1 将单个配置放到zookeeper上
	public void put2Zk() {
		ZkClient client = new ZkClient("localhost:2181");
		client.setZkSerializer(new MyZkSerializer());
		String configPath = "/config1";
		String value = "1111111";
		if (client.exists(configPath)) {
			client.writeData(configPath, value);
		} else {
			client.createPersistent(configPath, value);
		}
		client.close();
	}

	// 2 将配置文件的内容存放到zk节点上
	public void putConfigFile2ZK() throws IOException {

		File f = new File(this.getClass().getResource("/config.xml").getFile());
		FileInputStream fin = new FileInputStream(f);
		byte[] datas = new byte[(int) f.length()];
		fin.read(datas);
		fin.close();

		ZkClient client = new ZkClient("localhost:2181");
		client.setZkSerializer(new BytesPushThroughSerializer());
		String configPath = "/config2";
		if (client.exists(configPath)) {
			client.writeData(configPath, datas);
		} else {
			client.createPersistent(configPath, datas);
		}
		client.close();
	}

	// 3 需要配置的服务都从zk上取，并注册watch来实时获得配置更新
	public void getConfigFromZk() {
		ZkClient client = new ZkClient("localhost:2181");
		client.setZkSerializer(new MyZkSerializer());
		String configPath = "/config1";
		String value = client.readData(configPath);
		System.out.println("从zk读到配置config1的值为：" + value);
		// 监控配置的更新
		client.subscribeDataChanges(configPath, new IZkDataListener() {

			@Override
			public void handleDataDeleted(String dataPath) throws Exception {
				// TODO Auto-generated method stub

			}

			@Override
			public void handleDataChange(String dataPath, Object data) throws Exception {
				System.out.println("获得更新的配置值：" + data);
			}
		});

		// 这里只是为演示实时获取到配置值更新而加的等待。实际项目应用中根据具体场景写（可用阻塞方式）
		try {
			Thread.sleep(5 * 60 * 1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	public static void main(String[] args) throws IOException {
		ConfigCenterDemo demo = new ConfigCenterDemo();
		demo.put2Zk();
		demo.putConfigFile2ZK();

		demo.getConfigFromZk();
	}

}

~~~

## 4.2.ZooKeeper实现命名服务

![image-20200613190054046](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613190054046.png)

## 4.3.ZooKeeper实现Master选举

![image-20200613192403659](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613192403659.png)

## 4.4.ZooKeeper实现分布式队列

![image-20200613193417854](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613193417854.png)

# 5.ZooKeeper快速总结

## 5.1.Zookeeper的分布式协调场景

![image-20200613194057610](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200613194057610.png)

## 5.2.Zookeeper的分布式锁场景

![image-20200614064448200](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200614064448200.png)

## 5.3.Zookeeper的配置管理场景

![image-20200614094845560](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200614094845560.png)

## 5.4.Zookeeper的HA高可用性场景

![image-20200614101239415](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200614101239415.png)