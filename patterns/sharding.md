# 分片模式

将数据存储区划分为一组水平分区或分片。 这可以提高存储和访问大量数据时的扩展性。

### 背景和问题

由单个服务器托管的数据存储可能会受到以下限制：
* **储存空间**。预计大规模云应用的数据存储将包含大量可能随时间显著增加的数据。通常服务器仅提供有限数量的磁盘存储，但是可以用较大的磁盘替换现有磁盘，或者随着数据卷的增长，将额外的磁盘添加到服务器。然而，系统最终将达到极限，无法再轻松地给服务器增加存储容量。
* **计算资源**。需要云应用程序来支持大量并发用户，每个并发用户运行查询从数据存储中检索信息。托管数据存储的单个服务器可能无法提供必要的计算能力来支持此负载，从而在尝试存储和检索数据超时的应用程序时，导致更多的响应时间和频繁的故障。可以添加内存或升级处理器，但如果无法进一步增加计算资源，系统将达到极限。
* **网络带宽**。最终，在单个服务器上运行的数据存储的性能受服务器可以接收请求和发送回复的速率的约束。网络流量可能会超过用于连接到服务器的网络的容量，导致请求失败。
* **地理**。因为法律、合规性或者性能、减少数据访问延迟的原因，可能需要为同一区域的特定用户存储生成的数据。如果用户分散在不同的国家或地区，可能无法将应用程序的所有数据存储在单个数据存储中。

通过添加更多磁盘容量，处理能力，内存和网络连接来垂直扩展可以延缓其中一些限制的影响，但这些只是临时解决方案。能够支持大量用户和数据的商业云应用程序必须能够无限伸缩，因此垂直扩展不一定是最佳解决方案。

## 解决方案

将数据存储区划分为水平分区或分片。每个分片具有相同的schema，但拥有自己独特的数据子集。分片是自己的数据存储（可以包含许多不同类型实体的数据），在作为存储节点的服务器上运行。
这种模式具有以下好处：
* 可以通过添加在其它存储节点上运行的碎片进一步的来扩展系统。
* 系统可以使用现成的硬件，而不是为每个存储节点使用定制和昂贵的服务器。
* 通过平衡分片上的工作负载，可以减少争用并提高性能。
* 在云中碎片可以靠近访问数据的用户。

将数据存储区划分成碎片时，决定在每个分片中放置哪些数据。分片通常包含落在由数据的一个或多个属性确定的指定范围内的项目。这些属性形成分片键（有时称为分区键）。分片键应该是静态的。不应该基于可能会改变的数据。
分片物理组织数据。当应用程序存储和检索数据时，分片逻辑将应用程序引导到适当的碎片。分片逻辑可以用应用程序中的数据访问代码的一部分实现，或者如果数据存储系统透明地支持分片，也可以实现该分片逻辑。
在分片逻辑中提取数据的物理位置可以高效地控制分片包含哪些数据。如果分片中的数据稍后需要重新分配（例如，如果分片变得不平衡），它还可以使数据在分片之间迁移，而无需重新调整应用程序的业务逻辑。需要权衡每个数据项的检索位置时所需的附加数据访问开销。
为了确保最佳性能和可扩展性，重要的是以适合应用程序执行的查询类型的方式拆分数据。在许多场景下，分片方案不太可能完全符合每个查询的要求。例如，在多租户系统中，应用程序可能需要使用租户ID检索租户数据，但也可能需要根据某些其它属性（如租户的姓名或位置）查找此数据。要处理这些情况，使用支持最常执行查询的分片键来实施分片策略。

如果查询使用属性值的组合定期检索数据，则可以通过将属性链接在一起来定义复合分片键。或者使用诸如[索引表模式](index-table.html)可以基于未被分片键覆盖的属性快速查找数据。

## 分片策略

通常使用三种策略选择分片键并决定如何在分片间分配数据。请注意，分片和托管它们的服务器之间不一定是一一对应的-单个服务器可以托管多个分片。 这些策略是：
**查询策略**。该策略中，分片逻辑实现了使用分片键将数据请求路由到包含该数据的分片的映射。在多租户应用程序中，租户的所有数据可以使用租户ID作为分片键一起存储在分片中。多个租户可能共享相同的分片，但单个租户的数据不会分散在多个分片之间。下图示出了基于租户ID划分租户数据。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/sharding-tenant.png)

碎片键和物理存储之间的映射可以基于物理碎片，其中每个碎片键映射到物理分区。或者，更灵活的重新平衡分片的技术是虚拟分区，其中分片键映射到相同数量的虚拟分片，其又映射到更少的物理分区。在这种方法中，应用程序使用指向虚拟分片的分片密钥来定位数据，系统将虚拟分片透明地映射到物理分区。虚拟分片和物理分区之间的映射可以改变，而不需要修改应用程序代码以使用不同的分片键集合。

