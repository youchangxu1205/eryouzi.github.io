From a454c80226464c8a166c67cf95aea8d1a6a7c3fc Mon Sep 17 00:00:00 2001
From: "norval.xu" <norval.xu@cloudwise.com>
Date: Wed, 30 Apr 2025 16:25:35 +0800
Subject: [PATCH] =?UTF-8?q?add=20#DOSM-101=20INAA-14412=20=E7=8A=B6?=
 =?UTF-8?q?=E6=80=81=E4=BF=AE=E6=94=B9=E4=B8=BA=E5=90=8C=E6=AD=A5=E6=93=8D?=
 =?UTF-8?q?=E4=BD=9C=EF=BC=8Csignoff=E5=90=8C=E6=AD=A5=20ichamp?=
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit

---
 .../biz/controller/SignOffController.java     | 31 ++++++++++++++++---
 .../email/impl/MdlInstanceServiceImpl.java    | 14 +++++++++
 .../service/ichampsync/DataSynchronizer.java  | 13 ++++++--
 3 files changed, 52 insertions(+), 6 deletions(-)

diff --git a/src/main/java/com/cloudwise/douc/customization/biz/controller/SignOffController.java b/src/main/java/com/cloudwise/douc/customization/biz/controller/SignOffController.java
index a58b0b0..9c5b04a 100644
--- a/src/main/java/com/cloudwise/douc/customization/biz/controller/SignOffController.java
+++ b/src/main/java/com/cloudwise/douc/customization/biz/controller/SignOffController.java
@@ -4,6 +4,7 @@ import cn.hutool.core.map.MapUtil;
 import com.cloudwise.dosm.api.bean.form.GroupBean;
 import com.cloudwise.dosm.dbs.entity.DbsSignOffEntity;
 import com.cloudwise.dosm.dbs.service.DbsSignOffHistoryService;
+import com.cloudwise.dosm.dbs.service.DbsSignOffService;
 import com.cloudwise.dosm.facewall.extension.core.exception.BaseException;
 import com.cloudwise.douc.customization.biz.dao.MdlInstanceMapper;
 import com.cloudwise.douc.customization.biz.facade.UserService;
@@ -11,6 +12,7 @@ import com.cloudwise.douc.customization.biz.facade.user.UserGroupInfo;
 import com.cloudwise.douc.customization.biz.model.signoff.SignReturn;
 import com.cloudwise.douc.customization.biz.model.table.MdlInstance;
 import com.cloudwise.douc.customization.biz.service.email.MdlInstanceService;
+import com.cloudwise.douc.customization.biz.service.ichampsync.DataSynchronizer;
 import com.cloudwise.douc.customization.biz.service.signoff.SignOffService;
 import com.cloudwise.douc.customization.common.util.JsonUtils;
 import com.cloudwise.douc.dto.DubboCommonResp;
