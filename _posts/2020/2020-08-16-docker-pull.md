---
layout: post
title: "docker 是怎么下载镜像的？"
date: "2020-08-17 20:40"
category: docker
tags: docker
author: lework
---
* content
{:toc}


我们在执行`docker pull images` 的时候，其实是由`docker` 这个可执行文件向 [docker registry server](https://docs.docker.com/registry/) 去下载镜像对应的数据，那它们之间是怎样通讯的呢？

用户想要拉出或下载镜像。涉及的步骤如下：

1. 获取 registry  v2 api 的信息。
   1. 需要认证时，则需要进行身份验证并获取token。
2. 获取镜像的`manifests`。
3. 获取镜像的`layers`信息。
4. 依次下载layers层数据。




下面我们将使用`docker` 来拉取主流的镜像仓库中的镜像，并截取其通信的https数据包。

## 下载 Docker Hub仓库镜像

```bash
docker pull alpine
```

### 1. 获取 registry  v2 api 信息

```bash
GET https://registry-1.docker.io/v2/ HTTP/1.1
Host: registry-1.docker.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: gzip
Connection: close



HTTP/1.1 401 Unauthorized
Content-Type: application/json
Docker-Distribution-Api-Version: registry/2.0
Www-Authenticate: Bearer realm="https://auth.docker.io/token",service="registry.docker.io"
Date: Thu, 20 Aug 2020 06:17:41 GMT
Content-Length: 87
Connection: close
Strict-Transport-Security: max-age=31536000

{"errors":[{"code":"UNAUTHORIZED","message":"authentication required","detail":null}]}

```

返回`200`状态码：表示 registry  中有v2 api注册信息，客户端可以操作其他v2 api。

返回`401`状态码：表示需要认证，在hearder里有`Www-Authenticate`来详细说明如何对该注册中心进行身份验证

返回`404`状态码：表示 registry  中没有v2 api注册信息或者其他原因。

### 2. 获取 token

需要认证时，下一步就是获取token了。

```bash
GET https://auth.docker.io/token?scope=repository%3Alibrary%2Falpine%3Apull&service=registry.docker.io HTTP/1.1
Host: auth.docker.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: gzip
Connection: close



HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 20 Aug 2020 06:17:42 GMT
Connection: close
Strict-Transport-Security: max-age=31536000
Content-Length: 4179

{"token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvYWxwaW5lIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiYXVkIjoicmVnaXN0cnkuZG9ja2VyLmlvIiwiZXhwIjoxNTk3OTA0NTYyLCJpYXQiOjE1OTc5MDQyNjIsImlzcyI6ImF1dGguZG9ja2VyLmlvIiwianRpIjoiYktISHFCRkRjRnFvb3phc3FmVXQiLCJuYmYiOjE1OTc5MDM5NjIsInN1YiI6IiJ9.Eoak7qmncDqGRbklBwy2RNXvRoIkj4DUEr_p3diQqg8PsuDRLnB6J02X9_hioI6g2s_rAb8U_AThVsnOIu_nKV7J9LEoFRd08z2h1578vvGKfQufv0yiWLhchZThujqg2AKGzExGsZa2ZJclJBF316rSX88wZyH9BMrcN5IV9paJjHktHqnv3Beev2-59PDmSFKseJn6AHoQmkUvvNcfd25qXmlMsjv9P61-55pC065MfW9XbSbpm_hk9UIqQKjmoJ01lGLlxskreei42TSlJbnLXsNNjNMHGYc3lTpV8hRaUL2kn2nHVVTIlsm_dv9XE1YJtwKL6wrsHMeatA4Mgg","access_token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvYWxwaW5lIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiYXVkIjoicmVnaXN0cnkuZG9ja2VyLmlvIiwiZXhwIjoxNTk3OTA0NTYyLCJpYXQiOjE1OTc5MDQyNjIsImlzcyI6ImF1dGguZG9ja2VyLmlvIiwianRpIjoiYktISHFCRkRjRnFvb3phc3FmVXQiLCJuYmYiOjE1OTc5MDM5NjIsInN1YiI6IiJ9.Eoak7qmncDqGRbklBwy2RNXvRoIkj4DUEr_p3diQqg8PsuDRLnB6J02X9_hioI6g2s_rAb8U_AThVsnOIu_nKV7J9LEoFRd08z2h1578vvGKfQufv0yiWLhchZThujqg2AKGzExGsZa2ZJclJBF316rSX88wZyH9BMrcN5IV9paJjHktHqnv3Beev2-59PDmSFKseJn6AHoQmkUvvNcfd25qXmlMsjv9P61-55pC065MfW9XbSbpm_hk9UIqQKjmoJ01lGLlxskreei42TSlJbnLXsNNjNMHGYc3lTpV8hRaUL2kn2nHVVTIlsm_dv9XE1YJtwKL6wrsHMeatA4Mgg","expires_in":300,"issued_at":"2020-08-20T06:17:42.575597053Z"}

```

认证成功后，会返回带有`token` 的json数据体。



### 3. 获取镜像标签的清单信息

```bash
GET https://registry-1.docker.io/v2/library/alpine/manifests/latest HTTP/1.1
Host: registry-1.docker.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept: application/json
Accept: application/vnd.docker.distribution.manifest.v2+json
Accept: application/vnd.docker.distribution.manifest.list.v2+json
Accept: application/vnd.oci.image.index.v1+json
Accept: application/vnd.oci.image.manifest.v1+json
Accept: application/vnd.docker.distribution.manifest.v1+prettyjws
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvYWxwaW5lIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiYXVkIjoicmVnaXN0cnkuZG9ja2VyLmlvIiwiZXhwIjoxNTk3OTA0NTYyLCJpYXQiOjE1OTc5MDQyNjIsImlzcyI6ImF1dGguZG9ja2VyLmlvIiwianRpIjoiYktISHFCRkRjRnFvb3phc3FmVXQiLCJuYmYiOjE1OTc5MDM5NjIsInN1YiI6IiJ9.Eoak7qmncDqGRbklBwy2RNXvRoIkj4DUEr_p3diQqg8PsuDRLnB6J02X9_hioI6g2s_rAb8U_AThVsnOIu_nKV7J9LEoFRd08z2h1578vvGKfQufv0yiWLhchZThujqg2AKGzExGsZa2ZJclJBF316rSX88wZyH9BMrcN5IV9paJjHktHqnv3Beev2-59PDmSFKseJn6AHoQmkUvvNcfd25qXmlMsjv9P61-55pC065MfW9XbSbpm_hk9UIqQKjmoJ01lGLlxskreei42TSlJbnLXsNNjNMHGYc3lTpV8hRaUL2kn2nHVVTIlsm_dv9XE1YJtwKL6wrsHMeatA4Mgg
Accept-Encoding: gzip
Connection: close



HTTP/1.1 200 OK
Content-Length: 1638
Content-Type: application/vnd.docker.distribution.manifest.list.v2+json
Docker-Content-Digest: sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321
Docker-Distribution-Api-Version: registry/2.0
Etag: "sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321"
Date: Thu, 20 Aug 2020 06:17:43 GMT
Connection: close
Strict-Transport-Security: max-age=31536000

{"manifests":[{"digest":"sha256:a15790640a6690aa1730c38cf0a440e2aa44aaca9b0e8931a9f2b0d7cc90fd65","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"amd64","os":"linux"},"size":528},{"digest":"sha256:71465c7d45a086a2181ce33bb47f7eaef5c233eace65704da0c5e5454a79cee5","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"arm","os":"linux","variant":"v6"},"size":528},{"digest":"sha256:c929c5ca1d3f793bfdd2c6d6d9210e2530f1184c0f488f514f1bb8080bb1e82b","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"arm","os":"linux","variant":"v7"},"size":528},{"digest":"sha256:3b3f647d2d99cac772ed64c4791e5d9b750dd5fe0b25db653ec4976f7b72837c","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"arm64","os":"linux","variant":"v8"},"size":528},{"digest":"sha256:90baa0922fe90624b05cb5766fa5da4e337921656c2f8e2b13bd3c052a0baac1","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"386","os":"linux"},"size":528},{"digest":"sha256:5d950b30f229f0c53dd7dd7ed6e0e33e89d927b16b8149cc68f59bbe99219cc1","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"ppc64le","os":"linux"},"size":528},{"digest":"sha256:a5426f084c755f4d6c1d1562a2d456aa574a24a61706f6806415627360c06ac0","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"s390x","os":"linux"},"size":528}],"mediaType":"application\/vnd.docker.distribution.manifest.list.v2+json","schemaVersion":2}
```

如果有多个平台manifests，返回的是镜像标签的所有manifest信息列表，如果只有1个平台，则返回这个平台manifest的分层数据信息。

### 4. 获取适合宿主机的清单信息

```bash
GET https://registry-1.docker.io/v2/library/alpine/manifests/sha256:a15790640a6690aa1730c38cf0a440e2aa44aaca9b0e8931a9f2b0d7cc90fd65 HTTP/1.1
Host: registry-1.docker.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept: application/vnd.docker.distribution.manifest.v1+prettyjws
Accept: application/json
Accept: application/vnd.docker.distribution.manifest.v2+json
Accept: application/vnd.docker.distribution.manifest.list.v2+json
Accept: application/vnd.oci.image.index.v1+json
Accept: application/vnd.oci.image.manifest.v1+json
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvYWxwaW5lIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiYXVkIjoicmVnaXN0cnkuZG9ja2VyLmlvIiwiZXhwIjoxNTk3OTA0NTYyLCJpYXQiOjE1OTc5MDQyNjIsImlzcyI6ImF1dGguZG9ja2VyLmlvIiwianRpIjoiYktISHFCRkRjRnFvb3phc3FmVXQiLCJuYmYiOjE1OTc5MDM5NjIsInN1YiI6IiJ9.Eoak7qmncDqGRbklBwy2RNXvRoIkj4DUEr_p3diQqg8PsuDRLnB6J02X9_hioI6g2s_rAb8U_AThVsnOIu_nKV7J9LEoFRd08z2h1578vvGKfQufv0yiWLhchZThujqg2AKGzExGsZa2ZJclJBF316rSX88wZyH9BMrcN5IV9paJjHktHqnv3Beev2-59PDmSFKseJn6AHoQmkUvvNcfd25qXmlMsjv9P61-55pC065MfW9XbSbpm_hk9UIqQKjmoJ01lGLlxskreei42TSlJbnLXsNNjNMHGYc3lTpV8hRaUL2kn2nHVVTIlsm_dv9XE1YJtwKL6wrsHMeatA4Mgg
Accept-Encoding: gzip
Connection: close


HTTP/1.1 200 OK
Content-Length: 528
Content-Type: application/vnd.docker.distribution.manifest.v2+json
Docker-Content-Digest: sha256:a15790640a6690aa1730c38cf0a440e2aa44aaca9b0e8931a9f2b0d7cc90fd65
Docker-Distribution-Api-Version: registry/2.0
Etag: "sha256:a15790640a6690aa1730c38cf0a440e2aa44aaca9b0e8931a9f2b0d7cc90fd65"
Date: Thu, 20 Aug 2020 06:17:44 GMT
Connection: close
Strict-Transport-Security: max-age=31536000

{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1509,
      "digest": "sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 2797541,
         "digest": "sha256:df20fa9351a15782c64e6dddb2d4a6f50bf6d3688060a34c4014b0d9a752eb4c"
      }
   ]
}

```

返回镜像标签的所有分层信息



### 5. 依次拉取layers层的数据

```bash
GET https://registry-1.docker.io/v2/library/alpine/blobs/sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e HTTP/1.1
Host: registry-1.docker.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvYWxwaW5lIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiYXVkIjoicmVnaXN0cnkuZG9ja2VyLmlvIiwiZXhwIjoxNTk3OTA0NTYyLCJpYXQiOjE1OTc5MDQyNjIsImlzcyI6ImF1dGguZG9ja2VyLmlvIiwianRpIjoiYktISHFCRkRjRnFvb3phc3FmVXQiLCJuYmYiOjE1OTc5MDM5NjIsInN1YiI6IiJ9.Eoak7qmncDqGRbklBwy2RNXvRoIkj4DUEr_p3diQqg8PsuDRLnB6J02X9_hioI6g2s_rAb8U_AThVsnOIu_nKV7J9LEoFRd08z2h1578vvGKfQufv0yiWLhchZThujqg2AKGzExGsZa2ZJclJBF316rSX88wZyH9BMrcN5IV9paJjHktHqnv3Beev2-59PDmSFKseJn6AHoQmkUvvNcfd25qXmlMsjv9P61-55pC065MfW9XbSbpm_hk9UIqQKjmoJ01lGLlxskreei42TSlJbnLXsNNjNMHGYc3lTpV8hRaUL2kn2nHVVTIlsm_dv9XE1YJtwKL6wrsHMeatA4Mgg
Connection: close



HTTP/1.1 307 Temporary Redirect
Content-Type: application/octet-stream
Docker-Distribution-Api-Version: registry/2.0
Location: https://production.cloudflare.docker.com/registry-v2/docker/registry/v2/blobs/sha256/a2/a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e/data?verify=1597907265-RDidtXVhlEQSkIqho9mv6cc5dtQ%3D
Date: Thu, 20 Aug 2020 06:17:45 GMT
Content-Length: 0
Connection: close
Strict-Transport-Security: max-age=31536000

```

这里返回了`307`跳转

### 6. 下载数据

```bash
GET https://production.cloudflare.docker.com/registry-v2/docker/registry/v2/blobs/sha256/a2/a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e/data?verify=1597907265-RDidtXVhlEQSkIqho9mv6cc5dtQ%3D HTTP/1.1
Host: production.cloudflare.docker.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Referer: https://registry-1.docker.io/v2/library/alpine/blobs/sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e
Connection: close


HTTP/1.1 200 OK
Date: Thu, 20 Aug 2020 06:17:46 GMT
Content-Type: application/octet-stream
Content-Length: 1509
Connection: close
Set-Cookie: __cfduid=d04ebe20bc6d7e2a107b72a8235eeb2d01597904266; expires=Sat, 19-Sep-20 06:17:46 GMT; path=/; domain=.production.cloudflare.docker.com; HttpOnly; SameSite=Lax; Secure
CF-Ray: 5c59fe3f5d61258d-HKG
Accept-Ranges: bytes
Age: 2277324
Cache-Control: public, max-age=14400
ETag: "b7b8406ca68cc818cb35f053f93caf07"
Expires: Thu, 20 Aug 2020 10:17:46 GMT
Last-Modified: Fri, 29 May 2020 21:20:08 GMT
Vary: Accept-Encoding
CF-Cache-Status: HIT
cf-request-id: 04ac1d3b980000258d563b0200000001
Expect-CT: max-age=604800, report-uri="https://report-uri.cloudflare.com/cdn-cgi/beacon/expect-ct"
x-amz-id-2: whm9KH4dbHtjtDWcTwlhr82h55cWfMdT63uBSHRzglSO0H0BZlZIitUzROmo+3i3SyCLCYltzuI=
x-amz-request-id: 02DD1574BF2664F6
x-amz-version-id: x61eC1buF1kfpFnSwidmv5_7gxWle14z
Server: cloudflare

{"architecture":"amd64","config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr
一大波数据
```



## 以代理的方式下载镜像

```
docker pull alpine
```

### 1. 获取 registry  v2 api 信息

```
GET https://2h3po24q.mirror.aliyuncs.com/v2/ HTTP/1.1
Host: 2h3po24q.mirror.aliyuncs.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: gzip
Connection: close


HTTP/1.1 200 OK
Content-Length: 2
Content-Type: application/json; charset=utf-8
Docker-Distribution-Api-Version: registry/2.0
Date: Thu, 20 Aug 2020 06:14:18 GMT
Connection: close

{}
```

这里不需要认证



### 2. 获取镜像标签的清单信息

```
GET https://2h3po24q.mirror.aliyuncs.com/v2/library/alpine/manifests/latest HTTP/1.1
Host: 2h3po24q.mirror.aliyuncs.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept: application/vnd.docker.distribution.manifest.v2+json
Accept: application/vnd.docker.distribution.manifest.list.v2+json
Accept: application/vnd.oci.image.index.v1+json
Accept: application/vnd.oci.image.manifest.v1+json
Accept: application/vnd.docker.distribution.manifest.v1+prettyjws
Accept: application/json
Accept-Encoding: gzip
Connection: close


HTTP/1.1 200 OK
Content-Length: 1638
Content-Type: application/vnd.docker.distribution.manifest.list.v2+json
Docker-Content-Digest: sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321
Docker-Distribution-Api-Version: registry/2.0
Etag: "sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321"
Date: Thu, 20 Aug 2020 06:14:33 GMT
Connection: close

{"manifests":[{"digest":"sha256:a15790640a6690aa1730c38cf0a440e2aa44aaca9b0e8931a9f2b0d7cc90fd65","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"amd64","os":"linux"},"size":528},{"digest":"sha256:71465c7d45a086a2181ce33bb47f7eaef5c233eace65704da0c5e5454a79cee5","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"arm","os":"linux","variant":"v6"},"size":528},{"digest":"sha256:c929c5ca1d3f793bfdd2c6d6d9210e2530f1184c0f488f514f1bb8080bb1e82b","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"arm","os":"linux","variant":"v7"},"size":528},{"digest":"sha256:3b3f647d2d99cac772ed64c4791e5d9b750dd5fe0b25db653ec4976f7b72837c","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"arm64","os":"linux","variant":"v8"},"size":528},{"digest":"sha256:90baa0922fe90624b05cb5766fa5da4e337921656c2f8e2b13bd3c052a0baac1","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"386","os":"linux"},"size":528},{"digest":"sha256:5d950b30f229f0c53dd7dd7ed6e0e33e89d927b16b8149cc68f59bbe99219cc1","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"ppc64le","os":"linux"},"size":528},{"digest":"sha256:a5426f084c755f4d6c1d1562a2d456aa574a24a61706f6806415627360c06ac0","mediaType":"application\/vnd.docker.distribution.manifest.v2+json","platform":{"architecture":"s390x","os":"linux"},"size":528}],"mediaType":"application\/vnd.docker.distribution.manifest.list.v2+json","schemaVersion":2}
```

### 3. 获取适合宿主机的清单信息

```
GET https://2h3po24q.mirror.aliyuncs.com/v2/library/alpine/manifests/sha256:a15790640a6690aa1730c38cf0a440e2aa44aaca9b0e8931a9f2b0d7cc90fd65 HTTP/1.1
Host: 2h3po24q.mirror.aliyuncs.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept: application/vnd.docker.distribution.manifest.list.v2+json
Accept: application/vnd.oci.image.index.v1+json
Accept: application/vnd.oci.image.manifest.v1+json
Accept: application/vnd.docker.distribution.manifest.v1+prettyjws
Accept: application/json
Accept: application/vnd.docker.distribution.manifest.v2+json
Accept-Encoding: gzip
Connection: close



HTTP/1.1 200 OK
Content-Length: 528
Content-Type: application/vnd.docker.distribution.manifest.v2+json
Docker-Content-Digest: sha256:a15790640a6690aa1730c38cf0a440e2aa44aaca9b0e8931a9f2b0d7cc90fd65
Docker-Distribution-Api-Version: registry/2.0
Etag: "sha256:a15790640a6690aa1730c38cf0a440e2aa44aaca9b0e8931a9f2b0d7cc90fd65"
Date: Thu, 20 Aug 2020 06:14:33 GMT
Connection: close

{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1509,
      "digest": "sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 2797541,
         "digest": "sha256:df20fa9351a15782c64e6dddb2d4a6f50bf6d3688060a34c4014b0d9a752eb4c"
      }
   ]
}
```

### 4. 依次拉取layers层的数据

```
GET https://2h3po24q.mirror.aliyuncs.com/v2/library/alpine/blobs/sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e HTTP/1.1
Host: 2h3po24q.mirror.aliyuncs.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Connection: close



HTTP/1.1 307 Temporary Redirect
Content-Type: application/octet-stream
Docker-Distribution-Api-Version: registry/2.0
Location: https://acs-cn-hangzhou-mirror.oss-cn-hangzhou.aliyuncs.com/docker/registry/v2/blobs/sha256/a2/a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e/data?Expires=1597905273&OSSAccessKeyId=LTAI4G6zf8H9RhDWxXfrrhkr&Signature=%2FNOGZkZMkQOgkhYcaRm%2B1e5JX7w%3D
Date: Thu, 20 Aug 2020 06:14:33 GMT
Content-Length: 0
Connection: close
```

### 5. 下载数据
```
GET https://acs-cn-hangzhou-mirror.oss-cn-hangzhou.aliyuncs.com/docker/registry/v2/blobs/sha256/a2/a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e/data?Expires=1597905273&OSSAccessKeyId=LTAI4G6zf8H9RhDWxXfrrhkr&Signature=%2FNOGZkZMkQOgkhYcaRm%2B1e5JX7w%3D HTTP/1.1
Host: acs-cn-hangzhou-mirror.oss-cn-hangzhou.aliyuncs.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Referer: https://2h3po24q.mirror.aliyuncs.com/v2/library/alpine/blobs/sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e
Connection: close

HTTP/1.1 200 OK
Server: AliyunOSS
Date: Thu, 20 Aug 2020 06:14:34 GMT
Content-Type: application/octet-stream
Content-Length: 1509
Connection: close
x-oss-request-id: 5F3E14CAC6CA7E31381A44B0
Accept-Ranges: bytes
ETag: "B7B8406CA68CC818CB35F053F93CAF07"
Last-Modified: Fri, 29 May 2020 21:41:36 GMT
x-oss-object-type: Normal
x-oss-hash-crc64ecma: 6766004739995481995
x-oss-storage-class: Standard
Content-MD5: t7hAbKaMyBjLNfBT+TyvBw==
x-oss-server-time: 2

{"architecture":"amd64","config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr
一大堆数据
```





## 下载 gcr 仓库镜像

```
docker pull gcr.io/google-containers/kube-controller-manager:v1.16.10
```

### 1. 获取 registry  v2 api 信息
```
GET https://gcr.io/v2/ HTTP/1.1
Host: gcr.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: gzip
Connection: close


HTTP/1.1 401 Unauthorized
Docker-Distribution-API-Version: registry/2.0
WWW-Authenticate: Bearer realm="https://gcr.io/v2/token",service="gcr.io"
Content-Type: application/json
Date: Thu, 20 Aug 2020 03:38:42 GMT
Server: Docker Registry
Cache-Control: private
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Connection: close
Content-Length: 69

{"errors":[{"code":"UNAUTHORIZED","message":"Unauthorized access."}]}
```

### 2. 获取 token

```
GET https://gcr.io/v2/token?scope=repository%3Agoogle-containers%2Fkube-controller-manager%3Apull&service=gcr.io HTTP/1.1
Host: gcr.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: gzip
Connection: close


HTTP/1.1 200 OK
Docker-Distribution-API-Version: registry/2.0
Content-Type: application/json
Date: Thu, 20 Aug 2020 03:38:43 GMT
Server: Docker Registry
Cache-Control: private
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Connection: close
Content-Length: 500

{"expires_in":43200,"issued_at":"2020-08-19T20:38:43.00565746-07:00","token":"AJAD5v0sN9g4glsFyOyeRUycw1lLDUpkrSkVn+8YbmfHPUlCS4+hSi9DSa+uPAzpqFV/VLFREmvXggl4y08aPirghFnnBNcanwVhbbs+frhLGouymult0lA/JUoe4QSj/hDf4EU503TFdwMSax6K75C/QCXuDILNCSXYOXdwwhroZwraFDxlc1INXtps4i+kzSFNBTEoUZmqUE60YAcv3ZMlLgKOMyVFByUdaV9rjZ925xHgQYOt5/2TLdvL1qHD3f97zWwd1fSEBFNquqz7DguB2IgTaw8+sEE01kBDmv4X9giZx/eBjBekrIbJIiiDUoRe5wSEuvyk3ocPh99/Fqd9iYMaX7/lVBwJXoxfLYSxHc/XzlsmDoplkKvScVL7Q92Ko7JeJzCCXcMR40i32bJzj5mHQBUPoVo="}
```

### 3. 获取镜像标签的清单信息

```
GET https://gcr.io/v2/google-containers/kube-controller-manager/manifests/v1.16.10 HTTP/1.1
Host: gcr.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept: application/vnd.docker.distribution.manifest.v1+prettyjws
Accept: application/json
Accept: application/vnd.docker.distribution.manifest.v2+json
Accept: application/vnd.docker.distribution.manifest.list.v2+json
Accept: application/vnd.oci.image.index.v1+json
Accept: application/vnd.oci.image.manifest.v1+json
Authorization: Bearer AJAD5v0sN9g4glsFyOyeRUycw1lLDUpkrSkVn+8YbmfHPUlCS4+hSi9DSa+uPAzpqFV/VLFREmvXggl4y08aPirghFnnBNcanwVhbbs+frhLGouymult0lA/JUoe4QSj/hDf4EU503TFdwMSax6K75C/QCXuDILNCSXYOXdwwhroZwraFDxlc1INXtps4i+kzSFNBTEoUZmqUE60YAcv3ZMlLgKOMyVFByUdaV9rjZ925xHgQYOt5/2TLdvL1qHD3f97zWwd1fSEBFNquqz7DguB2IgTaw8+sEE01kBDmv4X9giZx/eBjBekrIbJIiiDUoRe5wSEuvyk3ocPh99/Fqd9iYMaX7/lVBwJXoxfLYSxHc/XzlsmDoplkKvScVL7Q92Ko7JeJzCCXcMR40i32bJzj5mHQBUPoVo=
Accept-Encoding: gzip
Connection: close


HTTP/1.1 200 OK
Docker-Distribution-API-Version: registry/2.0
Content-Type: application/vnd.docker.distribution.manifest.list.v2+json
Content-Length: 1665
Docker-Content-Digest: sha256:2f162989247c7b6f3df63f98e139fec04058cd414063312564ced2e0500a3086
Date: Thu, 20 Aug 2020 03:38:44 GMT
Server: Docker Registry
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Connection: close

{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 741,
         "digest": "sha256:7ff962b6cdeff0bf4ab18f93adbec51aae32f4987e7c97566460247284bbde76",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 741,
         "digest": "sha256:b9ce15b077a63e410aa9697877e340cd96c7436d8a42757c068005d6480368fb",
         "platform": {
            "architecture": "arm",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 741,
         "digest": "sha256:36bf963fa04eb9ce9111cf807bbc0336905aad032d9e38304ba1d5b4dee62028",
         "platform": {
            "architecture": "arm64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 741,
         "digest": "sha256:b1109ab4f37e796dd4ca5710b637328c39528fa51e32550581b2e85913835655",
         "platform": {
            "architecture": "ppc64le",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 741,
         "digest": "sha256:b728371a76e777fdfc958d88f26d47c6366708842a1c18b6a124f9a9ebc20ba7",
         "platform": {
            "architecture": "s390x",
            "os": "linux"
         }
      }
   ]
}
```

### 4. 获取适合宿主机的清单信息
```
GET https://gcr.io/v2/google-containers/kube-controller-manager/manifests/sha256:7ff962b6cdeff0bf4ab18f93adbec51aae32f4987e7c97566460247284bbde76 HTTP/1.1
Host: gcr.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept: application/vnd.docker.distribution.manifest.v2+json
Accept: application/vnd.docker.distribution.manifest.list.v2+json
Accept: application/vnd.oci.image.index.v1+json
Accept: application/vnd.oci.image.manifest.v1+json
Accept: application/vnd.docker.distribution.manifest.v1+prettyjws
Accept: application/json
Authorization: Bearer AJAD5v0sN9g4glsFyOyeRUycw1lLDUpkrSkVn+8YbmfHPUlCS4+hSi9DSa+uPAzpqFV/VLFREmvXggl4y08aPirghFnnBNcanwVhbbs+frhLGouymult0lA/JUoe4QSj/hDf4EU503TFdwMSax6K75C/QCXuDILNCSXYOXdwwhroZwraFDxlc1INXtps4i+kzSFNBTEoUZmqUE60YAcv3ZMlLgKOMyVFByUdaV9rjZ925xHgQYOt5/2TLdvL1qHD3f97zWwd1fSEBFNquqz7DguB2IgTaw8+sEE01kBDmv4X9giZx/eBjBekrIbJIiiDUoRe5wSEuvyk3ocPh99/Fqd9iYMaX7/lVBwJXoxfLYSxHc/XzlsmDoplkKvScVL7Q92Ko7JeJzCCXcMR40i32bJzj5mHQBUPoVo=
Accept-Encoding: gzip
Connection: close


HTTP/1.1 200 OK
Docker-Distribution-API-Version: registry/2.0
Content-Type: application/vnd.docker.distribution.manifest.v2+json
Docker-Content-Digest: sha256:7ff962b6cdeff0bf4ab18f93adbec51aae32f4987e7c97566460247284bbde76
Content-Length: 741
Date: Thu, 20 Aug 2020 03:38:45 GMT
Server: Docker Registry
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Connection: close

{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1786,
      "digest": "sha256:95b2e4f548f19fafcc7fbf683024ee9abc250394e22b0fb879d8d488099c0177"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 21082831,
         "digest": "sha256:83b4483280e5187b2801b449338d5755e5874ab80c44bf1ce615d258142e7c8b"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 27897052,
         "digest": "sha256:7cc17f1decf73673c3093bbbab0e5770ebd17b2201cbfd992e8284530467a97a"
      }
   ]
}
```
### 5. 依次拉取layers层的数据

```
GET https://gcr.io/v2/google-containers/kube-controller-manager/blobs/sha256:83b4483280e5187b2801b449338d5755e5874ab80c44bf1ce615d258142e7c8b HTTP/1.1
Host: gcr.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Authorization: Bearer AJAD5v0sN9g4glsFyOyeRUycw1lLDUpkrSkVn+8YbmfHPUlCS4+hSi9DSa+uPAzpqFV/VLFREmvXggl4y08aPirghFnnBNcanwVhbbs+frhLGouymult0lA/JUoe4QSj/hDf4EU503TFdwMSax6K75C/QCXuDILNCSXYOXdwwhroZwraFDxlc1INXtps4i+kzSFNBTEoUZmqUE60YAcv3ZMlLgKOMyVFByUdaV9rjZ925xHgQYOt5/2TLdvL1qHD3f97zWwd1fSEBFNquqz7DguB2IgTaw8+sEE01kBDmv4X9giZx/eBjBekrIbJIiiDUoRe5wSEuvyk3ocPh99/Fqd9iYMaX7/lVBwJXoxfLYSxHc/XzlsmDoplkKvScVL7Q92Ko7JeJzCCXcMR40i32bJzj5mHQBUPoVo=
Connection: close


HTTP/1.1 302 Found
Docker-Distribution-API-Version: registry/2.0
Location: https://storage.googleapis.com/artifacts.google-containers.appspot.com/containers/images/sha256:83b4483280e5187b2801b449338d5755e5874ab80c44bf1ce615d258142e7c8b
Content-Type: application/json
Date: Thu, 20 Aug 2020 03:38:46 GMT
Server: Docker Registry
Cache-Control: private
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Accept-Ranges: none
Vary: Accept-Encoding
Connection: close
Content-Length: 13

{"errors":[]}
```


```
GET https://gcr.io/v2/google-containers/kube-controller-manager/blobs/sha256:7cc17f1decf73673c3093bbbab0e5770ebd17b2201cbfd992e8284530467a97a HTTP/1.1
Host: gcr.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Authorization: Bearer AJAD5v0sN9g4glsFyOyeRUycw1lLDUpkrSkVn+8YbmfHPUlCS4+hSi9DSa+uPAzpqFV/VLFREmvXggl4y08aPirghFnnBNcanwVhbbs+frhLGouymult0lA/JUoe4QSj/hDf4EU503TFdwMSax6K75C/QCXuDILNCSXYOXdwwhroZwraFDxlc1INXtps4i+kzSFNBTEoUZmqUE60YAcv3ZMlLgKOMyVFByUdaV9rjZ925xHgQYOt5/2TLdvL1qHD3f97zWwd1fSEBFNquqz7DguB2IgTaw8+sEE01kBDmv4X9giZx/eBjBekrIbJIiiDUoRe5wSEuvyk3ocPh99/Fqd9iYMaX7/lVBwJXoxfLYSxHc/XzlsmDoplkKvScVL7Q92Ko7JeJzCCXcMR40i32bJzj5mHQBUPoVo=
Connection: close


HTTP/1.1 302 Found
Docker-Distribution-API-Version: registry/2.0
Location: https://storage.googleapis.com/artifacts.google-containers.appspot.com/containers/images/sha256:7cc17f1decf73673c3093bbbab0e5770ebd17b2201cbfd992e8284530467a97a
Content-Type: application/json
Date: Thu, 20 Aug 2020 03:38:46 GMT
Server: Docker Registry
Cache-Control: private
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Accept-Ranges: none
Vary: Accept-Encoding
Connection: close
Content-Length: 13

{"errors":[]}
```

```
GET https://gcr.io/v2/google-containers/kube-controller-manager/blobs/sha256:95b2e4f548f19fafcc7fbf683024ee9abc250394e22b0fb879d8d488099c0177 HTTP/1.1
Host: gcr.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Authorization: Bearer AJAD5v0sN9g4glsFyOyeRUycw1lLDUpkrSkVn+8YbmfHPUlCS4+hSi9DSa+uPAzpqFV/VLFREmvXggl4y08aPirghFnnBNcanwVhbbs+frhLGouymult0lA/JUoe4QSj/hDf4EU503TFdwMSax6K75C/QCXuDILNCSXYOXdwwhroZwraFDxlc1INXtps4i+kzSFNBTEoUZmqUE60YAcv3ZMlLgKOMyVFByUdaV9rjZ925xHgQYOt5/2TLdvL1qHD3f97zWwd1fSEBFNquqz7DguB2IgTaw8+sEE01kBDmv4X9giZx/eBjBekrIbJIiiDUoRe5wSEuvyk3ocPh99/Fqd9iYMaX7/lVBwJXoxfLYSxHc/XzlsmDoplkKvScVL7Q92Ko7JeJzCCXcMR40i32bJzj5mHQBUPoVo=
Connection: close

HTTP/1.1 302 Found
Docker-Distribution-API-Version: registry/2.0
Location: https://storage.googleapis.com/artifacts.google-containers.appspot.com/containers/images/sha256:95b2e4f548f19fafcc7fbf683024ee9abc250394e22b0fb879d8d488099c0177
Content-Type: application/json
Date: Thu, 20 Aug 2020 03:38:46 GMT
Server: Docker Registry
Cache-Control: private
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Accept-Ranges: none
Vary: Accept-Encoding
Connection: close
Content-Length: 13

{"errors":[]}
```

### 6. 下载数据

```
GET https://storage.googleapis.com/artifacts.google-containers.appspot.com/containers/images/sha256:7cc17f1decf73673c3093bbbab0e5770ebd17b2201cbfd992e8284530467a97a HTTP/1.1
Host: storage.googleapis.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Referer: https://gcr.io/v2/google-containers/kube-controller-manager/blobs/sha256:7cc17f1decf73673c3093bbbab0e5770ebd17b2201cbfd992e8284530467a97a
Connection: close

HTTP/1.1 200 OK
X-GUploader-UploadID: AAANsUlCiop-4CGwAF78mzgpfKTNkDrajNOcqbIbCi5afcRcgt_8OehGBTRt85Nm3RktAiFx5N-Szwhc_iFa6atJSPE
Expires: Thu, 20 Aug 2020 04:38:46 GMT
Date: Thu, 20 Aug 2020 03:38:46 GMT
Cache-Control: public, max-age=3600
Last-Modified: Wed, 20 May 2020 17:27:37 GMT
ETag: "9901dbfbbdb362a6647b188e21c30767"
x-goog-generation: 1589995657415353
x-goog-metageneration: 1
x-goog-stored-content-encoding: identity
x-goog-stored-content-length: 27897052
Content-Type: application/octet-stream
x-goog-hash: crc32c=1lRBcQ==
x-goog-hash: md5=mQHb+72zYqZkexiOIcMHZw==
x-goog-storage-class: STANDARD
Accept-Ranges: bytes
Content-Length: 27897052
Server: UploadServer
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Connection: close

数据
```

```
GET https://storage.googleapis.com/artifacts.google-containers.appspot.com/containers/images/sha256:95b2e4f548f19fafcc7fbf683024ee9abc250394e22b0fb879d8d488099c0177 HTTP/1.1
Host: storage.googleapis.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Referer: https://gcr.io/v2/google-containers/kube-controller-manager/blobs/sha256:95b2e4f548f19fafcc7fbf683024ee9abc250394e22b0fb879d8d488099c0177
Connection: close


HTTP/1.1 200 OK
X-GUploader-UploadID: AAANsUk7ywkUgTV5OAqpwmWv0fFO6f6Nhf_HJXYcUXn6B9LeCvY9b_0Dz7BPVd-7FJl1cWks8koKlfQz0hxNQ7IW_WY
Expires: Thu, 20 Aug 2020 04:30:20 GMT
Date: Thu, 20 Aug 2020 03:30:20 GMT
Last-Modified: Wed, 20 May 2020 17:27:38 GMT
ETag: "1af6019565a86f8205dd28692cac02e1"
x-goog-generation: 1589995658964510
x-goog-metageneration: 1
x-goog-stored-content-encoding: identity
x-goog-stored-content-length: 1786
Content-Type: application/octet-stream
x-goog-hash: crc32c=+bDrmg==
x-goog-hash: md5=GvYBlWWob4IF3ShpLKwC4Q==
x-goog-storage-class: STANDARD
Accept-Ranges: bytes
Content-Length: 1786
Server: UploadServer
Cache-Control: public, max-age=3600
Age: 506
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Connection: close

{"architecture":"amd64","config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr

```

```
GET https://storage.googleapis.com/artifacts.google-containers.appspot.com/containers/images/sha256:83b4483280e5187b2801b449338d5755e5874ab80c44bf1ce615d258142e7c8b HTTP/1.1
Host: storage.googleapis.com
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Referer: https://gcr.io/v2/google-containers/kube-controller-manager/blobs/sha256:83b4483280e5187b2801b449338d5755e5874ab80c44bf1ce615d258142e7c8b
Connection: close

HTTP/1.1 200 OK
X-GUploader-UploadID: AAANsUneWrnCLG3iuj92Mii-OFQd3WSHRdDMBX_nU2jostFsgY0QoWF8_yBiJ7HFEUBXOki1pCov79lXC9D36BGCm1Q
Expires: Thu, 20 Aug 2020 04:38:47 GMT
Date: Thu, 20 Aug 2020 03:38:47 GMT
Cache-Control: public, max-age=3600
Last-Modified: Wed, 15 Jul 2020 19:04:05 GMT
ETag: "fb9c3a08f03e341737af0cad2c45ddcf"
x-goog-generation: 1594839845698496
x-goog-metageneration: 1
x-goog-stored-content-encoding: identity
x-goog-stored-content-length: 21082831
Content-Type: application/octet-stream
x-goog-hash: crc32c=wOQJCQ==
x-goog-hash: md5=+5w6CPA+NBc3rwytLEXdzw==
x-goog-storage-class: STANDARD
Accept-Ranges: bytes
Content-Length: 21082831
Server: UploadServer
Alt-Svc: h3-29=":443"; ma=2592000,h3-27=":443"; ma=2592000,h3-T050=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Connection: close

```



## 下载 quay 仓库镜像

```bash
docker pull quay.io/watchdogpolska/alpine-curl
```

### 1. 获取 registry  v2 api 信息

```
GET https://quay.io/v2/ HTTP/1.1
Host: quay.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: gzip
Connection: close


HTTP/1.1 401 UNAUTHORIZED
Server: nginx/1.12.1
Date: Thu, 20 Aug 2020 06:25:06 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 4
Connection: close
Docker-Distribution-API-Version: registry/2.0
WWW-Authenticate: Bearer realm="https://quay.io/v2/auth",service="quay.io"

true
```

### 2. 获取 token

```
GET https://quay.io/v2/auth?scope=repository%3Awatchdogpolska%2Falpine-curl%3Apull&service=quay.io HTTP/1.1
Host: quay.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: gzip
Connection: close


HTTP/1.1 200 OK
Server: nginx/1.12.1
Date: Thu, 20 Aug 2020 06:25:07 GMT
Content-Type: application/json
Content-Length: 890
Connection: close
Cache-Control: no-cache, no-store, must-revalidate
X-Frame-Options: DENY
Strict-Transport-Security: max-age=63072000; preload

{"token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImMyNmRkODViZmNjYjNkOGYxMTZhMDUwNDAzYTQ3ZGI2ODVhNGEzMjg4ZDNiMDk0MjA0NTQ3Y2FjNjczMTJkYmEifQ.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6IndhdGNoZG9ncG9sc2thL2FscGluZS1jdXJsIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiY29udGV4dCI6eyJjb20uYXBvc3RpbGxlLnJvb3RzIjp7IndhdGNoZG9ncG9sc2thL2FscGluZS1jdXJsIjoiJGRpc2FibGVkIn0sImNvbS5hcG9zdGlsbGUucm9vdCI6IiRkaXNhYmxlZCJ9LCJhdWQiOiJxdWF5LmlvIiwiZXhwIjoxNTk3OTA4MzA3LCJpc3MiOiJxdWF5IiwiaWF0IjoxNTk3OTA0NzA3LCJuYmYiOjE1OTc5MDQ3MDcsInN1YiI6Iihhbm9ueW1vdXMpIn0.xViKnuzeK5AfTZweN5G2COowd6EP-JL4MUXlnDQewyFjYR3sc-ruNgsnmW5D2b8fxs9wSwPI546NwK9qv3ggjEV7tXB-DyzPY_x_WVaC91OY0WolF20kUMtVZ_cKkhjI7eNA3UvBh7C7HMnnZhIFGm0W5nKX3TT1iY9BfY-f8dJyWzyyD-vW8qySjrzaFw6nQlX5V7Bv8hFX7rLCGJ2upVSMf83Tio-OEwJ1xCRrs4W6DTSrcJ4_M_ujgjO6YyZlZxz8gVkMn2ziKZJoqTb_nQMjjnOvG2zkzki2254WARrKGpr0AWj30tHmX-ve6DMEiYxDsJsSw5Da1lNr2ttbeQ"}
```

### 3. 获取镜像标签的清单信息

```
GET https://quay.io/v2/watchdogpolska/alpine-curl/manifests/latest HTTP/1.1
Host: quay.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept: application/vnd.docker.distribution.manifest.v2+json
Accept: application/vnd.docker.distribution.manifest.list.v2+json
Accept: application/vnd.oci.image.index.v1+json
Accept: application/vnd.oci.image.manifest.v1+json
Accept: application/vnd.docker.distribution.manifest.v1+prettyjws
Accept: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImMyNmRkODViZmNjYjNkOGYxMTZhMDUwNDAzYTQ3ZGI2ODVhNGEzMjg4ZDNiMDk0MjA0NTQ3Y2FjNjczMTJkYmEifQ.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6IndhdGNoZG9ncG9sc2thL2FscGluZS1jdXJsIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiY29udGV4dCI6eyJjb20uYXBvc3RpbGxlLnJvb3RzIjp7IndhdGNoZG9ncG9sc2thL2FscGluZS1jdXJsIjoiJGRpc2FibGVkIn0sImNvbS5hcG9zdGlsbGUucm9vdCI6IiRkaXNhYmxlZCJ9LCJhdWQiOiJxdWF5LmlvIiwiZXhwIjoxNTk3OTA4MzA3LCJpc3MiOiJxdWF5IiwiaWF0IjoxNTk3OTA0NzA3LCJuYmYiOjE1OTc5MDQ3MDcsInN1YiI6Iihhbm9ueW1vdXMpIn0.xViKnuzeK5AfTZweN5G2COowd6EP-JL4MUXlnDQewyFjYR3sc-ruNgsnmW5D2b8fxs9wSwPI546NwK9qv3ggjEV7tXB-DyzPY_x_WVaC91OY0WolF20kUMtVZ_cKkhjI7eNA3UvBh7C7HMnnZhIFGm0W5nKX3TT1iY9BfY-f8dJyWzyyD-vW8qySjrzaFw6nQlX5V7Bv8hFX7rLCGJ2upVSMf83Tio-OEwJ1xCRrs4W6DTSrcJ4_M_ujgjO6YyZlZxz8gVkMn2ziKZJoqTb_nQMjjnOvG2zkzki2254WARrKGpr0AWj30tHmX-ve6DMEiYxDsJsSw5Da1lNr2ttbeQ
Accept-Encoding: gzip
Connection: close



HTTP/1.1 200 OK
Server: nginx/1.12.1
Date: Thu, 20 Aug 2020 06:25:09 GMT
Content-Type: application/vnd.docker.distribution.manifest.v2+json
Content-Length: 738
Connection: close
Docker-Content-Digest: sha256:51a5d57783c91797433db97b0ce1582ab1a6cf0631e6d0e45c2e7fb41608121f
X-Frame-Options: DENY
Strict-Transport-Security: max-age=63072000; preload

{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1926,
      "digest": "sha256:7e7499db3a4c397250db4d7e3d3e93312dc8ebaf5a8533d62c6e41b7a44083fa"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 2797541,
         "digest": "sha256:df20fa9351a15782c64e6dddb2d4a6f50bf6d3688060a34c4014b0d9a752eb4c"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 688971,
         "digest": "sha256:c365e8ee56bb1c3ec9230d567ab48c6ff9fa00fb116c7dafcdb673c4c6652ba4"
      }
   ]
}
```

这里因为镜像标签只有一个manifests，直接返回了该manifests的分层信息

### 4. 依次拉取layers层的数据

```
GET https://quay.io/v2/watchdogpolska/alpine-curl/blobs/sha256:c365e8ee56bb1c3ec9230d567ab48c6ff9fa00fb116c7dafcdb673c4c6652ba4 HTTP/1.1
Host: quay.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImMyNmRkODViZmNjYjNkOGYxMTZhMDUwNDAzYTQ3ZGI2ODVhNGEzMjg4ZDNiMDk0MjA0NTQ3Y2FjNjczMTJkYmEifQ.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6IndhdGNoZG9ncG9sc2thL2FscGluZS1jdXJsIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiY29udGV4dCI6eyJjb20uYXBvc3RpbGxlLnJvb3RzIjp7IndhdGNoZG9ncG9sc2thL2FscGluZS1jdXJsIjoiJGRpc2FibGVkIn0sImNvbS5hcG9zdGlsbGUucm9vdCI6IiRkaXNhYmxlZCJ9LCJhdWQiOiJxdWF5LmlvIiwiZXhwIjoxNTk3OTA4MzA3LCJpc3MiOiJxdWF5IiwiaWF0IjoxNTk3OTA0NzA3LCJuYmYiOjE1OTc5MDQ3MDcsInN1YiI6Iihhbm9ueW1vdXMpIn0.xViKnuzeK5AfTZweN5G2COowd6EP-JL4MUXlnDQewyFjYR3sc-ruNgsnmW5D2b8fxs9wSwPI546NwK9qv3ggjEV7tXB-DyzPY_x_WVaC91OY0WolF20kUMtVZ_cKkhjI7eNA3UvBh7C7HMnnZhIFGm0W5nKX3TT1iY9BfY-f8dJyWzyyD-vW8qySjrzaFw6nQlX5V7Bv8hFX7rLCGJ2upVSMf83Tio-OEwJ1xCRrs4W6DTSrcJ4_M_ujgjO6YyZlZxz8gVkMn2ziKZJoqTb_nQMjjnOvG2zkzki2254WARrKGpr0AWj30tHmX-ve6DMEiYxDsJsSw5Da1lNr2ttbeQ
Connection: close



HTTP/1.1 302 FOUND
Server: nginx/1.12.1
Date: Thu, 20 Aug 2020 06:25:10 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1229
Connection: close
Location: https://cdn02.quay.io/sha256/c3/c365e8ee56bb1c3ec9230d567ab48c6ff9fa00fb116c7dafcdb673c4c6652ba4?Expires=1597905310&Signature=ETFKe7k0WMPM5AgJ4qbT9CIq-v4Bl1XrUfOa1WiGwAYo4ZQC-gO9VPX1wfo-X5fLQEzzlhhTf~NnAzp3Gd5vTpSwveFNpkYUVeDhtW1uVmJ8PIdiC3zRAeRf0nUgEHYUCSY-Fv4S9JQUk0-7E4vKmlrXN25GClDH9H~KlfBG8lPlM58c5GACV34JNJzdemY06LREGXavNXFiPKzAKaokDmRoJp7RzDzZkOAiT3L9kSG9I~JKV3IztDE6J5YxA~XRSyyhyrTsdPLYU-s0sT-sWpEj0fyPFzCb5Ye4qbj5OunU4v9LV3FAyTBCjTKdA-AAmgTbg~x9Cr0lbKXOyiOrOA__&Key-Pair-Id=APKAJ67PQLWGCSP66DGA
Accept-Ranges: bytes
Docker-Content-Digest: sha256:c365e8ee56bb1c3ec9230d567ab48c6ff9fa00fb116c7dafcdb673c4c6652ba4
Cache-Control: max-age=31536000
X-Frame-Options: DENY
Strict-Transport-Security: max-age=63072000; preload

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="https://cdn02.quay.io/sha256/c3/c365e8ee56bb1c3ec9230d567ab48c6ff9fa00fb116c7dafcdb673c4c6652ba4?Expires=1597905310&amp;Signature=ETFKe7k0WMPM5AgJ4qbT9CIq-v4Bl1XrUfOa1WiGwAYo4ZQC-gO9VPX1wfo-X5fLQEzzlhhTf~NnAzp3Gd5vTpSwveFNpkYUVeDhtW1uVmJ8PIdiC3zRAeRf0nUgEHYUCSY-Fv4S9JQUk0-7E4vKmlrXN25GClDH9H~KlfBG8lPlM58c5GACV34JNJzdemY06LREGXavNXFiPKzAKaokDmRoJp7RzDzZkOAiT3L9kSG9I~JKV3IztDE6J5YxA~XRSyyhyrTsdPLYU-s0sT-sWpEj0fyPFzCb5Ye4qbj5OunU4v9LV3FAyTBCjTKdA-AAmgTbg~x9Cr0lbKXOyiOrOA__&amp;Key-Pair-Id=APKAJ67PQLWGCSP66DGA">https://cdn02.quay.io/sha256/c3/c365e8ee56bb1c3ec9230d567ab48c6ff9fa00fb116c7dafcdb673c4c6652ba4?Expires=1597905310&amp;Signature=ETFKe7k0WMPM5AgJ4qbT9CIq-v4Bl1XrUfOa1WiGwAYo4ZQC-gO9VPX1wfo-X5fLQEzzlhhTf~NnAzp3Gd5vTpSwveFNpkYUVeDhtW1uVmJ8PIdiC3zRAeRf0nUgEHYUCSY-Fv4S9JQUk0-7E4vKmlrXN25GClDH9H~KlfBG8lPlM58c5GACV34JNJzdemY06LREGXavNXFiPKzAKaokDmRoJp7RzDzZkOAiT3L9kSG9I~JKV3IztDE6J5YxA~XRSyyhyrTsdPLYU-s0sT-sWpEj0fyPFzCb5Ye4qbj5OunU4v9LV3FAyTBCjTKdA-AAmgTbg~x9Cr0lbKXOyiOrOA__&amp;Key-Pair-Id=APKAJ67PQLWGCSP66DGA</a>.  If not click the link.
```

```
GET https://quay.io/v2/watchdogpolska/alpine-curl/blobs/sha256:7e7499db3a4c397250db4d7e3d3e93312dc8ebaf5a8533d62c6e41b7a44083fa HTTP/1.1
Host: quay.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImMyNmRkODViZmNjYjNkOGYxMTZhMDUwNDAzYTQ3ZGI2ODVhNGEzMjg4ZDNiMDk0MjA0NTQ3Y2FjNjczMTJkYmEifQ.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6IndhdGNoZG9ncG9sc2thL2FscGluZS1jdXJsIiwiYWN0aW9ucyI6WyJwdWxsIl19XSwiY29udGV4dCI6eyJjb20uYXBvc3RpbGxlLnJvb3RzIjp7IndhdGNoZG9ncG9sc2thL2FscGluZS1jdXJsIjoiJGRpc2FibGVkIn0sImNvbS5hcG9zdGlsbGUucm9vdCI6IiRkaXNhYmxlZCJ9LCJhdWQiOiJxdWF5LmlvIiwiZXhwIjoxNTk3OTA4MzA3LCJpc3MiOiJxdWF5IiwiaWF0IjoxNTk3OTA0NzA3LCJuYmYiOjE1OTc5MDQ3MDcsInN1YiI6Iihhbm9ueW1vdXMpIn0.xViKnuzeK5AfTZweN5G2COowd6EP-JL4MUXlnDQewyFjYR3sc-ruNgsnmW5D2b8fxs9wSwPI546NwK9qv3ggjEV7tXB-DyzPY_x_WVaC91OY0WolF20kUMtVZ_cKkhjI7eNA3UvBh7C7HMnnZhIFGm0W5nKX3TT1iY9BfY-f8dJyWzyyD-vW8qySjrzaFw6nQlX5V7Bv8hFX7rLCGJ2upVSMf83Tio-OEwJ1xCRrs4W6DTSrcJ4_M_ujgjO6YyZlZxz8gVkMn2ziKZJoqTb_nQMjjnOvG2zkzki2254WARrKGpr0AWj30tHmX-ve6DMEiYxDsJsSw5Da1lNr2ttbeQ
Connection: close


HTTP/1.1 302 FOUND
Server: nginx/1.12.1
Date: Thu, 20 Aug 2020 06:25:13 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1229
Connection: close
Location: https://cdn02.quay.io/sha256/7e/7e7499db3a4c397250db4d7e3d3e93312dc8ebaf5a8533d62c6e41b7a44083fa?Expires=1597905313&Signature=W0xPj-XiY5bssnrO5RWsT-Y38WVOv3YQwQkRqEgwBKimbyu160LbKv5bo44cg4p7aA~z-QdqRixfy0rA66TH-1tuuUAXv7O-MJxyuR8zIXTwnvb2~14lZ3wdK4qpk7Iqm-ZEb5p1ogWWPyFUmZCmOfhkmiWwpMzF0xkEpK6Obk3Xzh6wa7lbpN-xIFrkXE1sDk-A8gUp8uOgKiJMS6Zs9~6OTlpPA1kcE568eQHy~DbB63-81Fbnq8L7kYyjwVYrGqGWZhUA4bOVaqXjEZ8WW0Gtsv8FdPwXH7hqLkS-vC5mKX3M8iXCVZA~eT5c~ckMJ48vO8VoOxvFBM9jU8e5AA__&Key-Pair-Id=APKAJ67PQLWGCSP66DGA
Accept-Ranges: bytes
Docker-Content-Digest: sha256:7e7499db3a4c397250db4d7e3d3e93312dc8ebaf5a8533d62c6e41b7a44083fa
Cache-Control: max-age=31536000
X-Frame-Options: DENY
Strict-Transport-Security: max-age=63072000; preload

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="https://cdn02.quay.io/sha256/7e/7e7499db3a4c397250db4d7e3d3e93312dc8ebaf5a8533d62c6e41b7a44083fa?Expires=1597905313&amp;Signature=W0xPj-XiY5bssnrO5RWsT-Y38WVOv3YQwQkRqEgwBKimbyu160LbKv5bo44cg4p7aA~z-QdqRixfy0rA66TH-1tuuUAXv7O-MJxyuR8zIXTwnvb2~14lZ3wdK4qpk7Iqm-ZEb5p1ogWWPyFUmZCmOfhkmiWwpMzF0xkEpK6Obk3Xzh6wa7lbpN-xIFrkXE1sDk-A8gUp8uOgKiJMS6Zs9~6OTlpPA1kcE568eQHy~DbB63-81Fbnq8L7kYyjwVYrGqGWZhUA4bOVaqXjEZ8WW0Gtsv8FdPwXH7hqLkS-vC5mKX3M8iXCVZA~eT5c~ckMJ48vO8VoOxvFBM9jU8e5AA__&amp;Key-Pair-Id=APKAJ67PQLWGCSP66DGA">https://cdn02.quay.io/sha256/7e/7e7499db3a4c397250db4d7e3d3e93312dc8ebaf5a8533d62c6e41b7a44083fa?Expires=1597905313&amp;Signature=W0xPj-XiY5bssnrO5RWsT-Y38WVOv3YQwQkRqEgwBKimbyu160LbKv5bo44cg4p7aA~z-QdqRixfy0rA66TH-1tuuUAXv7O-MJxyuR8zIXTwnvb2~14lZ3wdK4qpk7Iqm-ZEb5p1ogWWPyFUmZCmOfhkmiWwpMzF0xkEpK6Obk3Xzh6wa7lbpN-xIFrkXE1sDk-A8gUp8uOgKiJMS6Zs9~6OTlpPA1kcE568eQHy~DbB63-81Fbnq8L7kYyjwVYrGqGWZhUA4bOVaqXjEZ8WW0Gtsv8FdPwXH7hqLkS-vC5mKX3M8iXCVZA~eT5c~ckMJ48vO8VoOxvFBM9jU8e5AA__&amp;Key-Pair-Id=APKAJ67PQLWGCSP66DGA</a>.  If not click the link.
```
### 5. 下载数据

```
GET https://cdn02.quay.io/sha256/c3/c365e8ee56bb1c3ec9230d567ab48c6ff9fa00fb116c7dafcdb673c4c6652ba4?Expires=1597905310&Signature=ETFKe7k0WMPM5AgJ4qbT9CIq-v4Bl1XrUfOa1WiGwAYo4ZQC-gO9VPX1wfo-X5fLQEzzlhhTf~NnAzp3Gd5vTpSwveFNpkYUVeDhtW1uVmJ8PIdiC3zRAeRf0nUgEHYUCSY-Fv4S9JQUk0-7E4vKmlrXN25GClDH9H~KlfBG8lPlM58c5GACV34JNJzdemY06LREGXavNXFiPKzAKaokDmRoJp7RzDzZkOAiT3L9kSG9I~JKV3IztDE6J5YxA~XRSyyhyrTsdPLYU-s0sT-sWpEj0fyPFzCb5Ye4qbj5OunU4v9LV3FAyTBCjTKdA-AAmgTbg~x9Cr0lbKXOyiOrOA__&Key-Pair-Id=APKAJ67PQLWGCSP66DGA HTTP/1.1
Host: cdn02.quay.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Referer: https://quay.io/v2/watchdogpolska/alpine-curl/blobs/sha256:c365e8ee56bb1c3ec9230d567ab48c6ff9fa00fb116c7dafcdb673c4c6652ba4
Connection: close


HTTP/1.1 200 OK
Content-Type: binary/octet-stream
Content-Length: 688971
Connection: close
Date: Thu, 20 Aug 2020 06:25:33 GMT
x-amz-replication-status: COMPLETED
Last-Modified: Sun, 05 Jul 2020 11:54:33 GMT
ETag: "a612d14a71a235194ca09d83c0557aaf-1"
x-amz-storage-class: INTELLIGENT_TIERING
x-amz-server-side-encryption: AES256
x-amz-version-id: Sp4fhSKA83BwoG9q94JemslUoWRjxOeN
Accept-Ranges: bytes
Server: AmazonS3
X-Cache: Miss from cloudfront
Via: 1.1 92ebddd34a5dacfb924391ae6946602a.cloudfront.net (CloudFront)
X-Amz-Cf-Pop: SIN5-C1
X-Amz-Cf-Id: vRaFGg8pXPoxtA8trVV3OmQNjXdTrrkL-K20PKq2PRR6T-FJBC2aCA==
数据。。。

```

```
GET https://cdn02.quay.io/sha256/7e/7e7499db3a4c397250db4d7e3d3e93312dc8ebaf5a8533d62c6e41b7a44083fa?Expires=1597905313&Signature=W0xPj-XiY5bssnrO5RWsT-Y38WVOv3YQwQkRqEgwBKimbyu160LbKv5bo44cg4p7aA~z-QdqRixfy0rA66TH-1tuuUAXv7O-MJxyuR8zIXTwnvb2~14lZ3wdK4qpk7Iqm-ZEb5p1ogWWPyFUmZCmOfhkmiWwpMzF0xkEpK6Obk3Xzh6wa7lbpN-xIFrkXE1sDk-A8gUp8uOgKiJMS6Zs9~6OTlpPA1kcE568eQHy~DbB63-81Fbnq8L7kYyjwVYrGqGWZhUA4bOVaqXjEZ8WW0Gtsv8FdPwXH7hqLkS-vC5mKX3M8iXCVZA~eT5c~ckMJ48vO8VoOxvFBM9jU8e5AA__&Key-Pair-Id=APKAJ67PQLWGCSP66DGA HTTP/1.1
Host: cdn02.quay.io
User-Agent: docker/19.03.12 go/go1.13.10 git-commit/48a66213fe kernel/5.8.0-1.el7.elrepo.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/19.03.12 \(linux\))
Accept-Encoding: identity
Referer: https://quay.io/v2/watchdogpolska/alpine-curl/blobs/sha256:7e7499db3a4c397250db4d7e3d3e93312dc8ebaf5a8533d62c6e41b7a44083fa
Connection: close


HTTP/1.1 200 OK
Content-Type: binary/octet-stream
Content-Length: 1926
Connection: close
Date: Thu, 20 Aug 2020 06:25:36 GMT
x-amz-replication-status: COMPLETED
Last-Modified: Sun, 05 Jul 2020 11:54:48 GMT
ETag: "00acd2993c51139b494a948712b812d4-1"
x-amz-server-side-encryption: AES256
x-amz-version-id: Lf1J.g6gpajmr7UFYzLAv_QyHdmFYk_w
Accept-Ranges: bytes
Server: AmazonS3
X-Cache: Miss from cloudfront
Via: 1.1 b95596d6887b20449c59c2fc9d141c4a.cloudfront.net (CloudFront)
X-Amz-Cf-Pop: SIN5-C1
X-Amz-Cf-Id: poHJXatHw0npFw1_zc5XTCNk85fI1dM5oWCu5eKRaGlBe3gsaflu0Q==

{"architecture":"amd64","author":"Adam Dobrawy \"http://github.com/ad-m/\"","config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/sh","-c","#(nop) ","CMD [\"curl\" \"--version\"]"],"ArgsEscaped":true,"Image":"sha256:415f5f6826e442f4051044858b6195aef1c1e9795500df0350cfa87fde540ada","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":null,"Labels":null},"container_config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":null,"Cmd":null,"Image":"","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":null,"Labels":null},"created":"2020-07-05T11:54:27.215613905Z","docker_version":"18.02.0-ce","history":[{"created":"2020-05-29T21:19:46.192045972Z","created_by":"/bin/sh -c #(nop) ADD file:c92c248239f8c7b9b3c067650954815f391b7bcb09023f984972c082ace2a8d0 in / "},{"created":"2020-05-29T21:19:46.363518345Z","created_by":"/bin/sh -c #(nop)  CMD [\"/bin/sh\"]","empty_layer":true},{"created":"2020-07-05T11:54:25.799315127Z","author":"Adam Dobrawy \"http://github.com/ad-m/\"","created_by":"/bin/sh -c #(nop)  MAINTAINER Adam Dobrawy \"http://github.com/ad-m/\"","empty_layer":true},{"created":"2020-07-05T11:54:27.089683779Z","author":"Adam Dobrawy \"http://github.com/ad-m/\"","created_by":"/bin/sh -c apk add --no-cache curl"},{"created":"2020-07-05T11:54:27.215613905Z","author":"Adam Dobrawy \"http://github.com/ad-m/\"","created_by":"/bin/sh -c #(nop)  CMD [\"curl\" \"--version\"]","empty_layer":true}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:50644c29ef5a27c9a40c393a73ece2479de78325cae7d762ef3cdc19bf42dd0a","sha256:fd424638085ccdeb7d3dfbac941b10b275c360bd0dd4dee8843b468d426cf65e"]}}
```



## 参考

- https://docs.docker.com/registry/spec/api/
- https://docs.docker.com/registry/spec/auth/token/
- https://docs.docker.com/registry/spec/manifest-v2-2/