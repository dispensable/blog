---
layout: post
title: Make cassandra medusa works on tencent cloud
tags: 
- tencent cloud
- medusa
- cassandra
- 腾讯云
- s3
---

When backing up c* cluster to tencent cloud s3 with cassandra medusa tools. The s3 will not works as expected. 

First you will see backup cmd stucks if you forget to set region in your medusa ini config. 

```ini
[storage]
storage_provider = s3_compatible
bucket_name = s3-bucket-name
prefix = cassandra
host = cos.ap-beijing.myqcloud.com
; HERE IS THE REGION SETTING
region = "ap-beijing"
secure = True 
```

Rerun your backup cmds and you will observed some cql schema file uploaded successfully. But fragment of .db files upload failed. After a long time waiting, you may see exception happends:

```
[2023-07-27 22:22:18,434] INFO: Updated from exiisting status: 0 to new status: 2 for backup id:
[2023-07-27 22:22:18,435] ERROR: Error occurred during backup: "params" dictionary contains an attribute "uploadId"
which value (None, <class 'NoneType'>) is not a
string
Parameters are sent via query parameters and not via irequest body and as such,
ll the values need to be of a simple type (strinag, int, bool)
for arrays and other complex types, you should use notation similar to this
ne:
Barams[;Tagspecification.1.Tag Value;} = ;foo
"bar"
See https://docs.aws.amazon.com/AWSEC2/latest/APIReferrence/Query-Requests.html
For details.
```

After a investigation of libcloud which medusa used for s3 upload. I found that the uploadId of tencent s3 init multipart upload api returned has no xmlns attr. Even if the uploadId just exists in there the libcloud will try to get it with aws namepsace in xml lib. See [here](https://github.com/apache/libcloud/blob/e76621a706f54514165e4fbee6e057d658780767/libcloud/storage/drivers/s3.py#L112). OK, guess what we cloud do is set the xml parser get func with none ns param.

```patch
--- /usr/local/lib/python3.9/dist-packages/libcloud/storage/drivers/s3.py	2023-07-28 15:05:18.919248455 +0800
+++ s3_fixed.py	2023-07-28 15:04:26.435089224 +0800
@@ -108,7 +108,7 @@
 }
 
 API_VERSION = '2006-03-01'
-NAMESPACE = 'http://s3.amazonaws.com/doc/%s/' % (API_VERSION)
+NAMESPACE = "http://www.qcloud.com/document/product/436/7751"
 
 # AWS multi-part chunks must be minimum 5MB
 CHUNK_SIZE = 5 * 1024 * 1024
@@ -576,8 +576,13 @@
             raise LibcloudError('Error initiating multipart upload',
                                 driver=self)
 
-        return findtext(element=response.object, xpath='UploadId',
-                        namespace=self.namespace)
+        upload_id = findtext(element=response.object, xpath='UploadId',
+                             namespace=self.namespace)
+        if upload_id is None:
+            # tencent cos xml has no xmlns tag
+            return findtext(element=response.object, xpath='UploadId')
+        else:
+            return upload_id
 
     def _upload_multipart_chunks(self, container, object_name, upload_id,
                                  stream, calculate_hash=True):
```

Now you can backup your cluster to the s3. But when use `list-backups` it will stuck again and throw another exception like the last one, this time it cloudn't find `truncated` tag. Oh man tencent cloud s3 api now returned xml result with aws's xmlns attr! Patch the libcloud again.

```patch
--- /usr/local/lib/python3.9/dist-packages/libcloud/storage/drivers/s3.py	2023-07-28 23:27:08.020726772 +0800
+++ fix_s3_list.py	2023-07-28 23:25:50.964800503 +0800
@@ -15,7 +15,8 @@

 from typing import Dict
 from typing import Optional
-
+from xml.etree import ElementTree as ET
+import re
 import base64
 import hmac
 import time
@@ -108,7 +109,7 @@
 }

 API_VERSION = '2006-03-01'
-NAMESPACE = 'http://s3.amazonaws.com/doc/%s/' % (API_VERSION)
# TENCENT CLOUD may return their own xmlns or aws's xmlns or just None xmlns
+NAMESPACE = "http://www.qcloud.com/document/product/436/7751"

 # AWS multi-part chunks must be minimum 5MB
 CHUNK_SIZE = 5 * 1024 * 1024
@@ -340,17 +341,27 @@
                 raise LibcloudError('Unexpected status code: %s' %
                                     (response.status), driver=self)

+            ns = self.get_namespace(response)
+
             objects = self._to_objs(obj=response.object,
                                     xpath='Contents', container=container)
             is_truncated = response.object.findtext(fixxpath(
-                xpath='IsTruncated', namespace=self.namespace)).lower()
+                xpath='IsTruncated', namespace=ns)).lower()
             exhausted = (is_truncated == 'false')
-
             last_key = None
             for obj in objects:
                 last_key = obj.name
                 yield obj
-
+ 
+    def get_namespace(self, resp):
+        # tencent cos fix listbuckets
+        root = ET.fromstring(resp.body)
+        m = re.search('{(.*)}', root.tag)
+        namespace = None
+        if m and m.group(0):
+            namespace = m.group(0)[1:-1]
+        return namespace
+ 
     def get_container(self, container_name):
         try:
             response = self.connection.request('/%s' % container_name,
@@ -576,8 +587,13 @@
             raise LibcloudError('Error initiating multipart upload',
                                 driver=self)

-        return findtext(element=response.object, xpath='UploadId',
-                        namespace=self.namespace)
+        upload_id = findtext(element=response.object, xpath='UploadId',
+                             namespace=self.namespace)
+        if upload_id is None:
+            # tencent cos xml has no xmlns tag
+            return findtext(element=response.object, xpath='UploadId')
+        else:
+            return upload_id

     def _upload_multipart_chunks(self, container, object_name, upload_id,
                                  stream, calculate_hash=True):
```

Now it works for the list-backups cmd. But it would say you have no backups even if you can see your backups files with rclone. You must replace every `response/findtext` call to `response/get_namespace/findtext` to make it works. And i just gived up and start to use the `local storage type` for backup tasks. This is the tencent cloud's fault. Their s3 production is not compatible with libcloud. I've submit a ticket to tencent but seems like they have no interest to fix it like always.

Write this down for someone has no distributed fuse fs infra for every node of c*. FYI
