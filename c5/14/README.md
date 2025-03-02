# 5.14 用表间公式调用勤哲的系统表和系统视图的方法
### 声明
本方案在es2010版测试通过,其他版本请自行测试.请提前做好备份!
以下教程均为本人在以上版本的使用中摸索的结果,非官方数据,不保证100%正确性

### 相关系统表说明
1. `ES_datatable`: 所有数据表属性(7个系统表+在模板中创建的表(无论是否在数据库创建)+视图+注册过的外部数据源表): 主键表编号dtid,数据表名dtname/realname,创建时间,更新时间,dsid数据源编号 本数据库为0 内置为null,dsname 数据源名 本数据库为'',state是否实体表 视图或未创建的的表=0 实体表=1,isview 是否是视图,builtin是否系统内置表 如角色表 角色表

2. `ES_datafield`: 即es_datatable表里的字段,所有字段数据简要说明: 字段编号fldid,字段名fldname/realname,类型innertype/datatype,dtid表编号,fldno这个表中的第几个字段 系统表或视图才有 用户表=0, 等

3. `ES_rtts`: 数据表属性(在模板中创建的表(无论是否在数据库创建),不含视图),类似于管理数据表中的该数据表属性:主键rttid,rtid模板编号,sheetid第几个工作表,dtid表编号(all),occurno?,style单一1/明细2,extendable向下扩展,extensible1向右扩展,ifhidden是否隐藏,num?,alias别名--r即real

4. `ES_rtfs`: 即es_rtts表里的字段字典: 主键编号rtfid,模板编号rtid,,fldid字段编号(all),单元格区域datarng,stdid数据规范编号,rtfno?,rttid表编号

5. `ES_SysId`: 系统编号记录:idname编号类型,maxid此类型对应的当前最大号(6为es_datatable.dtid,7为es_datafield.fldid,8为es_rtts.rttid,9为es_rtfs.rtfid)

本教程只涉及1.2.5三个表

![](./5.14.1.png)

