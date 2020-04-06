# tmpResource

基于cloudsop平台是指使用平台微服务提供的底层功能，比如记录安全日志、操作日志、lincese权限等功能，便于简化开发流程。

后台架构：springboot、spring、mybatis、zenith高斯数据库

web浏览器端发起https请求到web后端，由于后台所有微服务的各种接口会定义在微服务配置文件的app_define.json的pattern中ER、IR接口会注册到平台的总线上，因此请求就可以通过总线服务找到对应的微服务，然后根据引入了平台的bsp包底层封装好实现了请求映射到相应的controller进入对应的方法，走相应的业务逻辑。

微服务之间的接口请求接收层是通过配置yaml文件，根据swagger以及引入codegen自研代码生成插件根据模板自动生成rest接口的请求与接收代码的。

用户发送请求道API GW（以下称为网关），网关将请求转发到IR总线（内部路由），再由总线转发到各个微服务。想通过IR总线转发请求，首先需要将转发的接口发布到总线，发布方式为在微服务的app_define.json文件中添加。

1、每个服务的APP_ROOT路径下有个pub目录，pub目录下面可以保存该服务的定制信息，由服务自己管理和维护。
2、CloudSOP平台上有个DeployAgent服务，DeployAgent服务是服务部署在每个节点的代理，DeployAgent提供了很多rest接口供外部调用
A服务想要获取B服务的定制化配置信息，可以这样做：
1、A服务向DeployAgent注册需要的文件列表（如careFile.json）
2、B服务将配置项配置到自己pub路径下的careFile.json中
3、A服务调用DeployAgent的rest接口去获取B服务careFile.json的配置信息并更新到A的缓存中。
<br>
  这样有一个问题：B的配置项由B自己维护管理，那么B想怎么改就怎么改，想什么时候改就什么时候改，但是A却要用B的配置项，如果B的配置发生了变化（也就是B重新部署了），A却不知道，那A岂不是很尴尬？为了解决这个问题，A需要向DeployAgent打声招呼，告诉DeployAgent：A关注了某某文件，如果这个文件有变动，请告诉A它（它们）变成什么了。这就是服务注册的回调接口。
<br>
  注册是通过在服务的app_define.json中配置文件列表和rest接口来实现的，比如A服务需要实时获取某配置项，该配置项是由B定制的，则A可以和B约定：B将配置项配置在B的pub目录下，假设文件名约定为careFile.json（也可以是.xml）。然后A在自己的app_define.json中进行配置。这样，当DeployAgent发现pub路径下有careFile.json变化时（变化指：随某服务部署新增，随某服务卸载删除。手动修改不算），就会回调A服务的/rest/plat/XXX/v1/pub-files接口来告诉A服务：某某careFile.json变更啦，它（它们）的摘要现在是什么什么（注意：DeployAgent不仅关注B服务的careFile.json，它关注所有服务的pub目录下的careFile.json）。然后A就可以根据DeployAgent发来的请求消息中的文件名全路径和摘要获取变更或者新增的文件中的配置，获取的方式有两种：
1、直接读文件。此时A已经有了文件全路径，可以自己读文件
2、调用DeployAgent的GET /rest/plat/deploy-agent/v1/pub-files?file-etag=\*接口，通过摘要获取文件内容。其中\*为文件摘要。