**范围策略**。此策略将相关项目组合在同一个分片中，并通过分片键排序-分片键是顺序的。对于使用范围查询（返回一个分配给指定范围内的分片数据项的一组数据项的查询）的应用程序很有用。例如，如果应用程序需要经常查找给定月份中的所有订单，一个月的所有订单以相同的分片的日期和时间顺序存储，可以更快地检索到数据。如果每个订单都存储在不同的分片中，则必须通过执行大量的点查询（返回单个数据项的查询）来单独获取。下图说明了以碎片存储数据的顺序集（范围）。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/sharding-sequential-sets.png)

在这个例子中，分片键是一个复合键，其中最重要的元素包含订单月份，以及订单日期和时间。订单数据自然会在创建新订单并添加到分片时排序。某些数据存储支持两部分的分片键，其中包含标识分片的分区键元素和唯一标识分片中的项目的行键。数据通常在分片中按行键顺序保存。范围查询且需要分组在一起的项目可以使用分区键具有相同值，同时行键的值唯一的分片键，。
**哈希策略**。该策略的目的是减少热点（接收到不相称的负载的碎片）的机会。它可以在数据分片之间分配数据，以实现每个分片的大小与每个分片将遇到的平均负载之间的平衡。分片逻辑基于数据的一个或多个属性的散列来计算分片以存储项目。使用的散列函数应该将数据均匀地分布在分片上，可能需要在计算中引入一些随机元素。下图说明了基于租户ID散列的租户数据。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/sharding-data-hash.png)

要了解哈希策略相比其它分片策略的优势，考虑顺序注册新租户的多租户应用程序，如何将租户分配给数据存储中的分片。当使用范围策略时，租户1到n的数据将全部存储在分片A中，租户n + 1至m的数据将全部存储在分片B中，依此类推。如果最近注册的租户也是最活跃的，大多数数据活动将发生在少数分片中，这可能导致热点。相比之下，哈希策略根据租户ID的哈希值将租户分配给分片。这意味着顺序租户最有可能被分配到不同的分片，这将分散负载。如上图中租户55和56。

三个分片策略具有以下优点和考虑：
**查询**。这样可以更好地控制碎片的配置和使用方式。使用虚拟分片减少重新平衡数据时的影响，因为可以添加新的物理分区以使工作负载平衡。可以修改虚拟分片和实现分片的物理分区之间的映射，而不会影响使用分片键来存储和检索数据的应用程序代码。查找碎片位置可能会增加额外的开销。
**范围**。这很容易实现，并且适用于范围查询，因为它们通常可以在单个操作中从单个分片中获取多个数据项。这一策略提供更简单的数据管理，例如，如果同一区域中的用户处于相同的分片，则可以根据本地的负载和需求模式在每个时区中调度更新。但是，这种策略不能在分片之间提供最佳平衡。如果大多数活动用于相邻的分片键，重新平衡分片是困难的，并且可能无法解决负载不均匀的问题。
**哈希**。这种策略提供了更加均匀的数据和负载分配的机会。可以通过使用哈希函数直接完成请求路由。没有必要维护映射。请注意，计算哈希可能会增加额外的开销。此外，重新平衡分片比较困难。

最常见的分片系统都使用上述方法之一实现，但还应考虑应用程序的业务需求及其数据使用模式。例如，在多租户应用程序中：
* 可以根据工作负载分割数据。将单独碎片中高度易失性租户的数据隔离。这样其他租户的数据访问速度可能会得到改善。
* 可以根据租户的位置来划分数据。将特定地理区域内的租户的数据在该地区的非高峰时段离线进行备份和维护，而其他地区的租户的数据在工作时间内保持在线状态。
* 高价值租户可以分配自己的私人，高性能，低负载的分片，而低价值租户可能会分享更密集包装、繁忙的分片。
* 需要高度数据隔离和隐私的租户的数据可以存储在完全独立的服务器上。

## 扩展和数据移动操作

每个分片策略意味着管理收缩，扩展，数据移动和维护状态的不同能力和复杂级别。
查询策略允许在用户级别执行缩放和数据移动操作，无论是在线还是离线。该技术会暂停某些或所有用户活动（可能在非高峰期），将数据移动到新的虚拟分区或物理分片，更改映射，使保存此数据的任何缓存无效或刷新，然后允许用户活动恢复。通常这种类型的操作可以集中管理。查询策略要求状态高度可缓存和复制友好。
范围策略对缩放和数据移动操作施加了一些限制，通常必须在数据存储的一部分或全部脱机时执行，因为数据必须在分片之间进行拆分和合并。移动数据平衡分片可能无法解决不均匀加载的问题，如果大多数活动用于相同范围内的相邻分片键或数据标识符。范围策略还可能需要维护一些状态，以将范围映射到物理分区。
哈希策略使缩放和数据移动操作更复杂，因为分区键是分片键或数据标识符的散列。每个分片的新位置必须从散列函数确定，或修改为提供正确映射的函数。但是，哈希策略不需要维护状态。

