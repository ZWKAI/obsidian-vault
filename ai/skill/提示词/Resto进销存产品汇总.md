### init
你一名开发人员，具备专业测财务会计知识，现在要开发一个合并数据的skill，先帮我出一个编写计划(存放到doc下)，有什么疑问可以给我提出，现在的信息如下：
功能描述：依据源表，根据规则和映射关系，在摸版的基础上生成汇总表
一、技术要求：
	1. skill名称：fin-resto-stock-summary
	2. 中文名称：Resto进销存产品汇总
	3. 根据把资源里的摸版文件，生成新的摸版放入skill中，保留标题样式
	4. 根据摸版生成excel文件，放到源文件目录下即可
	5. 复制数据时样式参考摸版row2


二、业务要求
	1. 忽略源表前四个sheet(HQ Finance Records、Finance Report、SG Procurement List、Dynamic Stock List),复制之后的sheet数据到摸版sheet
	2. 映射关系
	采购发票号：Invoice
	厂商名称：sheet名称
	序列号：Serial Number
	入库数量：固定值1
	入库时间：Stock-In Date
	库存地点：Stock By
	型号名称：Name
	小计：同行Invoice值相同的数量汇总
	
	销售发票号：Invoice No
	客户名称：Merchant Outlet
	发货小计：同行Invoice值相同且Stock Out不为空的数量汇总
	出库时间：Stock Out
	备注：Remarks


三、资源
1. 源文件地址：/Users/wenkai/ai办公自动化/财务/核算组/合并财务报表-上市/开发/Resto进销存产品汇总/_2026 SG Procuement Management.xlsx
2. 摸版：/Users/wenkai/ai办公自动化/财务/核算组/合并财务报表-上市/开发/Resto进销存产品汇总/2026年Resto进销存共享表-2026.6.22 v2.xlsx


### 生成用户使用文档
现在生成一份给用户可操作的使用文档，放到doc文件夹下面，用户是不懂技术的财务会计人员，使用智能体是openclaw。