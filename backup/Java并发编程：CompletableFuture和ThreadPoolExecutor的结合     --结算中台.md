
业务背景：
        读取数据库多表数据，且部分单表数据量较大，串行任务执行时间较长。为加快同步速度，避免影响下游大数据
团队使用数据生成报表，决定采用并发方式推送。

组合使用的好处：
       1、通过池化技术避免线程膨胀
       2、可以对不同任务指定不同的线程池，防止业务互相影响。尤其是存在依赖场景的任务。
       3、通过任务组合确保在所有任务执行完成后，才发送通知到xxljob admin，同步任务结束状态




```
@Component
@Slf4j
public class PushAwsTask {


    @Autowired
    private AmazonS3Service amazonS3Service;

    @XxlJob("PushRebateDataToAws")
    public void pushRebateDataToAws() throws Exception{
        //设置推送的文件类型
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentType("txt");
        //获取执行线程池
        ThreadPoolExecutor threadPoolExecutor = ThreadPoolUtil.getThreadPool();
        CompletableFuture<Void> rebateScheduleHistoryFuture = CompletableFuture.runAsync(() -> {
            try {
                log.info("推送RebateScheduleHistory信息开始");
                this.amazonS3Service.pushRebateScheduleHistory(metadata);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }, threadPoolExecutor);

        CompletableFuture<Void> rebateAccrualFuture = CompletableFuture.runAsync(() -> {
            try {
                log.info("推送rebateAccrual信息开始");
                this.amazonS3Service.pushRebateAccrual(metadata);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }, threadPoolExecutor);

        CompletableFuture<Void> rebateAgreementFuture = CompletableFuture.runAsync(() -> {
            try {
                log.info("推送rebateAgreement信息开始");
                this.amazonS3Service.pushRebateAgreement(metadata);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }, threadPoolExecutor);

        CompletableFuture<Void> completableFuture = CompletableFuture.allOf(rebateScheduleHistoryFuture,rebateAccrualFuture,rebateAgreementFuture);
        completableFuture.get();
        log.info("-------------------------------------------");
        log.info("--Export Success---------------------pushRebateDataToAws");
        log.info("-------------------------------------------");
    }

}
```