### 相关SQL
```sql
/**
 * 此存储过程用于将es系统表或系统视图加入表间公式的选择列表
 * 例如: 运行 exec 显示sys 'es_repcase' 即把es_repcase加入了表间公式数据源列表
 */
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create proc [dbo].[显示sys]
@tbname nvarchar(100) --表名称
as
if OBJECT_ID(@tbname) is null
or (select 1 from ES_DataTable x 
		where x.RealName=@tbname )=1
	begin
	raiserror('此表不存在或已登记过!',16,1)
	return
	end
else

begin
set nocount on
	begin try
	begin tran
		declare @dtid int --此次插入的表编号
				,@fldid int --此次插入的字段编号
				,@n int --字段数量
--新增1个dtid
		exec GetNewId 6,1
		set @dtid =(select maxid from ES_SysId where IdName =6 )
--字段数量n
		set @n=(select count(*) from INFORMATION_SCHEMA.COLUMNS
					where TABLE_NAME =@tbname )
--新增n个fldid
		exec GetNewId 7,@n	
		--设置@fldid为增加前的值	
		set @fldid =(select maxid from ES_SysId 
				where IdName =7 )-@n

--插入一行表记录
		insert into ES_DataTable 
			  (DtId ,DtName ,State,CreDate,RealName ,IsView,IfReadOnly ,IfCanMap ,BuiltIn ,updTime  )
		values(@dtid ,@tbname ,1,GETDATE(),@tbname ,0,0,0,1,GETDATE() )


--插入n行字段记录
		insert into ES_DataField 
		(FldId,DtId,FldName,FldNo ,BaseType  ,InnerType ,isIdentity ,NotNull,RealName,IfPk  )
			select  
			fldid=(@fldid+ROW_NUMBER()over(order by (select 1)) ),
			dtid=@dtid,

			FldName=COLUMN_NAME ,
			FldNo=ROW_NUMBER()over(order by (select 1)) ,
			BaseType=(case when DATA_TYPE like'%char%' then 2
					when DATA_TYPE like'%int%' or DATA_TYPE like'%decimal%' then 1
					else 3 end ),
			InnerType=(case when DATA_TYPE='nvarchar' then 'nvarchar('+cast(CHARACTER_MAXIMUM_LENGTH AS nvarchar)+')'
						else 
						DATA_TYPE 
						end ) ,
			isIdentity=0 ,
			NotNull=0,
			RealName='['+COLUMN_NAME+']' ,
			IfPk=0 

			 from INFORMATION_SCHEMA.COLUMNS
			where TABLE_NAME =@tbname 
	commit tran
	end try

	begin catch
		raiserror('执行失败',16,1)
		if @@TRANCOUNT >0
			begin
			rollback
			end
	end catch
end
/**
 * 若想取消操作,请执行以下存储过程
 * 此存储过程用于将已加入的es系统表或系统视图从数据源列表中清除
 * 例如: 运行 exec 取消sys 'es_repcase' 即把es_repcase从表见公式数据源列表中清除
 */
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create proc [dbo].[取消sys]
@tbname nvarchar(50)
as
if not exists(select 1 from ES_DataTable 
	where RealName=@tbname )
	begin
		raiserror('此表未登记',16,1)
		return
	end
else
begin
	begin try
	begin tran
		set nocount on
		declare @dtid int
		set @dtid =(select dtid from ES_DataTable where RealName=@tbname )
		delete from ES_DataField 
			where DtId =@dtid
		delete from ES_Datatable
			where DtId =@dtid 
	commit tran
	end try
		
	begin catch
		raiserror('执行失败',16,1)
		rollback
	end catch
end

/*若想取消某数据表的隐藏字段的显示,执行此存储过程
如: "exec 取消字段 '出库单_明细'"*/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create proc [dbo].[取消字段]
@tbname nvarchar(50)
as
declare @dtid int--表编号
set @dtid=(select a.dtid from ES_DataTable a 
			where a.DtName =@tbname  )
			
delete from ES_DataField 
	where FldName like 'ExcelServer%'
	and DtId=@dtid

/*本存储过程可让es2010表间公式提数时,提取Excelserverrcid,Excelserverrtid等隐藏字段
如: "exec 显示字段 '出库单_明细'"
应用举例1 表单rcid和es_repcase关联,可取出此表单最后修改日期和修改人
应用举例2 多模板映射到一个数据表,可用rtid区分数据处于哪个模板
应用举例3 通过rn,可判断出该行数据处于表单明细表第几行
*/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create proc [dbo].[显示字段]
@tbname nvarchar(100) --表名称
as
 declare @dtid int --此次涉及的表编号
 set @dtid =(select dtid from ES_DataTable a where a.DtName =@tbname )
 
if @dtid is null
or (select COUNT(*) from ES_Datafield x 
        where x.DtId =@dtid and x.FldName like 'ExcelServer%')>0
    begin
    raiserror('此表不存在或已登记过!',16,1)
    return
    end
else

begin
set nocount on
    begin try
    begin tran
				declare
                @fldid int --此次插入前的最大字段编号
                ,@n int --此次插入的字段数量

--字段数量n
        set @n=(select count(*) from INFORMATION_SCHEMA.COLUMNS
                    where TABLE_NAME =@tbname and COLUMN_NAME like 'ExcelServer%')
		
--新增n个fldid
        exec GetNewId 7,@n    
        --设置@fldid为增加前的值    
        set @fldid =(select maxid from ES_SysId 
                where IdName =7 )-@n

--插入n行字段记录
        insert into ES_DataField 
        (FldId,DtId,FldName,FldNo ,BaseType  ,InnerType ,isIdentity ,NotNull,RealName,IfPk  )
            select  
            fldid=(@fldid+ROW_NUMBER()over(order by (select 1)) ),
            dtid=@dtid,

            FldName=COLUMN_NAME ,
            FldNo=0 ,--因为此次是用户自定义表,so此处=0
            BaseType=(case when DATA_TYPE like'%char%' then 2
                    when DATA_TYPE like'%int%' or DATA_TYPE like'%decimal%' then 1
                    else 3 end ),
            InnerType=(case when DATA_TYPE='nvarchar' then 'nvarchar('+cast(CHARACTER_MAXIMUM_LENGTH AS nvarchar)+')'
                        else 
                        DATA_TYPE 
                        end ) ,
            isIdentity=0 ,
            NotNull=0,
            RealName='['+COLUMN_NAME+']' ,
            IfPk=0 

             from INFORMATION_SCHEMA.COLUMNS
            where TABLE_NAME =@tbname and COLUMN_NAME like 'ExcelServer%'
    commit tran
    end try

    begin catch
        raiserror('执行失败',16,1)
        if @@TRANCOUNT >0
            begin
            rollback
            end
    end catch
end
```

### 本节贡献者
@王达
