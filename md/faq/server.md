# Server FAQ

#### Q. 如何把IM服务倒入到IDE中？
A. [回答](server/q1.md)

#### Q. 我部署完成Demo服务后，客户端能登陆，但一只显示连接中？
A. [回答](server/q2.md)

#### Q. 消息存储在数据库中的那张表中
A. 使用h2db存储消息时，消息存储在t_message标中；使用mysql存储消息时，消息存储在t_message_X表中，X等于 ***月份-1+(年份%3)*12***，例如2019年4月存储在t_message_3表中(4-1+(2019%3)*12).

#### Q. IM服务启动脚本在windows下改过之后，放到linux服务器上运行报错
A. 这是因为Windows下有些编辑器更改完会把脚本转变成DOS格式，linux服务器无法识别DOS格式。解决办法是保存时选择保存为unix格式，或者在linux服务上使用vim打开，然后输入命令```:set fileformat=unix```，最后保存。

#### Q. 为什么数据库中的敏感词数目要比通过API获取到的数目要多
A. 敏感词列表是从网络上搜索出来的，里面有部分重复敏感词。通过server api获取到的敏感词是个集合数据结构，去掉了重复的内容，所以导致了不一致，可以不进行处理，没有任何影响，如果您想清除掉重复内容，请使用下面语句删除重复内容，注意执行时间较长，需要耐心等待
```
DELETE from  `t_sensitiveword` WHERE `_word` IN (SELECT `_word` FROM (SELECT * from `t_sensitiveword`) as t1 GROUP BY `_word` HAVING count(`_word`) > 1) AND `id` NOT IN (SELECT min(`id`) FROM (SELECT * from `t_sensitiveword`) as t2 GROUP BY `_word` HAVING count(`_word`) > 1);
```

#### Q. 为什么在数据库中添加了一个用户/群组/好友，为啥不起作用？
A. 因为服务使用了大量的缓存，只更新数据库是不能起效果的，另外数据还有完整性需求，直接写数据库可能会出问题。所以可以直接从数据库中读信息，但所有更新操作都***必须只能通过服务API的接口来进行***。

