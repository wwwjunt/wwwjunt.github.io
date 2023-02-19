你好

<img src="https://user-images.githubusercontent.com/8096557/219956063-56dbfb21-a500-49d6-bb3a-8d44a0b00190.png" alt="图片替换文本" width="500" height="313" align="bottom" />

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
