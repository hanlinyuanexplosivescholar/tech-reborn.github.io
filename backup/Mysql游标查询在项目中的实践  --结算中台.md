业务场景：
        需求是我们系统需要通过定时任务，将某些Mysql及MongoDB数据表的全量数据读取到文件中，并将文件推送到AWS S3中。
        由于数据量比较大，量级大约在数百万，需要分批读取数据写入文件。如果通过分页方式读取，会出现深度分页，查询速度越来越慢，导致任务执行时间过长。影响下游数据使用，时效性太差。
        为解决时效问题，在合理时间范围内处理任务，考虑通过游标方式读取数据。并将文件分段处理，通过并发上传方式将文件段推送到AWS S3中


Mysql游标查询业务代码: 

```
@Mapper
public interface SrRebateScheduleHistoryMapper extends BaseMapper<SrRebateScheduleHistory> {


    @Select("SELECT * FROM sr_rebate_schedule_history ORDER BY id DESC")
    @Options(resultSetType = ResultSetType.FORWARD_ONLY, fetchSize = 1000)
    @ResultType(SrRebateScheduleHistory.class)
    void getRebateScheduleHistory(@Param(Constants.WRAPPER) QueryWrapper<SrRebateScheduleHistory> wrapper, ResultHandler handler);
}
```


什么是MySQL游标？

在MySQL中，游标是一个数据库对象，用于在查询结果集上执行逐行或逐批的数据操作。游标允许我们遍历查询结果，并以一种有序的方式访问每一行数据。通常，游标用于存储过程和函数中，但也可以在SQL语句中使用。


MySQL游标的主要作用包括：

逐行或逐批处理数据： 游标允许我们在查询结果集上逐行或逐批执行数据处理操作。这对于需要对每一行数据进行特定处理的场景非常有用，如数据转换、数据清洗、复杂计算等。
浏览大型结果集： 在处理大型查询结果时，不必一次性将所有数据加载到内存中，可以使用游标来逐个获取和处理数据，从而节省内存资源。
控制数据访问： 游标允许我们在结果集中前进、后退、跳过特定行等，以灵活地控制数据的访问方式。



MySQL游标的适用场景：

数据转换和清洗： 当需要对查询结果进行复杂的数据转换、清洗或归档操作时，游标可以逐行处理数据并执行必要的转换操作。
报表生成： 生成复杂的报表通常需要从数据库中检索大量数据并对其进行处理。游标可用于逐行处理数据以生成报表。
数据分析： 在数据分析任务中，游标可用于按行执行统计或分析操作，以获取更精确的结果。
大数据集处理： 处理大型查询结果集时，游标允许按需加载和处理数据，而不会占用大量内存。
     
        