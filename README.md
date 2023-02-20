**4.磁盘IO为什么这么慢**

&emsp;&emsp;(1)一次磁盘IO花费的时间

&emsp;&emsp;读写磁头在磁盘扇区上读取或者写入数据，也就是一次一次完整的磁盘IO花费的时间，包括如下三个方面：

&emsp;&emsp;**一.寻道时间**

&emsp;&emsp;指的就是读写磁头移动到正确的半径上所需要的时间，寻道时间越短，磁盘IO操作越快；

&emsp;&emsp;一般磁盘的寻道时间是3-15ms，主流的磁盘寻道时间是5ms；

&emsp;&emsp;**二.旋转延迟时间**

&emsp;&emsp;找到正确的磁道之后，读写磁头移动到正确的位置上所消耗的时间；

&emsp;&emsp;一般取磁盘旋转周期的一半作为旋转延时的近似值：

&emsp;&emsp;7200转/分 -> 120转/秒 -> 每转1/120秒 -> 每转的一半就是1/240秒即4ms；

&emsp;&emsp;**三.数据传输时间**

&emsp;&emsp;指的是将数据从磁盘盘片读取或者写入的时间，一般是1ms以内，可以忽略不计；

&emsp;&emsp;<font color="red">所以主流磁盘的一次磁盘IO的时间为：5ms + 4ms = 9ms；</font>

&#x9;&#x9;

&emsp;&emsp;**(2)一次内存IO花费的时间**

&emsp;&emsp;内存读取一次数据，一般是`100ns`以内，而1ms = 10^6ns = 100万ns；

&emsp;&emsp;所以一次磁盘IO花费的时间是一次内存IO花费时间的约9万倍，几万倍吧；


<img src="https://user-images.githubusercontent.com/8096557/219956063-56dbfb21-a500-49d6-bb3a-8d44a0b00190.png" alt="图片替换文本" width="40%" height="40%" align="bottom" />

```java
@Service
@Slf4j
public class PromotionServiceImpl implements PromotionService {
    //开启促销活动DAO
    @Autowired
    private SalesPromotionDAO salesPromotionDAO;

    //redis缓存工具
    @Resource
    private RedisCache redisCache;
    
    @Resource
    private PromotionConverter promotionConverter;
    
    //新增或修改一个促销活动
    @Transactional(rollbackFor = Exception.class)
    @Override
    public SaveOrUpdatePromotionDTO saveOrUpdatePromotion(SaveOrUpdatePromotionRequest request) {
    	// 判断是否活动是否重复
    	String result = redisCache.get(PROMOTION_CONCURRENCY_KEY +
            request.getName() +
            request.getCreateUser() +
            request.getStartTime().getTime() +
            request.getEndTime().getTime());
    	if (StringUtils.isNotBlank(result)) {
        	return null;
    	}

    	log.info("活动内容：{}", request);
    	// 活动规则
    	String rule = JsonUtil.object2Json(request.getRule());

    	// 构造促销活动实体
    	SalesPromotionDO salesPromotionDO = promotionConverter.convertPromotionDO(request);
    	salesPromotionDO.setRule(rule);

    	// 促销活动落库
    	salesPromotionDAO.saveOrUpdatePromotion(salesPromotionDO);

    	// 写Redis缓存用于下次创建去重
    	redisCache.set(PROMOTION_CONCURRENCY_KEY + request.getName() + request.getCreateUser() + request.getStartTime().getTime() + request.getEndTime().getTime(), UUID.randomUUID().toString(),30 * 60);

    	// 为所有用户推送促销活动，发MQ
    	sendPlatformPromotionMessage(salesPromotionDO);

    	// 构造响应数据
    	SaveOrUpdatePromotionDTO dto = new SaveOrUpdatePromotionDTO();
    	dto.setName(request.getName());
    	dto.setType(request.getType());
    	dto.setRule(rule);
    	dto.setCreateUser(request.getCreateUser());
    	dto.setSuccess(true);
    	return dto;
    }
    ...
}
```
