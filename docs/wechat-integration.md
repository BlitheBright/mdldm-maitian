# 公众号接入（设计说明）

`examples/wechat/` 里的示例是按这份说明实现的。核心文件不用改：平台清单只影响 `doctor`/`login` 的默认列表，`run`/`sync`/`metrics` 都以 `publish.json.platforms` 为准，单期页以数据库里的 `tasks` 为准。

## 三条链路

### 1. 采集（`collectors.wechat`）

- 数据源：`/cgi-bin/appmsgpublish?sub=list&begin=0&count=10&token=…&lang=zh_CN&f=json&ajax=1`
- **两层字符串**：`publish_page` 是字符串，`publish_list[].publish_info` 也是字符串，都要二次 `JSON.parse`。
- 取 `appmsg_info[]` 里的 `read_num` / `like_num` / `old_like_num`（在看）/ `comment_num`，`sent_info.time` 是发表时间的 unix 秒。
- 匹配：先按 `content_url` 的 slug，再按标题精确/前缀；找不到就返回 null，**不拿别人的数**。

### 2. 粉丝（`followerCollectors.wechat`）

- `/misc/useranalysis` 的 JSON 接口所有 action 都返回空，改读页面 DOM。
- 页面里的趋势图坐标轴也带着日期，直接正则会把坐标轴当数据行；**只从「详细数据表格」之后开始解析**。

### 3. 发布（适配器 `fill`）

- 封面：`POST /cgi-bin/filetransfer?action=upload_material&…`，multipart `type=image`；先页面内 canvas 居中裁 900×383 转 JPEG 再传，省流量也符合封面比例。
- 草稿：`POST /cgi-bin/operate_appmsg?t=ajax-response&sub=create&type=77`，`application/x-www-form-urlencoded`，约 60 个字段（`isnew=0`、`show_cover_pic0=1`、四个 `cdn_*` 填同一个值、`share_imageinfo0='{"list":[]}'`）。
- `uploadDone: null`：公众号没有「填好等你自己点发布」的页面，草稿建好就结束，状态停在「已填写」，由人进后台发。
- `links.collect`：拉 `appmsgpublish` 按标题精确匹配，拿到 `content_url` 才算已发布。

## 踩过的坑

| 现象 | 原因 |
|---|---|
| 用错 token 打接口返回 `invalid csrf token` | token 要从当前后台页 URL 取，先落到 `/cgi-bin/home` |
| `action=list_card&type=77` 的 JSON 恒为 `{"items":[]}` | 该接口对本账号不可用，草稿箱只能读页面 DOM |
| 粉丝页抓到「11-05」这种假行 | 趋势图坐标轴；只解析「详细数据表格」之后 |
| 群发接口调用失败 | 公众号每日群发 1 次，`mass_send_left` 为 0 时只能建草稿——所以示例里根本不调群发 |

## 边界

- 只用本人浏览器里的登录态（cookie），不存密码、不走第三方中转。
- 接口是网页内部接口，不是开放平台 API；结构变了就按「采集为空 / 未登录」处理，不写脏数据。
