## Image Manifest V2, Schema 2

这第二个模式版本有两个主要目标。
+ 第一个目标是通过引用针对镜像的平台特定版本的镜像清单的“fat manifest”来实现多体系结构镜像。
+ 第二个目标是通过支持镜像模型，其中可以对镜像的配置进行哈希处理以生成镜像的 ID，从而将 Docker 引擎朝向内容可寻址的镜像迈进。

### Media Types

The following media types are used by the manifest formats described here, and the resources they reference:

`application/vnd.docker.distribution.manifest.v1+json`: schema1 (existing manifest format)
`application/vnd.docker.distribution.manifest.v2+json`: New image manifest format (schemaVersion = 2)
`application/vnd.docker.distribution.manifest.list.v2+json`: Manifest list, aka "fat manifest"
`application/vnd.docker.container.image.v1+json`: Container config JSON
`application/vnd.docker.image.rootfs.diff.tar.gzip`: "Layer", as a gzipped tar
`application/vnd.docker.image.rootfs.foreign.diff.tar.gzip`: "Layer", as a gzipped tar that should never be pushed
`application/vnd.docker.plugin.v1+json`: Plugin config JSON

### Manifest List

manifest list 是指向一个或多个平台的特定镜像清单的“fat manifest”。它的使用是可选的，并且相对较少的镜像将使用这些清单之一。客户端将根据HTTP响应中返回的Content-Type来区分 manifest list 和镜像清单。

### Manifest List Field Descriptions

1. `schemaVersion` int

    此字段将镜像清单模式版本指定为整数。此模式使用版本2。

2. `mediaType` string

    manifest list 的 MIME 类型。应将其设置为`application/vnd.docker.distribution.manifest.list.v2+json`。

