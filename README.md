# JK
商贸系统，SSM+Oracle

#数据库设计
1）	表是如何来
从《需求规格说明书》分析客户的业务，抽取出名称作为数据库表，抽取名称中修饰动作的作为关联关系。表的字段，字段名称中文名称，英文名称，大多从用户术语中来。字段类型，字符串VARCHAR2、整形INT、浮点型DOUBLE、日期型TIMESTAMP。长短。很清晰它的精度，设置固定；需求中比较模糊的，适当冗余。都要基于数据库设计的三范式来设计。

对数据库表的部分，用户使用频度高，并发压力大，数据量非常大对这样的表进行优化。
2）	打断设计
在实际业务中业务关联结构非常复杂，这样如果数据库设计表时也按照业务的这种关联关系，会导致当数据量随着用户对系统的使用，数据不断增多。而逐渐导致业务查询的速度越来越慢，到达一定临街时，查询效率极具降低。这时为了提高查询速度，将它们之间的关系硬性打断。这时不采用以往的外键关联，而采取在主表中增加一个字段，来存储和子表的一对多关系。实际存储子表的多个ID值，用特殊符号分割。这样设计后，再查询业务时，直接利用这个字段，利用SQL的in子查询直接查找所需要的数据。例如：购销合同和报运关系。如果按传统关联设计方式，一个报运要获得合同的下的货物信息，必须先找到多个对应的购销合同，在从多个购销合同找到多个对应的货物信息。按我们的优化的打断设计，就直接从报运找到合同下的货物信息。这样就实现了跳跃查询，也就是跳过了多个合同表，而直接获取货物信息。

3）	数据搬家（数据冗余）
在报运时，本身货物信息需要新增7个字段，毛重、净重、长、宽、高、出口单价、含税，还需要合同下的货物的货号、数量、包装单位、件数等。如果按原始设计，读取报运信息后，一部分货物信息要从关联的报运下的货物信息中读取，还有一部分要从合同下的货物读取。这样设计比较麻烦，开发结构复杂，业务不利于开发人员学习业务。我们对其进行优化，我们为了查询更快，关系更简单，代码更简单，将相关的合同在报运新增时，直接把所需要的部分货物信息，货号、数量、包装单位、件数直接复制到报运下的货物表中，这样在报运修改时，直接补充7个字段。在需要报运信息时。就变得简单，直接读取报运下的货物表即可。同时，为了后续业务读取这部分货物信息，还有它们关联的附件信息，我们直接也将合同下的附件信息整体搬迁。这样后续的业务直接可以从报运相关的表中查询，而无需通过多级关联去合同下去找，关联曾经变少，效率提高很多。

4）	一对一的特殊设计
业务装箱和委托、发票、财务都是一对一的。我们为了设计上清晰，我们采用4张。主外键为一个值，委托、发票、财务直接就存放的是装箱单的ID。这样在程序中只要获得其中一个的ID，就可以查询其中任何一个对象。改变传统设计，传统设计财务关联发票，发票关联委托，委托关联装箱。财务需要获取报运的信息和装箱的信息，这时要财务要找到发票，发票找到委托，委托找到装箱，装箱找到多个报运。这样关联层级过多，速度比较慢。我们进行优化设计，改变对应关系，直接设计财务和装箱进行关联。上面的需求就非常简单，通过财务找到装箱，直接获取装箱的部分需要的信息，在通过装箱的打断设计字段，直接获取报运下的货物信息。实现多次跳跃查询。


#技术亮点
页面结构，批量修改自定义控件
在报运货物的修改中，批量进行数据修改，批量的业务就可以直接套用。利用动态表格技术，实现动态的往一个table中增加tr，动态往tr总增加td，它文本框，单选框，多选框，下拉框，大文本。利用一个js函数addRecord反复调用，就可以实现往表格中插入数据。数据在后台service层进行js串拼接，在拼接过程中，将业务数据就拼接进去。然后利用jQuery ready事件进行js动态执行。由于js运行非常快，用户感知不到表格数据的动态增加，他以为直接展现。如果数据在600行。这时不能使用通过ready事件一行一行执行。如果超过600行，采用<c:forEach>的方式来实现。在批量修改时，我就增加了一个行标识，来标识本行记录是否修改，如果修改了，在后台就进行更新，如果没有修改，就不更新操作。

POI复杂报表打印
购销合同复杂报表的打印，这个报表中需要插入用户LOGO，可以插入多个货物的图片，可以插入一根分割线，对合计的值进行公式的设置，打印时用户可以自定义设置打印一款货物，还是打印两款货物，货物生产厂家如果不同，还必须另起一页。获取一个合同信息和多个货物信息。

POI百万数据导出
在POI高版本中提供了一个特殊的构造方法SXSSF对象，利用这个对象，在构造时可以设定一个参数，产生对象的数量，根据这个数量，在大数据量导出时，当创建这么多对象后，直接POI写到xml临时文件中。当真正写文件时，再从临时文件中获取信息，输出内容。这样做解决了早期版本中，直接在内存中创建大量对象，导致非常容易内存堆溢出。它不支持模板文件操作。

CXFWebService服务
客户要求可以跟踪它的订单，我们系统中为客户就提供了WebService服务，我们采用最强大的Apache CXF作为WebService服务。修改出口报运单Service层将它改造为WebService。WebService的实体对象必须序列化，所以保证po对象都序列化。在WebService调用的方法的参数不能是接口对象。必须改造为具体的实现类。然后利用cxf核心配置文件cxf-servlet.xml发布WebService服务。当系统启动时，cxf容器同时启动，就像外部的系统提供访问的地址。；例如：我们有个客户他的系统就是使用.net技术实现，我们发布的WebService。给他说明书，他在他的系统中集成了WebService。就直接可以再他的系统中跟踪他们的订单，直接实时了解订单走到哪个流程。