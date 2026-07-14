# cli.im 文本美化二维码请求链

## 结论

草料网页的普通文本二维码和美化二维码不是同一个单请求接口：

1. `/Apis/QrCode/saveStatic` 创建匿名静态码，返回二维码 ID 和默认标签样式；
2. 如果有本地 Logo，`upload-api.cli.im/upload?kid=cliim` 上传文件并返回公开 URL；
3. `qr.api.cli.im/create/paintLabelByParam` 将二维码配置、Logo、码点、码眼和标签背景合成为最终 PNG；
4. `qrdetector-api.cli.im/v1/detect_binary` 可回读 PNG，验证编码内容。

网页上的普通二维码预览最初表现为 `data:image/png;base64,...`，但进入美化编辑器时会建立一个匿名静态码记录，并围绕 `mh_str` 和标签样式参数重新绘制。

## 1. 初始化会话

```http
GET https://cli.im/
```

保留服务端下发的 `PHPSESSID` 等 Cookie。当前实测匿名请求可用，不要求登录 Cookie。

## 2. 创建静态文本码

```http
POST https://cli.im/Apis/QrCode/saveStatic
Content-Type: application/x-www-form-urlencoded
Origin: https://cli.im
Referer: https://cli.im/text/other
```

关键表单字段：

```text
info=<编码内容>
content=<编码内容>
level=Q
size=400
margin=2
version=3
codetype=qr
type=text
is_anonymous=1
static_create_from=10003
code_small_type=text
qrcodeconfig[size]=400
base64=
codeimg=1
is_pre_create=1
```

关键响应字段：

```json
{
  "status": 1,
  "data": {
    "qrid": "1121629085",
    "code_type": 3,
    "qr_web_url": "原始编码内容",
    "style_tpl": {
      "tpl_id": 171,
      "style_tplid": 215,
      "style_size_id": 85,
      "from_case_type": 1,
      "custom_tpl_param_config": "{...}",
      "variable_ass_field": "{...}"
    }
  }
}
```

注意：`qrid` 有时是 JSON 数字，有时是 JSON 字符串，客户端必须兼容两种类型。

## 3. 上传本地 Logo

```http
POST https://upload-api.cli.im/upload?kid=cliim
Content-Type: multipart/form-data
Origin: https://cli.im
```

表单：

```text
Filedata=@logo.png
blacklist=1
```

响应：

```json
{
  "code": 200,
  "data": {
    "url": "https://ncstatic.clewm.net/free/.../logo.png",
    "info": [128, 128]
  }
}
```

将 `data.url` 写入下一步的 `mh_str.logourl`。

## 4. 美化配置 `mh_str`

示例：

```json
{
  "logourl": "https://ncstatic.clewm.net/free/.../logo.png",
  "logoshape": "rect",
  "logo_pos": "0",
  "logo_shadow": 0,
  "logosize": 90,
  "logoh": "auto",
  "data": "",
  "level": "H",
  "transparent": 0,
  "bgcolor": "#ffffff",
  "forecolor": "#1E4299",
  "blockpixel": 12,
  "marginblock": "4",
  "size": "400",
  "version": "3",
  "foretype": 1,
  "forecolor2": "",
  "body_type": "17",
  "qrcode_eyes": "circle_circle",
  "outcolor": "#4A5E60",
  "incolor": "#8A2930",
  "eye_use_fore": 0
}
```

关键字段：

- `forecolor`：码点颜色；
- `body_type`：码点形态；
- `qrcode_eyes`：码眼形状；
- `outcolor`：码外眼颜色；
- `incolor`：码内眼颜色；
- `eye_use_fore=1`：码眼跟随 `forecolor`；
- `eye_use_fore=0`：使用 `outcolor` 和 `incolor`；
- `marginblock`：码边距；
- `level`：容错率；
- `version`：二维码版本；
- `logourl`：Logo URL。

## 5. 背景色的真实字段

这是协议复现中最容易写错的部分。

在基础标签样式下，仅修改：

```json
{"bgcolor":"#CED9CD"}
```

请求虽然返回成功，但最终图片背景仍可能是白色。网页实际把“码背景色”作为标签主题参数提交：

```text
theme_bg={"color":"#CED9CD"}
```

因此当前实现保持 `mh_str.bgcolor=#ffffff`，把最终可见背景色写入 `theme_bg.color`。像素直方图已经验证最终 PNG 中存在指定的 `#CED9CD` 背景。

## 6. 绘制最终 PNG

```http
POST https://qr.api.cli.im/create/paintLabelByParam
Content-Type: application/x-www-form-urlencoded
Origin: https://cli.im
Referer: https://cli.im/text/other
```

字段：

```text
code_type=3
custom_tpl_param_config=<saveStatic 返回值>
from_case_type=1
label_tplid=171
mh_str=<上面的 JSON>
need_qr_img=1
need_qr_version=1
need_tpl_tip=1
return_file=base64
style_tplid=215
theme_bg={"color":"#CED9CD"}
tpl_fields_num=0
variable_ass_field=<saveStatic 返回值>
web_url=<编码内容>
```

响应：

```json
{
  "code": 1,
  "msg": "ok",
  "data": {
    "qr_version": 3,
    "img_base64": "data:image/png;base64,..."
  }
}
```

解码 `img_base64` 即得到最终 PNG。

## 7. 回读验证

```http
POST https://qrdetector-api.cli.im/v1/detect_binary
Content-Type: application/x-www-form-urlencoded
```

字段：

```text
image_data=data:image/png;base64,...
remove_background=0
```

响应：

```json
{
  "status": 1,
  "message": "ok",
  "data": {
    "qrcode_content": "原始编码内容"
  }
}
```

CLI 默认比较 `qrcode_content` 与 `-text`；不一致则退出失败。
