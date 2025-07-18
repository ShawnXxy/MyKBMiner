# 技术译文 | 开发人员应该了解哪些 SQL 知识？

**原文链接**: https://opensource.actionsky.com/%e6%8a%80%e6%9c%af%e8%af%91%e6%96%87-%e5%bc%80%e5%8f%91%e4%ba%ba%e5%91%98%e5%ba%94%e8%af%a5%e4%ba%86%e8%a7%a3%e5%93%aa%e4%ba%9b-sql-%e7%9f%a5%e8%af%86%ef%bc%9f/
**分类**: 技术干货
**发布时间**: 2024-01-16T00:05:23-08:00

---

SQL（结构化查询语言）是数据库的通用语言，它无处不在、功能强大，并且对于开发人员来说理解非常重要。从这些技巧开始。
> 
作者：Charly Batista
本文和封面来源：[https://www.infoworld.com/，爱可生开源社区翻译。](https://www.infoworld.com/，爱可生开源社区翻译。)
本文约 2700 字，预计阅读需要 9 分钟。
自 20 世纪 70 年代初发明 [SQL](https://www.infoworld.com/article/3219795/what-is-sql-the-lingua-franca-of-data-analysis.html) 以来，它一直是管理与数据库交互的默认方式。根据 [Stack Overflow 的数据](https://survey.stackoverflow.co/2023/#overview)， **SQL 仍然是排名前五的编程语言之一，大约 50% 的开发人员在工作中使用它。** 尽管 SQL 无处不在，但它仍然以困难或令人生畏而闻名。只了解 SQL 是什么，还是远远不够的。
同时，由于当今的企业越来越重视他们的数据，因此熟练使用 SQL 将为你提供更多机会，让你成为一名优秀的软件开发人员并推动职业发展。那么应该了解 SQL 哪些知识，以及应该避免哪些问题呢？
# 不要害怕 SQL
SQL 很容易使用，因为它是结构化的。SQL 严格定义了如何将查询组合在一起，使它们更易于阅读和理解。如果你正在查看其他人的 SQL，应该很容易理解他的的查询目标。
然而，许多开发人员对复杂 SQL 望而却步，可能是因为当初学到的第一个命令：**SELECT**。开发人员在开始编写 SQL 时最常犯的错误就是 `SELECT *`。
使用 SELECT 查询内容太多，会对性能产生很大影响，并且随着时间的推移，它可能会导致优化查询变得困难。查询内容是否有必要，或者是否可以更具体？这会对现实世界产生影响，因为它可能会导致大量 ResultSet 响应，从而影响服务器高效运行所需的内存占用。如果查询涵盖太多数据，最终可能会为其分配超出所需的内存，特别是在云服务中运行数据库时。**云资源需要花钱，错误的 SQL 编写会让你浪费更多的钱。**
# 合适的数据类型
**开发人员在使用 SQL 时另一个常见问题是数据类型不合适。** 常用的两种主要类型的数据：INT 和 VARCHAR。INT 类型包含数字，而 VARCHAR 类型字段可以包含数字、字母或其他字符。如果处理数据时期望一种类型，然后获取另一种类型，则结果中可能会出现数据类型不匹配的情况。
为了避免此问题，请谨慎处理可能经常使用的语句命令和准备好的语句脚本。这将帮助你避免出现期望一种结果却得到其他结果的情况。同样，将任何数据库表放在一起时，应该评估 JOIN 语句。检查数据可以帮助您避免 JOIN 执行此操作时发生任何数据丢失，例如字段中的数据值被截断或隐式转换为不同的值。
**另一个经常被忽视的问题是字符集。** 这很容易被忽视，但请务必检查您的应用程序和数据库在工作中是否使用相同的字符集。使用不同的字符集可能会导致编码不匹配，这可能会完全扰乱您的应用程序视图并阻止您使用特定的语言或符号。在最坏的情况下，这可能会导致数据丢失或难以调试的奇怪错误。
# 数据顺序很重要
**许多开发人员在开始研究数据库时做出的一个假设是，列的顺序不再重要。** 毕竟，我们有许多数据库提供商告诉我们，我们不需要了解具体的数据库，他们的工具可以为我们处理所有这些事情。然而，虽然看起来没有影响，但我们的基础设施可能会产生相当大的计算成本。当使用按使用量收费的云服务时，这一费用会迅速增加。
重要的是要知道，并非所有数据库都是相同的，也不是所有索引都是相同的。例如，列的顺序对于组合索引非常重要，因为这些列是从索引创建顺序的最左边开始计算的。因此，随着时间的推移，这确实会对潜在性能产生影响。
但是，在子句中声明列的顺序 WHERE 不会产生相同的影响。这是因为数据库具有查询计划和查询优化器等组件，它们尝试以最佳执行方式重新组织查询。他们可以重新组织和更改子句中列的顺序 WHERE，但它们仍然依赖于索引中列的顺序。
所以，事情并不像听起来那么简单。了解数据顺序将影响操作和索引的位置可以为提高整体性能和优化设计提供机会。为了实现这一点，数据和运算符的基数非常重要。了解这一点将帮助您制定更好的设计并获得更多的长期价值。
# 注意编程语言差异
对于刚开始使用 SQL 的人来说，一个常见问题是 NULL 对于使用 Java 的开发人员，[Java 数据库连接器（JDBC）](https://www.infoworld.com/article/3388036/what-is-jdbc-introduction-to-java-database-connectivity.html)提供了一个 API 将其应用程序连接到数据库。然而，虽然 JDBC 确实将 SQL 映射 NULL 到 Java 的 null，但它们并不是一回事。SQL 中的命令 NULL 也可以称为 UNKNOWN，这意味着 SQLNULL = NULL 是 FALSE，与 Java 中的 null == null 不一样。
最终结果是算术运算 NULL 可能不会产生期望的结果。了解这一差异后，就可以避免从应用程序的一个元素转换为数据库和查询设计时出现的潜在问题。
在 Java 和数据库方面还有一些其他常见模式需要避免。这些都涉及操作如何以及在何处进行和处理。例如，您可以将来自单独查询的表加载到映射中，然后将它们连接到 Java 内存中进行处理。然而，这在内存中执行要复杂得多，计算成本也高。看看排序、聚合或执行任何数学运算，以便它可以由数据库处理。在绝大多数情况下，用 SQL 编写这些查询和计算比在 Java 内存中处理它们更容易。
# 让数据库完成工作
除了使解析和检查这项工作变得更容易之外，数据库执行计算的速度可能比算法更快。仅仅因为您可以在内存中处理结果并不意味着您应该这样做。出于整体速度的原因，不值得这样做。同样，在内存云服务上的支出比使用数据库提供结果的成本更高。
这也适用于分页。分页涵盖了如何在多个页面而不是一页中对查询结果进行排序和显示，并且可以在数据库或 Java 内存中执行。就像数学运算一样，分页结果应该在数据库中而不是在内存中进行。原因很简单——内存中的每个操作都必须将所有数据带到内存中，进行事务，然后返回到数据库。这一切都通过网络进行，每次执行都会增加一次往返，并增加交易延迟。使用数据库进行这些事务比尝试在内存中执行工作要高效得多。
数据库还有许多有用的命令，可以使这些操作更加高效。通过利用 LIMIT、OFFSET、TOP、START AT，和 FETCH 等命令，可以使分页请求在处理正在使用的数据集的方式方面更加高效。同样，我们可以避免过早的行查找以进一步提高性能。
# 使用连接池
在建立连接和执行事务之前，将应用程序链接到数据库需要工作和时间。因此，如果您的应用程序定期处于活动状态，这将是您想要避免的开销。标准方法是使用连接池，其中一组连接随着时间的推移保持打开状态，而不必在每次需要事务时打开和关闭它们。这是标准化的 JDBC 3.0 的一部分。
但是，并非每个开发人员都实现连接池或在其应用程序中使用它。这可能会导致应用程序性能下降，而这一点很容易避免。与没有连接池的相同系统相比，连接池极大地提高了应用程序的性能，并且还减少了总体资源使用。它还减少了连接创建时间并提供了对资源使用的更多控制。当然，重要的是要检查您的应用程序和数据库组件是否遵循有关关闭连接并将其交还给资源池的所有 JDBC 步骤，以及应用程序的哪个单元将在实践中负责此操作。
# 利用批处理
今天，我们看到人们非常重视实时交易。您可能认为整个应用程序应该实时运行才能满足客户需求或业务需求。然而，情况可能并非如此。与运行多个操作相比，批处理仍然是处理多个事务的最常见和最有效的方法。
使用 JDBC 确实可以提供帮助，因为它支持批处理。例如，您可以使用单个 SQL 语句和多个绑定值集创建批处理 INSERT ，这比独立操作更高效。需要记住的一个因素是在事务非高峰时段加载数据，这样就可以避免对性能造成任何影响。如果这是不可能的，那么您可以定期查看较小的批量操作。这将使您的数据库更容易保持最新，并保持事务列表较小并避免潜在的数据库锁定或竞争条件。
# 总结
无论您是 SQL 新手还是已经使用它多年，它仍然是未来的一项关键语言技能。通过将上述经验教训付诸实践，您应该能够提高应用程序性能并利用 SQL 提供的功能。