@@ -55,13 +57,20 @@ public class SignOffController {
     private DbsSignOffHistoryService dbsSignOffHistoryService;
     @Autowired
     private UserService userService;
+    @Autowired
+    private DataSynchronizer dataSynchronizer;
+
+    @Resource
+    private DbsSignOffService dbsSignOffService;
 
     @PostMapping("/insert")
     public Long insert(@RequestBody DbsSignOffEntity signOff, @RequestParam String workOrderId, HttpServletRequest request) {
         String userId = request.getHeader("userId");
         String accountId = request.getHeader("accountId");
         checkMdlIsCreateUser(workOrderId, userId, accountId);
-        return signOffService.insert(workOrderId, signOff, userId);
+        Long insert = signOffService.insert(workOrderId, signOff, userId);
+        dataSynchronizer.convertFormData2ResultData(workOrderId, null, com.cloudwise.douc.customization.common.util.JsonUtils.createObjectNode(),null,null, false);
+        return insert;
     }
 
     @PostMapping("/deleteBatch")
@@ -69,7 +78,9 @@ public class SignOffController {
         String userId = request.getHeader("userId");
         String accountId = request.getHeader("accountId");
         checkMdlIsCreateUser(workOrderId, userId, accountId);
-        return signOffService.deleteBatch(workOrderId, signOffId, userId);
+        int batch = signOffService.deleteBatch(workOrderId, signOffId, userId);
+        dataSynchronizer.convertFormData2ResultData(workOrderId, null, com.cloudwise.douc.customization.common.util.JsonUtils.createObjectNode(),null,null, false);
+        return batch;
     }
 
     @PostMapping("/insertBatch")
@@ -77,7 +88,9 @@ public class SignOffController {
         String userId = request.getHeader("userId");
         String accountId = request.getHeader("accountId");
         checkMdlIsCreateUser(workOrderId, userId, accountId);
-        return signOffService.insertBatch(workOrderId, signOff, userId);
+        int batch = signOffService.insertBatch(workOrderId, signOff, userId);
+        dataSynchronizer.convertFormData2ResultData(workOrderId, null, com.cloudwise.douc.customization.common.util.JsonUtils.createObjectNode(),null,null, false);
+        return batch;
     }
 
 
@@ -86,7 +99,9 @@ public class SignOffController {
         String userId = request.getHeader("userId");
         String accountId = request.getHeader("accountId");
         checkMdlIsCreateUser(workOrderId, userId, accountId);
-        return signOffService.update(workOrderId, signOff, userId);
+        Long update = signOffService.update(workOrderId, signOff, userId);
+        dataSynchronizer.convertFormData2ResultData(workOrderId, null, com.cloudwise.douc.customization.common.util.JsonUtils.createObjectNode(),null,null, false);
+        return update;
     }
 
 
@@ -105,6 +120,8 @@ public class SignOffController {
         String userId = request.getHeader("userId");
         String accountId = request.getHeader("accountId");
         signOffService.status(signOffId, status, userId, workOrderId);
+        dataSynchronizer.convertFormData2ResultData(workOrderId, null, com.cloudwise.douc.customization.common.util.JsonUtils.createObjectNode(),null,null, false);
+
     }
 
 
@@ -112,7 +129,10 @@ public class SignOffController {
     public void rejected(@RequestParam("signOffId") Long signOffId, @RequestParam(value = "rejectionReason", required = false) String rejectionReason, HttpServletRequest request) {
         String userId = request.getHeader("userId");
         String accountId = request.getHeader("accountId");
+        DbsSignOffEntity signOffEntity = dbsSignOffService.getSignOffById(signOffId);
         signOffService.rejected(signOffId, rejectionReason, userId, Boolean.TRUE,"SYSTEM");
+        dataSynchronizer.convertFormData2ResultData(signOffEntity.getWorkOrderId(), null, com.cloudwise.douc.customization.common.util.JsonUtils.createObjectNode(),null,null, false);
+
     }
 
 
@@ -121,7 +141,10 @@ public class SignOffController {
 
         String userId = request.getHeader("userId");
         String accountId = request.getHeader("accountId");
+        DbsSignOffEntity signOffEntity = dbsSignOffService.getSignOffById(signOffId);
         signOffService.approved(signOffId, userId, Boolean.TRUE,"SYSTEM");
+        dataSynchronizer.convertFormData2ResultData(signOffEntity.getWorkOrderId(), null, com.cloudwise.douc.customization.common.util.JsonUtils.createObjectNode(),null,null, false);
+
     }
 
 
diff --git a/src/main/java/com/cloudwise/douc/customization/biz/service/email/impl/MdlInstanceServiceImpl.java b/src/main/java/com/cloudwise/douc/customization/biz/service/email/impl/MdlInstanceServiceImpl.java
index 5206cf0..32b51ba 100644
--- a/src/main/java/com/cloudwise/douc/customization/biz/service/email/impl/MdlInstanceServiceImpl.java
+++ b/src/main/java/com/cloudwise/douc/customization/biz/service/email/impl/MdlInstanceServiceImpl.java
@@ -21,6 +21,7 @@ import com.cloudwise.douc.customization.biz.model.log.DbsApiLog;
 import com.cloudwise.douc.customization.biz.model.table.ActHiTask;
 import com.cloudwise.douc.customization.biz.model.table.MdlInstance;
 import com.cloudwise.douc.customization.biz.service.email.MdlInstanceService;
+import com.cloudwise.douc.customization.biz.service.ichampsync.DataSynchronizer;
 import com.cloudwise.douc.customization.common.config.DosmConfig;
 import com.cloudwise.douc.customization.common.util.JsonUtils;
 import com.fasterxml.jackson.databind.JsonNode;
@@ -28,6 +29,7 @@ import com.fasterxml.jackson.databind.node.ObjectNode;
 import lombok.extern.slf4j.Slf4j;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.beans.factory.annotation.Value;
+import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
 import org.springframework.stereotype.Service;
 
 import javax.annotation.Resource;
@@ -66,6 +68,10 @@ public class MdlInstanceServiceImpl implements MdlInstanceService {
     private UserSSOClient userSSOClient;
     @Autowired
     private DictMapper dictMapper;
+    @Autowired
+    private DataSynchronizer dataSynchronizer;
+    @Autowired
+    private ThreadPoolTaskExecutor taskExecutor;
 
 
     @Override
@@ -121,7 +127,11 @@ public class MdlInstanceServiceImpl implements MdlInstanceService {
                         log.info("System auto closed cancel on One ignore with not in time:{}", mdlInstance.getBizKey());
                     }
                 }
+                updateMdlInstance2ClosedCancel(mdlInstance);
                 handleRollbackAndUpdate(mdlInstance, isNeedRollback);
+                taskExecutor.execute(() -> {
+                    dataSynchronizer.convertFormData2ResultData(mdlInstance.getId(),null,JsonUtils.createObjectNode(),"Closed Cancel",null,false);
+                });
             } catch (Exception e) {
                 log.error("System auto closed cancel on One exception:{}", e.getMessage(), e);
             }
@@ -208,7 +218,11 @@ public class MdlInstanceServiceImpl implements MdlInstanceService {
                 } else {
                     log.info("System auto closed cancel on Five ignore with not open:bizkey:{}", mdlInstance.getBizKey());
                 }
+                updateMdlInstance2ClosedCancel(mdlInstance);
                 handleRollbackAndUpdate(mdlInstance, isNeedRollback);
+                taskExecutor.execute(() -> {
+                    dataSynchronizer.convertFormData2ResultData(mdlInstance.getId(),null,JsonUtils.createObjectNode(),"Closed Cancel",null,false);
+                });
             } catch (Exception e) {
                 log.error("updateMdlInstance2ClosedCancelOnFive error:{}", e.getMessage(), e);
             }