3. `manifests` arry
   
    manifests 字段包含特定平台的 manifest list。manifest list 中对象的字段包括：

    3.1. `mediaType` string

    所引用对象的 MIME 类型。通常为 `application/vnd.docker.distribution.manifest.v2+json`，但如果 manifest list 引用遗留的schema-1清单，则也可以为 `application/vnd.docker.distribution.manifest.v1+json`。

    3.2 size int

    对象的大小（以字节为单位）。此字段存在，以便客户端在验证之前知道内容的预期大小。如果检索到的内容的长度与指定的长度不匹配，则不应信任该内容。

    3.3 `digest` string

    内容的摘要，由[Registry V2 HTTP API规范定义](https://docs.docker.com/registry/spec/api/#digest-parameter)。

    3.4 `platform` object

    platform对象描述了映像在其中运行的平台。有效操作系统和架构值的完整列表在[Go语言文档中列出，可以参考\$GOOS和\$GOARCH的定义](https://golang.org/doc/install/source#environment)。
        
    + `architecture` string

        architecture字段指定CPU架构，例如amd64或ppc64le。
    
    + `os` string
  
        os字段指定操作系统，例如linux或windows。

    + `os.version` string

        可选的os.version字段指定操作系统版本，例如10.0.10586。
    
    + `os.features` array

        可选的os.features字段指定一个字符串数组，每个字符串列出一个所需的操作系统特性（例如，在Windows上是win32k）。

    + `variant` string

        可选的variant字段指定CPU的变种，例如armv6l用于指定ARM CPU的特定变种。

    + `features` arrary

        可选的features字段指定一个字符串数组，每个字符串列出一个所需的CPU特性（例如sse4或aes）。

### Example Manifest List

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
  "manifests": [
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 7143,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux",
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 7682,
      "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270",
      "platform": {
        "architecture": "amd64",
        "os": "linux",
        "features": [
          "sse4"
        ]
      }
    }
  ]
}
```

### Image Manifest

Image Manifest 提供了容器镜像的配置和一组 layers。它是 schema-1 manifest 的直接替代品。

### Image Manifest Field Descriptions

1. `schemaVersion` int
   
    该字段以整数形式指定 image manifest 模式版本。该模式使用版本2。

2. `mediaType` string

    清单的MIME类型。应设置为 `application/vnd.docker.distribution.manifest.v2+json`。

3. `config` object

    config字段通过摘要引用容器的配置对象。这个配置项是一个 JSON 块，运行时使用它来设置容器。这个新模式使用了这个配置的微调版本，以在守护程序端实现镜像内容可寻址性。

    3.1 `mediaType` string

    清单的MIME类型。应设置为 `application/vnd.docker.distribution.manifest.v2+json`。

    3.2 `config` object

    config 字段引用容器的配置对象，通过摘要。这个配置项是一个 JSON 块，运行时使用它来设置容器。这个新模式使用了这个配置的微调版本，以在守护程序端实现镜像内容可寻址性。

    + `mediaType` string

        引用对象的MIME类型。通常应设置为 `application/vnd.docker.container.image.v1+json`。
    
    + `size` int

        对象的字节大小。此字段存在是为了使客户端在验证之前能够得到内容的期望大小。如果检索到的内容长度与指定的长度不匹配，则不应信任内容。

    + `digest` string

        内容的摘要，由[Registry V2 HTTP API规范定义](https://docs.docker.com/registry/spec/api/#digest-parameter)。
    
    3.3 `layers` array

    图层列表按照从基础镜像开始的顺序排序（与schema1相反）。

    + `mediaType` string

    引用对象的MIME类型。通常应设置为 `application/vnd.docker.image.rootfs.diff.tar.gzip`。类型为 `application/vnd.docker.image.rootfs.foreign.diff.tar.gzip` 的图层可以从远程位置获取，但不应推送。

    + `size` int

    对象的字节大小。此字段存在是为了使客户端在验证之前能够得到内容的期望大小。如果检索到的内容长度与指定的长度不匹配，则不应信任内容。

    + `digest` string

    内容的摘要，由 [Registry V2 HTTP API规范定义](https://docs.docker.com/registry/spec/api/#digest-parameter)。

    + `urls` arrary

    提供了一个从中获取内容的URL列表。内容必须根据摘要和大小进行验证。此字段是可选的，不常见。

### Example Image Manifest 

```json
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
    "config": {
        "mediaType": "application/vnd.docker.container.image.v1+json",
        "size": 7023,
        "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
    },
    "layers": [
        {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 32654,
            "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f"
        },
        {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 16724,
            "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b"
        },
        {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 73109,
            "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184802ad867736"
        }
    ]
}
```

### Backward compatibility

Registry 将继续接受旧格式和新格式的清单上传。

在推送镜像时，支持新清单格式的客户端应首先构建新格式的清单。如果上传此清单失败，可能是因为 Registry 仅支持旧格式，客户端可以退回到上传旧格式的清单。

在拉取镜像时，客户端通过在请求 manifests 端点时发送 Accept 头中的 application/vnd.docker.distribution.manifest.v2+json 和 application/vnd.docker.distribution.manifest.list.v2+json 媒体类型来表示对清单格式的新版本的支持。更新后的客户端应检查 Content-Type 头，以查看从端点返回的清单是旧格式还是新格式的镜像清单或清单列表。

如果被请求的清单使用新格式，并且在Accept头中没有适当的媒体类型，Registry 将假定客户端无法按原样处理清单，并即时将其重写为旧格式。如果否则要返回的对象是清单列表，Registry 将查找 amd64 平台和 linux 操作系统的适当清单，如有必要将该清单重写为旧格式，并将结果返回给客户端。如果在清单列表中找不到合适的清单，Registry 将返回404错误。

将清单重写为旧格式的挑战之一是，旧格式涉及清单中每个图层的镜像配置，而新格式仅提供一个镜像配置。为解决此问题，Registry 将为除顶层以外的所有图层创建合成镜像配置。这些镜像配置本身不会生成可运行的镜像，只会以兼容的方式填充父链。这些合成配置中的 ID 将从其各自的 blob 的哈希派生。Registry 将使用与 Docker 1.10 在创建传统清单以推送到不支持新格式的 Registry 时相同的方案来创建这些配置和它们的 ID。