## 问题和注意事项

在决定如何实现此模式时，请考虑以下几点：

* 分片与其它形式的分区互补，如垂直分区和功能分区。例如，单个分片可以包含已经垂直分区的实体，并且功能分区可以以多个分片的方式实现。有关分区的更多信息，请参阅[数据分区指南](https://msdn.microsoft.com/library/dn589795.aspx)。
* 保持分片平衡，平衡I/O处理的容量。随着数据的插入和删除，有必要定期重新平衡分片，以保证均匀分布，并减少热点的问题。再平衡的成本可能很高。为了减少重新平衡的必要性，通过确保每个分片都包含足够的可用空间来处理预期的变化量来计划增长。如果有必要，还应该开发可用于快速重新平衡碎片的策略和脚本。
* 使用稳定的数据作为分片的键。如果键更改，相应的数据项可能需要在分片之间移动，从而增加更新操作执行的工作量。因此，避免将键置于潜在的易失性信息上。相反，寻找不变量或自然形成关键字的属性。

* 确保分片键是唯一的。例如，避免使用自增字段作为分片键。某些系统的自增字段无法在分片上进行协调，可能导致不同分片中的项目具有相同的分片键。
> 非分片键的自增字段也可能会导致问题。例如，如果使用自增字段来生成唯一的ID，那么位于不同分段中的两个不同的项目可能会被分配相同的ID。
* 可能无法设计匹配每个可能的数据查询要求的分片键。分割数据以支持最常执行的查询，如有必要可以创建辅助索引表，支持不属于分片键的属性的标准来检索数据的查询。详细信息，请参阅[索引表模式](index-table.html)。
* 仅访问单个分片的查询比从多个分片中检索数据的查询更有效，因此分片系统要避免设计成让应用程序在执行大量join查询时，从多个不同的分片读取数据。单个分片可以包含多种类型的实体的数据。考虑对数据进行非规范化，以便将通常在一起查询的相关实体（例如客户的详细信息和他们所放置的订单）保留在同一个分片中，以减少应用程序执行的独立读取次数。

> 如果一个分片中的实体引用存储在另一个分片中的实体，则将第二个实体的分片键作为第一个实体的schema的一部分。这可以帮助提高需要在分片上引用相关数据的查询的性能。

* 如果应用程序必须执行从多个分片中检索数据的查询，可能可以通过并行任务来获取这些数据。比如扇出查询的例子，并行检索来自多个分片的数据，然后聚合成单个结果。然而，这种方法不可避免地为数据访问逻辑增加了一些复杂性。

* 对于很多应用程序来说，创建大量小碎片可以比拥有少量大碎片更有效，因为它们可以提供更多的负载均衡机会。将碎片从一个物理位置迁移到另一个物理位置的场景也非常适用。移动一个小碎片比移动一个大碎片更快。

* 确保每个分片存储节点可用的资源足以在数据大小和吞吐量方面满足可扩展性要求。详细信息请参见[数据分区指南](https://msdn.microsoft.com/library/dn589795.aspx)中的“设计可扩展性分区”一节。

* 考虑将参考数据复制到所有分片。如果从分片中检索数据的操作也引用静态或缓慢移动的数据作为查询的一部分，请将此数据添加到分片。然后应用程序就可以轻松获取查询的所有数据，无需再次单独访问数据存储。

* 如果保存在多个分片中的参考数据发生更改，则系统必须在所有分片之间同步这些更改。同步发生时，系统可能会遇到一定程度的不一致。如果要这么做，应该设计应用程序具备处理的能力。

* 维护引用完整性和分片之间的一致性可能很困难，应该最大程度地减少影响多个分片中的数据的操作。如果应用程序必须在分片之间修改数据，请评估是否需要完整的数据一致性。相反，云中的常见做法是实现最终一致性。每个分区中的数据分别更新，应用程序逻辑必须承担责任，确保更新全部成功完成，并处理在最终一致的操作运行时查询数据时可能产生的不一致。有关实现最终一致性的更多内容，请参阅[数据一致性入门](https://msdn.microsoft.com/library/dn589800.aspx)。

* 配置和管理大量分片可能是一个挑战。监控，备份，检查一致性以及日志记录或审核等任务必须在多个碎片和服务器上完成，可能会保存在多个位置。这些任务可以使用脚本或其它自动化解决方案来实现，但可能无法完全消除额外的管理需求。

* 碎片可以进行地理位置定位，使其包含的数据与使用它的应用程序的实例接近。这种方法可以显著提高性能，但需要额外考虑对必须访问不同位置的多个分片的任务。

### 何时使用该模式

当数据存储可能需要扩展超过单个存储节点可用的资源时，或者通过减少数据存储中的资源争夺来提高性能，可以使用此模式。
> 分片的重点是提高系统的性能和可扩展性，其副产品还可以通过将数据分割为单独的分区来提高可用性。 一个分区中的故障不一定妨碍应用程序访问其他分区中保存的数据，并且操作员可以执行维护或恢复一个或多个分区，而不会使应用程序的所有数据无法访问。有关更多信息，请参阅[数据分区指南](https://msdn.microsoft.com/library/dn589795.aspx)。

## 案例

以下例子中C＃代码使用一组作为碎片的SQL Server数据库。每个数据库保存应用程序使用的数据子集。应用程序使用自己的分片逻辑（这里是扇出查询的例子）检索分片在碎片之间的数据。`GetShards`方法返回位于每个分片中的数据的详细信息。该方法返回`ShardInformation`对象的枚举列表，其中`ShardInformation`类型包含每个分片的标识符和应用程序用于连接到分片的SQL Server连接字符串（代码示例中未显示连接字符串）。

```c#
private IEnumerable<ShardInformation> GetShards()
{
  // This retrieves the connection information from a shard store
  // (commonly a root database).
  return new[]
  {
    new ShardInformation
    {
      Id = 1,
      ConnectionString = ...
    },
    new ShardInformation
    {
      Id = 2,
      ConnectionString = ...
    }
  };
}
```

下面的代码显示了应用程序如何使用`ShardInformation`对象列表执行并行从每个分片中获取数据的查询。查询的详细信息未显示，但例子中检索的数据包含一个字符串，如果该分片包含客户的详细信息，它可以保存诸如客户名称等信息。查询结果汇总到`ConcurrentBag`集合中以供应用程序处理。

```c#
// Retrieve the shards as a ShardInformation[] instance.
var shards = GetShards();

var results = new ConcurrentBag<string>();

// Execute the query against each shard in the shard list.
// This list would typically be retrieved from configuration
// or from a root/master shard store.
Parallel.ForEach(shards, shard =>
{
  // NOTE: Transient fault handling isn't included,
  // but should be incorporated when used in a real world application.
  using (var con = new SqlConnection(shard.ConnectionString))
  {
    con.Open();
    var cmd = new SqlCommand("SELECT ... FROM ...", con);

    Trace.TraceInformation("Executing command against shard: {0}", shard.Id);

    var reader = cmd.ExecuteReader();
    // Read the results in to a thread-safe data structure.
    while (reader.Read())
    {
      results.Add(reader.GetString(0));
    }
  }
});

Trace.TraceInformation("Fanout query complete - Record Count: {0}",
                        results.Count);
```

### 相关模式和指南

以下模式和指南在实现此模式时也可能相关：

* [数据一致性入门](https://msdn.microsoft.com/library/dn589800.aspx)。需要保持分布在不同分片之间的数据的一致性。这篇文章总结了关于保持分布式数据一致性的问题，并描述不同一致性模型的优点和权衡。
* [数据分区指南](https://msdn.microsoft.com/library/dn589795.aspx)。分割数据存储可能会引入一系列其它问题。这篇文章介绍了在云中分区数据存储的问题，以提高可扩展性，减少争用并优化性能。
* [索引表模式](index-table.html)。有时不可能通过分片键的设计完全支持查询。允许应用程序通过指定除了分片键之外的键来快速从大型数据存储中检索数据。
* [物化视图模式](materialized-view.html)。为了保持某些查询操作的性能，创建物化视图以聚合和汇总数据是非常有用的，特别是如果摘要数据基于分布在分片上的信息。物化视图模式描述了如何生成和填充这些视图。
* Adding Simplicity 网站上的[分片教程](http://www.addsimplicity.com/adding_simplicity_an_engi/2008/08/shard-lessons.html)。
* CodeFutures网站上的[数据库分片教程](http://dbshards.com/database-sharding/)。
* [可扩展性策略入门](http://blog.maxindelicato.com/2008/12/scalability-strategies-primer-database-sharding.html)：Max Indelicato博客上的数据库分片教程。
* Dare Obasanjo博客[构建可扩展数据库](http://www.25hoursaday.com/weblog/2009/01/16/BuildingScalableDatabasesProsAndConsOfVariousDatabaseShardingSchemes.aspx)：Dare Obasanjo博客中各种数据库分片方案的优缺点。