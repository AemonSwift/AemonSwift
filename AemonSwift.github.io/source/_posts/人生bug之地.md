---
title: 人生bug之地
date: 2019-10-29 08:23:09
tags: 缺陷
categories: 
- 缺陷
- 工作错误
toc: true
---
# bug引入时间
2019年10月27日，毕业半年多，第一次写了人生第一个bug。由于修改了别人的代码：其它服务调用此接口时会发生记录信息丢失问题，以致于线上出现了部分属性信息丢失原因。
```go
func (d *Dao) UpdateChannelInfo(ctx context.Context, req *v1.UpdateChannelInfoReq) (int64, error) {
	var (
		_updateChannelInfo = "update channel set  "
		field              []string
		args               []interface{}
	)
	if req.Name != "" {
		field = append(field, "name=? ")
		args = append(args, req.Name)
	}
	field = append(field, " Background=?") //引入bug，去掉了Background不为""的判断
	args = append(args, req.Background)
	if req.Latest != 0 {
		field = append(field, " latest=? ")
		args = append(args, req.Latest)
	}
	field = append(field, " alpha=?")
	args = append(args, req.Alpha)
	if len(args) == 0 {
		return 0, nil
	}
	args = append(args, req.Cid)
	res, err := d.db.Exec(ctx, _updateChannelInfo+strings.Join(field, ",")+" where cid=? ;", args...)
	if err != nil {
		log.Error("d.db.Exec() error(%v)", err)
		return 0, err
	}
	return res.LastInsertId()
}
```

# bug的原因

此接口调用未考虑到有一方调用——只修改Latest属性是另外一个服务来调用；其它属性的修改又是一个服务来调用。由于自己的疏忽，未考虑到此接口供多方服务来进行调用——只修改Latest属性的调用把Background和alpha都置为0.

## 潜在的bug

若对入参数进行这样设置：
```go
message UpdateChannelInfoReq {
    int64 cid = 1 [(gogoproto.moretags) = "form:\"cid\" validate:\"required,min=1\""];
	string name = 2 [(gogoproto.moretags) = "form:\"name\""];
    string icon = 3 [(gogoproto.moretags) = "form:\"icon\""];
    string color = 4 [(gogoproto.moretags) = "form:\"color\""];
    string background = 5 [(gogoproto.moretags) = "form:\"background\""];
	int64 latest = 6 [(gogoproto.jsontag) = "latest", (gogoproto.casttype) = "go-common/library/time.Time"];
    int32 alpha=7 [(gogoproto.moretags) = "form:\"alpha\" validate=\"gt=0\""];
}
```
出现如下问题：有一方服务想修改Latest属性，会一直发生请求错误。因为：alpha参数属性是必须大于0的。特别是job服务很难进行排查。
# 如何避免类似问题

1. 设计写接口的时候，尽量做到功能单一，且做到一个写接口只给一个服务方来调用。例如：上面只修改属性可以拆成一个接口；其它属性修改拆成另外一个接口。
2. 测试要认真：一定要从线上，预发，以及线上灰度上面认真测试，特别是修改别人代码的接口（修改一行也要测），以及关注mysql的qps（防止接口中写入慢查询），日志的错误信息（防止请求错误过多，例如上述的参数校验导致了更新Latest属性的接口一直请求错误）。
3. 数据库里面的数据一定要进行每日备份，错误怎么避免都会犯错的，但最坏打算就是恢复数据。

测试修改别人代码的接口时候，一定要先把之前记录copy出来，然后进行修改，进行对比，特别是线上更需要进行对比（经过大约三四个小时，再去对比几次）。通过app查看和表来查看。