#### Q. 为什么机器人服务部署以后，一直提示加密方式错误？
A. 机器人是个demo服务，为了演示机器人服务收发功能，对接了图灵机器人，客户也可以更换任何的自助问答服务。代码里的图灵机器人配置是无效的，需要可以去图灵注册账户，开通机器人权限，然后更新机器人服务的key。图灵地址在[这里](http://www.turingapi.com)

#### Q. 怎么修改小火机器人头像名称这些信息？
A. 使用server api的接口或者购买付费的管理后台可以添加或修改机器人。当然也可以在数据库中新建和修改机器人。新建的sql语句如下：
```
insert into t_user (`_uid`,`_name`,`_display_name`,`_portrait`,`_type`,`_dt`) values ('FireRobot','FireRobot','小火','http://cdn2.wildfirechat.cn/robot.png',1,1);

insert into t_robot (`_uid`,`_owner`,`_secret`,`_callback`,`_state`,`_dt`) values ('FireRobot', 'FireRobot', '123456', 'http://127.0.0.1:8883/robot/recvmsg', 0, 1);

```
新建的机器人是可以立即使用的。也可以修改机器人的user表，修改需要注意两点，一个是```_dt```字段需要增大以便客户端同步，另外一个就是更新后要重新启动IM服务，这两点不做会不生效。

#### Q. 如何做消息审查和过滤？
A. 消息审查和过滤是确保设计软件能够安全运营重要的一环。在野火IM中可以通过如下几种方式：
1. 使用内置的敏感词过滤。野火IM内置了敏感词，可以针对文本中的敏感词进行过滤。可以通过api接口对敏感词进行管理。命中敏感词时会有不通的策略，在配置文件中有说明（有发送失败，吞掉和替换成*三种策略）。命中敏感词的消息也会存储在一张特殊的消息表中（可以根据表明找出来）。野火IM官方提供的[管理后台](https://github.com/wildfirechat/admin)可以提供这方面的管理，客户也可以自己实现。这种方式的缺点是只能处理文本。
2. 使用消息转发功能，在野火IM配置文件中配置转发地址，这样客户端所有消息都会被转发到指定审核服务器，当出现违规信息时，使用api中的撤回功能，撤回消息。这种过滤方式可以过滤图片/视频/语音，或其他自定义消息。难点就是处理方式比较难，需要有大量的开发功能。

#### Q. 历史消息数据如何清理？
A. 野火IM的消息存放有3种选择。第一种是使用h2db，这种只能用户开发模式，或者用户数极少的情况下，对这种情况下重置数据库就行。第二种方法是使用mysql，其中对message表（t_messages_xxxx)和用户message关系表(t_user_message_xxx)都做了分表，message表是按照月份分的，按照文档说明找到对应的表清空就可以了。用户message关系表，是使用用户ID哈希成128个表，需要对每个表里的数据按照时间进行删除。如果表内数据比较大，需要在业务不繁忙阶段，分表分批次删除，需要部分的开发量。第三种方式就是mongodb的方式，系统会自动过期删除，不用人工干预。

#### Q. 服务器一直出现 ```javax.crypto.BadPaddingException: Given final block not properly padded. Such issues can arise if a bad key is used during decryption.``` 这个错误提示？
A. 野火IM客户端与服务器保存有每个客户端的密钥，如果因为某种原因服务器没有对应的密钥（更换服务器没有迁移数据或出现异常情况），服务器将与客户端无法进行配合工作。解决办法就是需要在客户端对某些特殊的状态码进行处理，请参考[连接状态码有什么需要注意的吗](./general.md).

#### Q. Windows系统部署，提示时区不合法（The server time zone value 'ÖÐ¹ú±ê×¼Ê±¼ä' xxx)

A. 去报Mysql版本5.6，或以上，然后参考[这个链接](https://www.cnblogs.com/smiler/p/9983146.html?tdsourcetag=s_pcqq_aiomsg)

#### Q. 服务器端如何发消息？类似问题服务器如何发图片，服务器端如何发自定义消息
A. IM服务的核心功能就是个管道，在管道内传输的不是文本消息，图片消息或自定义消息等具体消息，传输的是[Payload](../base_knowledge/message_payload.md)。客户端上会定义各种消息，包括文本、语音、图片及自定义消息，在发送时统一编码为Payload，然后再收到后再解码为具体的消息。服务器端发送消息时，首先要确认客户端上编码/解码的规则，发送对应的Payload即可。唯一例外的是Payload字段中的二进制数据需要做base64编码。例如图片消息，在android上的定义如下：
```
@ContentTag(type = MessageContentType.ContentType_Image, flag = PersistFlag.Persist_And_Count)
public class ImageMessageContent extends MediaMessageContent {

    @Override
    public MessagePayload encode() {
        MessagePayload payload = super.encode();
        payload.searchableContent = "[图片]";

        Bitmap thumbnail = ThumbnailUtils.extractThumbnail(BitmapFactory.decodeFile(localPath), 200, 200);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        thumbnail.compress(Bitmap.CompressFormat.JPEG, 75, baos);
        payload.binaryContent = baos.toByteArray();

        return payload;
    }


    @Override
    public void decode(MessagePayload payload) {
        super.decode(payload);
        thumbnailBytes = payload.binaryContent;
    }
  }

  ...
}

public abstract class MediaMessageContent extends MessageContent {

    @Override
    public MessagePayload encode() {
        MessagePayload payload = new MessagePayload();
        payload.localMediaPath = this.localPath;
        payload.remoteMediaUrl = this.remoteUrl;
        payload.mediaType = mediaType;
        return payload;
    }

    @Override
    public void decode(MessagePayload payload) {
        this.localPath = payload.localMediaPath;
        this.remoteUrl = payload.remoteMediaUrl;
        this.mediaType = payload.mediaType;
    }

    ...
}
```
因此服务器发送图片的参数为
```
{"payload":{"type":3,"searchableContent":"[图片]","base64edData":"图片缩略图的base64内容","mediaType":1,"remoteUrl":"http://图片的地址"}}
```
> type是消息类型，3(MessageContentType.ContentType_Image)为图片。mediaType是媒体类型，对应文件服务器的bucket，1(MessageContentMediaType.IMAGE)为图片的桶。

#### Q. 服务器端存储的离线消息数是多少，能改动吗，怎么改动？
A. 服务器端为每个用户在内存中保存了一个时间线，默认是1024条，当用户有超过1024条消息没有接收时，旧的消息将被抛弃，只保证最新1024条最新消息。这个值可以改大，但改大带来的风险是每个用户的内存占用量会提升很多（每增加1，每个用户的内存占用增加50个字节，需要考虑到用户数和服务器内存大小。专业版不受此影响），另外大量消息的拉取时间也变得很长，影响客户端体验。这个值在配置文件中能找到。

#### Q. 敏感词过滤太严格了。
A. 敏感词是社交运营必不可少的一环，国内运营IM必须进行监管，防止触犯法律。现在敏感词设置过于严格而且有些过时，因此客户可以在数据库中删掉现有的敏感词添加自己设置的敏感词（不要修改数据库脚本，因为修改了脚本，升级版本时flyway对数据库的校验会无法通过的，可以等生成了库后再删除），数据库添加的敏感词会在1小时内自动加载生效。另外也可以看一下配置文件，修改敏感词的策略，敏感词只记录下来，不阻拦发送。事后再检查敏感词消息。另外我们提供有[管理后台](https://github.com/wildfirechat/admin)，客户可以购买进行包括敏感词在内的管理.

#### Q. IM服务启动时报错，错误原因是flywayXXXX
A. 野火IM使用了[flyway](https://flywaydb.org)作为数据库migration工具。flyway会对数据库脚本的一致性做校验，如果您改动过sql脚本就会出现校验不过的问题。强烈建议您不要修改sql语句。不要尝试把业务逻辑加到IM服务中。

#### Q. t_messages_xxxx和t_user_messages_xxxx都是做什么用的有什么关系。
A. t_messages是存放消息实体的，每发送一条消息都会存储下来一条；t_user_messages是存放用户消息对应关系的，每条消息的每个目标用户都会存储一条记录。比如在一个1000人的群里发送一条消息，t_messages会存储一条记录，t_user_messages就会存储1000条记录。因此随着时间的增加这两张表的数据内容可能会非常大，因此对这两张表做了分表处理。t_messages的分表可以参加前面问题3，t_user_messages的分表是用户id的hash值对128取模，因此就有128张表。专业版可以选用mongodb存储这两部分内容，mongodb在海量数据时比mysql有着更好的性能表现，使用mongodb时，mysql数据库中还会继续存在这些表，只不过是不使用了，不要删除，保留不变，防止数据库有效性校验出错。

#### Q. 禁止多端登录无效！
A. 禁止多端的逻辑是这样的：客户端在login时同时带上```clientId```和```platform```，appserver在获取token时同时带上```clientId```和```platform```，IM服务生产token时会踢掉相同平台的其它设备。IM server中有个配置如下，需要设置为false，如果不生效，则检查appserver和客户端在登录时是否携带了```platform```。
```
# 是否支持多端登陆
server.multi_endpoint false
```
