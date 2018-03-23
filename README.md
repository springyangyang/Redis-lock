# Redis-lock
Redis实现分布式锁

================================================redis连接==================================================
package com.casic.redis.lock;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

public class RedisManager {
	
	
	private static JedisPool jedisPool;
	
	static{
		JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
		
		jedisPoolConfig.setMaxTotal(2);
		jedisPoolConfig.setMaxIdle(10);
//		new JedisPool(jedisPoolConfig, "redis://root:123456@192.168.163.132:6379/0", 6379);
		jedisPool = new JedisPool(jedisPoolConfig, "192.168.163.132", 6379, 3000, "123456", 0);
	}
	
	//
	public static Jedis getJedis() throws Exception{
		
		if (null!=jedisPool) {
			return jedisPool.getResource();
		}
		throw new Exception("JedisPool was not null");
		
	}
}


===================================获取锁与释放锁实现代码======================================
package com.casic.redis.lock;

import java.util.List;
import java.util.UUID;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.Transaction;

public class DistributeLockDemo {
	
	public String getLock(String key ,int timeOut){
		
		try {
			Jedis jedis = RedisManager.getJedis();
			String value = UUID.randomUUID().toString();
			long end = System.currentTimeMillis()+timeOut;
			long time = System.currentTimeMillis();
			while(time<end){//阻塞
				if (jedis.setnx(key, value)==1) {
					jedis.expire(key, timeOut);
					System.out.println("锁设置成功");
					return value;
				}
				
				//
				if (jedis.ttl(key)==-1) {//检测过期时间
					jedis.expire(key, timeOut);
				}
				Thread.sleep(1000);
			}
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("获取锁失败，返回null");
		return null;
	}
	//释放锁
	public boolean releaseLock(String key,String value){
		
		try {
			Jedis jedis = RedisManager.getJedis();
			while(true){
				
				jedis.watch(key);//开启watch机制
				if (value.equals(jedis.get(key))) {
					Transaction transaction = jedis.multi();
					transaction.del(key);
					List<Object> list = transaction.exec();
					if (list==null) {
						continue;
					}
					System.out.println("释放锁成功");
					return true;
				}
				jedis.unwatch();
				break;
			}
			
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("释放锁失败");
		return false;
	}
	
	
	public static void main(String[] args) {
		
	}
}

==========================================测试获取和释放锁=============================================
package com.casic.redis.lock;

public class TestLock {
	
	public static void main(String[] args) {
		DistributeLockDemo d = new DistributeLockDemo();
		
//		
//		String lock = d.getLock("sgy", 5000);
//		System.out.println(lock);
		//
		 
		boolean b = d.releaseLock("sgy", "1abb5f5a-def6-4c14-a308-c1bda5e7f2ba");
		System.out.println(b);
	}

	
	
}