diff --git a/src/main/java/com/cloudwise/douc/customization/biz/service/ichampsync/DataSynchronizer.java b/src/main/java/com/cloudwise/douc/customization/biz/service/ichampsync/DataSynchronizer.java
index 1157bb6..530d82c 100644
--- a/src/main/java/com/cloudwise/douc/customization/biz/service/ichampsync/DataSynchronizer.java
+++ b/src/main/java/com/cloudwise/douc/customization/biz/service/ichampsync/DataSynchronizer.java
@@ -7,7 +7,6 @@ import cn.hutool.core.io.resource.ClassPathResource;
 import cn.hutool.core.text.CharSequenceUtil;
 import cn.hutool.http.HttpResponse;
 import cn.hutool.http.HttpUtil;
-import cn.hutool.json.JSONUtil;
 import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
 import com.baomidou.mybatisplus.core.toolkit.Wrappers;
 import com.cloudwise.dosm.api.bean.form.GroupBean;
@@ -53,7 +52,14 @@ import java.io.File;
 import java.nio.charset.Charset;
 import java.time.Duration;
 import java.time.temporal.ChronoUnit;
-import java.util.*;
+import java.util.ArrayList;
+import java.util.Collections;
+import java.util.Date;
+import java.util.HashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.Objects;
+import java.util.Optional;
 import java.util.stream.Collectors;
 
 import static com.cloudwise.douc.customization.biz.service.ichampsync.SignoffConstants.SignoffItem.IDR_SIGNOFF_PROCUTOVER;
@@ -244,6 +250,9 @@ public class DataSynchronizer {
         } else {
             mdlInstance = mdlInstanceMapper.selectMdlInstanceById(workOrderId);
         }
+        if (mdlInstance == null) {
+            return;
+        }
 
         String formDataStr = mdlInstance.getFormData();
         ObjectNode formData = (ObjectNode) JsonUtils.parseJsonNode(formDataStr);
-- 
2.39.5 (Apple Git-154)

