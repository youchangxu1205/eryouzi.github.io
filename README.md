package com.cloudwise.douc.customization.biz.service.ichampsync;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.io.FileUtil;
import cn.hutool.core.io.resource.ClassPathResource;
import cn.hutool.core.text.CharSequenceUtil;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;
import cn.hutool.json.JSONUtil;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.cloudwise.dosm.api.bean.form.GroupBean;
import com.cloudwise.dosm.api.bean.utils.JsonUtils;
import com.cloudwise.dosm.dbs.mapper.DbsSignOffMapper;
import com.cloudwise.dosm.dbs.vo.DbsSignOffEntity;
import com.cloudwise.douc.customization.biz.dao.DbsApiLogMapper;
import com.cloudwise.douc.customization.biz.dao.DictMapper;
import com.cloudwise.douc.customization.biz.dao.MdlApproveRecordDetailMapper;
import com.cloudwise.douc.customization.biz.dao.MdlApproveRecordMapper;
import com.cloudwise.douc.customization.biz.dao.MdlInstanceMapper;
import com.cloudwise.douc.customization.biz.dao.MdlInstanceTableDataMapper;
import com.cloudwise.douc.customization.biz.dao.UploadFileMapper;
import com.cloudwise.douc.customization.biz.enums.CrApprovalEnum;
import com.cloudwise.douc.customization.biz.facade.UserSSOClient;
import com.cloudwise.douc.customization.biz.facade.user.BaseExtend;
import com.cloudwise.douc.customization.biz.facade.user.UserInfo;
import com.cloudwise.douc.customization.biz.model.email.ApproveRecord;
import com.cloudwise.douc.customization.biz.model.email.dosm.ApproveResultEnum;
import com.cloudwise.douc.customization.biz.model.log.DbsApiLog;
import com.cloudwise.douc.customization.biz.model.table.MdlApproveRecord;
import com.cloudwise.douc.customization.biz.model.table.MdlApproveRecordDetail;
import com.cloudwise.douc.customization.biz.model.table.MdlInstance;
import com.cloudwise.douc.customization.biz.model.table.MdlInstanceTableData;
import com.cloudwise.douc.customization.biz.model.table.UploadFileInfo;
import com.cloudwise.douc.customization.biz.service.email.DosmCustomApprovalRecordService;
import com.cloudwise.douc.customization.common.config.DbsProperties;
import com.cloudwise.storage.FileStorageService;
import com.cloudwise.storage.Urler;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.JsonNodeType;
import com.fasterxml.jackson.databind.node.NullNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.util.StopWatch;

import java.io.File;
import java.nio.charset.Charset;
import java.time.Duration;
import java.time.temporal.ChronoUnit;
import java.util.*;
import java.util.stream.Collectors;

import static com.cloudwise.douc.customization.biz.service.ichampsync.SignoffConstants.SignoffItem.IDR_SIGNOFF_PROCUTOVER;
import static com.cloudwise.douc.customization.biz.service.ichampsync.SignoffConstants.SignoffItem.MD_DELEGATE_SIGNOFF;

/**
 * <p>
 *
 * </p>
 *
 * @author Norval.Xu
 * @since 2025/2/17
 */
@Slf4j
@Component
public class DataSynchronizer {

    private final MdlInstanceMapper mdlInstanceMapper;
    private final MdlInstanceTableDataMapper mdlInstanceTableDataMapper;
    private final DbsSignOffMapper signOffMapper;

    private final MdlApproveRecordMapper mdlApproveRecordMapper;
    private final MdlApproveRecordDetailMapper mdlApproveRecordDetailMapper;
    private final UserSSOClient userSSOClient;
    private ExtParam extParam;
    private final DbsApiLogMapper dbsApiLogMapper;

    private final DbsProperties dbsProperties;

    private final DictMapper dictMapper;

    private final FileStorageService fileStorageService;

    private final UploadFileMapper uploadFileMapper;

    public static final int DEFAULT_EXPIRY_TIME = 7 * 24 * 3600;

    private static final List<String> CLOSE_STATUS_LIST = Lists.newArrayList("Closed Backoutfail", "Closed Issues");
    public static final String NEED_L1DOT5_CMTEAM_VALUE_CODE = "NeedL15CMTeamApprove_value";
    public static final String NEED_ISSAPPROVE_VALUE_CODE = "NeedIssApprove_value";
    public static final String MDDELEGATE_SIGNOFF_TYPE = "mddelegatesignofftype";
    public static final String GROUP_NAME = "groupName";

    @Autowired
    DosmCustomApprovalRecordService dosmCustomApprovalRecordService;

    public DataSynchronizer(MdlInstanceMapper mdlInstanceMapper, MdlInstanceTableDataMapper mdlInstanceTableDataMapper, DbsSignOffMapper dbsSignOffMapper, MdlApproveRecordMapper mdlApproveRecordMapper, MdlApproveRecordDetailMapper mdlApproveRecordDetailMapper, UserSSOClient userSSOClient, DbsApiLogMapper dbsApiLogMapper, DbsProperties dbsProperties, DictMapper dictMapper, FileStorageService fileStorageService, UploadFileMapper uploadFileMapper) {
        this.mdlInstanceMapper = mdlInstanceMapper;
        this.mdlInstanceTableDataMapper = mdlInstanceTableDataMapper;
        this.signOffMapper = dbsSignOffMapper;
        this.mdlApproveRecordMapper = mdlApproveRecordMapper;
        this.mdlApproveRecordDetailMapper = mdlApproveRecordDetailMapper;
        this.userSSOClient = userSSOClient;
        this.dbsApiLogMapper = dbsApiLogMapper;
        this.dbsProperties = dbsProperties;
        this.dictMapper = dictMapper;
        this.fileStorageService = fileStorageService;
        this.uploadFileMapper = uploadFileMapper;
    }


    public void send2Ichamp(ObjectNode resultData, boolean isNew, MdlInstance mdlInstance, ObjectNode formData, List<MdlInstanceTableData> mdlInstanceTableData) {
        JsonNode lastRequestor = formData.get("LastRequestor");
        String userId = mdlInstance.getCreatedBy();
        if (lastRequestor != null && !lastRequestor.isEmpty()) {
            JsonNode jsonNode = lastRequestor.get(0);
            JsonNode userIdNode = jsonNode.get("userId");
            if (!userIdNode.asText().equals(userId)) {
                userId = userIdNode.asText();
            }
        }
        List<UserInfo> userListByIds = userSSOClient.getUserListByIds(Collections.singletonList(userId), "2", "110");
        List<String> ibankIds = userListByIds.stream().map(item -> {
            Optional<BaseExtend> first = item.getExtend().stream().filter(user -> user.getAlias().equalsIgnoreCase("1bankId")).findFirst();
            if (first.isPresent()) {
                return first.get().getValue();
            } else {
                return "";
            }
        }).collect(Collectors.toList());
        ObjectNode result = JsonUtils.createObjectNode();
        ObjectNode data = JsonUtils.createObjectNode();
        result.put("action", isNew ? "Create" : "Modify");
        resultData.put("changerequestor", ibankIds.get(0));
        data.set("formData", resultData);
        data.put("workOrderNo", mdlInstance.getBizKey());
        data.put("createdBy", ibankIds.get(0));
        data.put("createdDate", mdlInstance.getCreatedTime().getTime());
        result.set("data", data);
        String url = isNew ? dbsProperties.getIchampCreateUrl() : dbsProperties.getIchampUpdateUrl();
        String jsonString = JsonUtils.toJsonString(result);
        DbsApiLog dbsApiLog = new DbsApiLog();
        dbsApiLog.setCreatedTime(new Date());
        dbsApiLog.setRequestStatus(true);
        dbsApiLog.setRequestUrl(url);
        dbsApiLog.setRequestBody(jsonString);
        dbsApiLog.setApiModule("crStatusSync");

        StopWatch stopWatch = new StopWatch();
        String body = "";
        Map<String, Object> origin = new HashMap<>();
        try {
            stopWatch.start();
            HttpResponse response = HttpUtil.createPost(url).header("Content-Type", "application/json").header("appCode", dbsProperties.getAppCode()).header("appKey", dbsProperties.getAppKey()).body(jsonString).execute();
            stopWatch.stop();
            long totalTimeMillis = stopWatch.getTotalTimeMillis();
            log.info("CrStatusSync ichamp sync time: {} ms", totalTimeMillis);
            body = response.body();
            log.info("CrStatusSyncTriggermakeResult:{}", body);

            JsonNode node = JsonUtils.parseJsonNode(body);
            origin.put("mdlInstance", mdlInstance);
            origin.put("mdlInstanceTableData", mdlInstanceTableData);
            origin.put("reqBody", body);
            origin.put("time", totalTimeMillis);
            dbsApiLog.setResponseBody(JsonUtils.toJsonString(origin));
            dbsApiLog.setWorkOrderNo(mdlInstance.getBizKey());

            if ("200".equals(node.get("code").asText())) {
                JsonNode dataNode = node.get("data");
                JsonNode status = dataNode.get("Status");
                if ("FAILED".equals(status.asText())) {
                    dbsApiLog.setRequestStatus(false);
                    dbsApiLogMapper.insert(dbsApiLog);
                } else {
                    dbsApiLog.setRequestStatus(true);
                    dbsApiLogMapper.insert(dbsApiLog);
                }
            } else {
                dbsApiLog.setRequestStatus(false);
                dbsApiLogMapper.insert(dbsApiLog);
            }
        } catch (Exception e) {
            stopWatch.stop();
            long totalTimeMillis = stopWatch.getTotalTimeMillis();
            origin.put("time", totalTimeMillis);
            origin.put("errorInfo", e.getMessage());
            dbsApiLog.setResponseBody(JsonUtils.toJsonString(origin));
            dbsApiLog.setRequestStatus(false);
            dbsApiLogMapper.insert(dbsApiLog);
        }
    }

    private void setRiskValue(ObjectNode formData, String itsmFieldCode, String ichampFieldCode, ObjectNode resultNode) {
        JsonNode fieldInfo = formData.get(itsmFieldCode + "_value");
        if (fieldInfo == null) {
            resultNode.put(ichampFieldCode, "3");
            return;
        }
        String value = fieldInfo.asText();
        if (value.equalsIgnoreCase("low")) {
            resultNode.put(ichampFieldCode, "3");
        } else {
            resultNode.put(ichampFieldCode, "1");
        }
    }

    public void convertFormData2ResultData(String workOrderId, String rcaWorkOrderId, ObjectNode resultData, String crStatus, String rcaStatus, Boolean isNew) {
        if (extParam == null) {
            ClassPathResource resource = new ClassPathResource("classpath:ichamp/extParam.json");
            String readString = FileUtil.readString(resource.getFile(), Charset.defaultCharset());
            extParam = JsonUtils.parseObject(readString, ExtParam.class);
        }
        MdlInstance mdlInstance = null;
        if (StringUtils.isNotBlank(rcaWorkOrderId)) {
            MdlInstance rcaMdlInstance = mdlInstanceMapper.selectMdlInstanceById(rcaWorkOrderId);
            String formData = rcaMdlInstance.getFormData();
            ObjectNode rcaFormData = (ObjectNode) JsonUtils.parseJsonNode(formData);
            JsonNode crTicketNumber = rcaFormData.get("CRTicketNumber");
            if (crTicketNumber == null) {
                return;
            }
            String crTicketNumberValue = crTicketNumber.asText();
            if (StringUtils.isBlank(crTicketNumberValue)) {
                return;
            }
            mdlInstance = mdlInstanceMapper.selectMdlInstanceByBizKey(crTicketNumberValue);
            if (mdlInstance == null) {
                return;
            }
            workOrderId = mdlInstance.getId();
            Map<String, String> rcaFieldMapping = extParam.getRcaFieldMapping();
            for (Map.Entry<String, String> stringStringEntry : rcaFieldMapping.entrySet()) {
                String ichampFieldCode = stringStringEntry.getKey();
                String itsmFieldCode = stringStringEntry.getValue();
                fillIChampValue(ichampFieldCode, rcaFormData, itsmFieldCode, resultData);
            }
        } else {
            mdlInstance = mdlInstanceMapper.selectMdlInstanceById(workOrderId);
        }

        String formDataStr = mdlInstance.getFormData();
        ObjectNode formData = (ObjectNode) JsonUtils.parseJsonNode(formDataStr);
        Map<String, String> fieldMapping = extParam.getFieldMapping();
        for (Map.Entry<String, String> stringStringEntry : fieldMapping.entrySet()) {
            String ichampFieldCode = stringStringEntry.getKey();
            String itsmFieldCode = stringStringEntry.getValue();
            fillIChampValue(ichampFieldCode, formData, itsmFieldCode, resultData);
        }
        JsonNode crStatusValue = formData.get("crStatus_value");
        JsonNode rcastatus = resultData.get("rcastatus");
        if ((rcastatus == null || StringUtils.isBlank(rcastatus.asText())) && CLOSE_STATUS_LIST.contains(crStatusValue.asText())) {
            resultData.put("rcastatus", "RCA Pending");
        }


        List<MdlInstanceTableData> mdlInstanceTableData = mdlInstanceTableDataMapper.selectList(Wrappers.lambdaQuery(MdlInstanceTableData.class).eq(MdlInstanceTableData::getWorkOrderId, workOrderId)
                .eq(MdlInstanceTableData::getIsDel, 0));
        Map<String, List<MdlInstanceTableData>> mdlInstanceTableMap = mdlInstanceTableData.stream().collect(Collectors.groupingBy(MdlInstanceTableData::getTableCode));


        //Special handling
        if (mdlInstanceTableMap.containsKey("F_Reversion_Plan")) {
            List<MdlInstanceTableData> implementInfo = mdlInstanceTableMap.get("F_Reversion_Plan");
            int cmcrollbackduration = 0;
            Long minDate = null;
            Long maxDate = null;
            for (MdlInstanceTableData mdlInstanceTableData1 : implementInfo) {
                String rowDataStr = mdlInstanceTableData1.getRowData();
                JsonNode rowData = JsonUtils.parseJsonNode(rowDataStr);
                JsonNode startTimeFieldInfo = rowData.get("F_Start_Time");
                if (minDate == null) {
                    minDate = startTimeFieldInfo.asLong();
                } else {
                    minDate = Math.min(minDate, startTimeFieldInfo.asLong());
                }
                JsonNode endTimeFieldInfo = rowData.get("F_End_Time");
                if (maxDate == null) {
                    maxDate = endTimeFieldInfo.asLong();
                } else {
                    maxDate = Math.max(maxDate, endTimeFieldInfo.asLong());
                }
            }
            if (minDate != null && maxDate != null) {
                cmcrollbackduration = (int) ((maxDate - minDate) / (1000 * 60));
                resultData.put("cmcrollbackduration", cmcrollbackduration);
            } else {
                resultData.put("cmcrollbackduration", "NA");
            }


        } else {
            log.error("CrStatusSyncTrigger: implementInfo not found");
            resultData.put("cmcrollbackduration", "NA");
        }


        Map<String, List<TableFieldMapping>> tableFieldMappings = extParam.getTableFieldMapping();
        for (Map.Entry<String, List<TableFieldMapping>> item : tableFieldMappings.entrySet()) {
            String key = item.getKey();
            ObjectNode itemResult = JsonUtils.createObjectNode();
            List<TableFieldMapping> itemValue = item.getValue();
            for (TableFieldMapping tableFieldMapping : itemValue) {
                String itsmTableFieldInfo = tableFieldMapping.getItsmTableFieldInfo();
                List<MdlInstanceTableData> jobDetailsSection = mdlInstanceTableMap.get(itsmTableFieldInfo);
                if (jobDetailsSection != null) {
                    log.error("CrStatusSyncTrigger find table info:{}", itsmTableFieldInfo);
                    ArrayNode jobDetailResult = JsonUtils.createArrayNode();
                    Map<String, String> fieldMapping1 = tableFieldMapping.getFieldMapping();
                    for (MdlInstanceTableData rowData : jobDetailsSection) {
                        ObjectNode jobDetailItem = JsonUtils.createObjectNode();
                        String rowDataRowData = rowData.getRowData();
                        ObjectNode jsonNode = (ObjectNode) JsonUtils.parseJsonNode(rowDataRowData);
                        for (Map.Entry<String, String> stringStringEntry : fieldMapping1.entrySet()) {
                            fillIChampValue(stringStringEntry.getKey(), jsonNode, stringStringEntry.getValue(), jobDetailItem);
                        }
                        jobDetailResult.add(jobDetailItem);
                    }
                    itemResult.put(tableFieldMapping.getIchampTableFieldInfo(), jobDetailResult);
                } else {
                    log.error("CrStatusSyncTrigger not find table info:{}", itsmTableFieldInfo);
                }
            }
            if (!itemResult.isEmpty()) {
                resultData.put(key, itemResult);
            }
        }


        List<DbsSignOffEntity> signOffEntities = signOffMapper.selectList(Wrappers.lambdaQuery(DbsSignOffEntity.class).eq(DbsSignOffEntity::getWorkOrderId, workOrderId));
        for (DbsSignOffEntity signoffManager : signOffEntities) {
            String signOffType = signoffManager.getSignOffType();
            JsonNode signOffTypes = JsonUtils.parseJsonNode(signOffType);
            for (JsonNode offType : signOffTypes) {
                SignoffConstants.SignoffItem signoffItem = SignoffConstants.SignoffItem.fromString(offType.asText());
                if (signoffItem != null) {
                    if (signoffItem == IDR_SIGNOFF_PROCUTOVER) {
                        ArrayNode caseIdsResult = JsonUtils.createArrayNode();
                        JsonNode caseIds = JsonUtils.parseJsonNode(signoffManager.getCaseId());
                        if (caseIds != null && !caseIds.isEmpty()) {
                            for (JsonNode caseId : caseIds) {
                                ObjectNode caseIdResult = JsonUtils.createObjectNode();
                                caseIdResult.put("caseid", caseId.get("caseId").asText());
                                caseIdResult.put("remarks", caseId.get("showStatus").asText());
                                caseIdsResult.add(caseIdResult);
                            }
                        }
                        resultData.put("idrsignoffcaseid", caseIdsResult);
                    }

                    if (MD_DELEGATE_SIGNOFF == signoffItem) {
                        String groupNames = "";
                        if (CharSequenceUtil.isNotBlank(signoffManager.getSignOffUserGroup())) {
                            List<GroupBean> groupBeans = JsonUtils.convertList(signoffManager.getSignOffUserGroup(), GroupBean.class);
                            if (CollUtil.isNotEmpty(groupBeans)) {
                                groupNames = groupBeans.stream().
                                        filter(Objects::nonNull).map(GroupBean::getGroupName).
                                        filter(CharSequenceUtil::isNotBlank).
                                        collect(Collectors.joining(","));
                            }
                        }
                        resultData.put(MDDELEGATE_SIGNOFF_TYPE, groupNames);
                    }

                    JsonNode signOffUser = JsonUtils.parseJsonNode(signoffManager.getSignOffUser());
                    if (signOffUser != null && !signOffUser.isEmpty()) {
                        if (signoffItem.getSignOffApproverLoginCode() != null) {
                            String userId = signOffUser.get(0).get("userId").asText();
                            if (!"Approver Same as CR MD Approver".equalsIgnoreCase(userId)) {
                                UserInfo userInfo = userSSOClient.getUserById(userId);
                                List<BaseExtend> extend = userInfo.getExtend();
                                extend.stream().filter(item -> item.getAlias().equalsIgnoreCase("1bankid")).findFirst().ifPresent(item -> {
                                    String ibankId = item.getValue();
                                    resultData.put(signoffItem.getSignOffApproverLoginCode(), ibankId);
                                });
                            } else {
                                resultData.put(signoffItem.getSignOffApproverLoginCode(), userId);
                            }
                        }
                        SignOffStatusEnum signOffStatusEnum = SignOffStatusEnum.valueOf(signoffManager.getStatus());
                        if (signOffStatusEnum == SignOffStatusEnum.REJECTED || signOffStatusEnum == SignOffStatusEnum.APPROVED) {
                            if (StringUtils.isNotBlank(signoffItem.getSignOffStatusCode())) {
                                resultData.put(signoffItem.getSignOffStatusCode(), signOffStatusEnum.getDesc());
                            }
                            if (StringUtils.isNotBlank(signoffItem.getSignOffRejectionReasonCode())) {
                                resultData.put(signoffItem.getSignOffRejectionReasonCode(), signoffManager.getRejectionReason() == null ? "" : signoffManager.getRejectionReason());
                            }
                        } else {
                            if (StringUtils.isNotBlank(signoffItem.getSignOffStatusCode())) {
                                handleEmptyField(signoffItem.getSignOffStatusCode(), resultData);
                            }
                            if (StringUtils.isNotBlank(signoffItem.getSignOffRejectionReasonCode())) {
                                handleEmptyField(signoffItem.getSignOffRejectionReasonCode(), resultData);
                            }
                        }
                    }
                    handleSignoffUrl(signoffManager, signoffItem, resultData);
                }
            }
        }

        String techMdApproverTime = "";
        List<ApproveRecord> customApprovalRecords = dosmCustomApprovalRecordService.getCustomApprovalRecordsV1(workOrderId);
        log.info("Sync cr status customApprovalRecords:{},workOrderId:{}", customApprovalRecords, workOrderId);
        Map<String, ApproveRecord> map = new HashMap<>();
        if (!customApprovalRecords.isEmpty()) {
            log.info("CustomApprovalRecords is not empty");
            for (ApproveRecord approveRecord : customApprovalRecords) {
                String nodeId = approveRecord.getNodeId();
                map.put(nodeId, approveRecord);
            }
        }
        log.info("Sync cr status customApprovalRecords map:{}", map);

        //设置审批数据
        List<ApproveInfo> approveInfoMapping = extParam.getApproveInfoMapping();
        for (ApproveInfo approveInfo : approveInfoMapping) {
            String nodeId = approveInfo.getNodeId();
            ApproveRecord approveRecord = map.get(nodeId);
            log.info("Sync cr status nodeId:{},approveRecord:{}", nodeId, approveRecord);
            Map<String, String> mapFieldMapping = approveInfo.getMapFieldMapping();
            log.debug("Sync mapFieldMapping:{}", mapFieldMapping);
            if (Objects.isNull(mapFieldMapping)) {
                mapFieldMapping = new HashMap<>();
            }
            if (approveRecord != null) {
                for (Map.Entry<String, String> stringStringEntry : mapFieldMapping.entrySet()) {
                    String ichampFieldCode = stringStringEntry.getKey();
                    String itsmFieldCode = stringStringEntry.getValue();
                    String[] itsmFieldCodeAndType = itsmFieldCode.split(":");
                    if (itsmFieldCodeAndType.length == 2) {
                        String type = itsmFieldCodeAndType[1];
                        if ("MEMBER".equals(type)) {
                            String approverId = approveRecord.getApproverId();
                            if (CharSequenceUtil.isNotBlank(approverId)) {
                                fillIChampValueWithMember(ichampFieldCode, approverId, resultData);
                            } else {
                                fillIChampValue(ichampFieldCode, formData, itsmFieldCode, resultData);
                            }
                        } else {
                            fillIChampValue(ichampFieldCode, formData, itsmFieldCode, resultData);
                        }
                    } else {
                        fillIChampValue(ichampFieldCode, formData, itsmFieldCode, resultData);
                    }
                }
                String status = approveRecord.getStatus();
                if (CharSequenceUtil.isNotBlank(approveRecord.getReasonLable())) {
                    resultData.put(approveInfo.getRejectionCode(), approveRecord.getReasonLable());
                }
                if (StringUtils.isNotBlank(approveInfo.getStatusTimeCode())) {
                    if (Objects.isNull(approveRecord.getUpdateTime())) {
                        resultData.put(approveInfo.getStatusTimeCode(), "");
                    } else {
                        resultData.put(approveInfo.getStatusTimeCode(), DateUtil.formatDateTime(new Date(approveRecord.getUpdateTime())));
                    }
                }
                if (CharSequenceUtil.isNotBlank(approveRecord.getApproveMsg())) {
                    resultData.put(approveInfo.getReasonCode(), approveRecord.getApproveMsg());
                } else {
                    resultData.put(approveInfo.getReasonCode(), "");
                }
                if (approveInfo.getNodeId().equals("SingleApprove_0cntz02")) {
                    if (!Objects.isNull(approveRecord.getUpdateTime())) {
                        techMdApproverTime = DateUtil.formatDateTime(new Date(approveRecord.getUpdateTime()));
                    }
                }
                if (CrApprovalEnum.PENDING_APPROVAL.getName().equals(status)) {
                    resultData.put(approveInfo.getStatusCode(), "");
                    resultData.put(approveInfo.getRejectionCode(), "");
                } else {
                    resultData.put(approveInfo.getStatusCode(), CrApprovalEnum.APPROVED.getName().equals(status) ? "Approved" : "Rejected");
                }
            } else {
                log.info("Sync cr status is:{}", crStatusValue);
                //new 状态取表单数据同步
                if ("New".equals(crStatusValue.asText())) {
                    for (Map.Entry<String, String> stringStringEntry : mapFieldMapping.entrySet()) {
                        String ichampFieldCode = stringStringEntry.getKey();
                        String itsmFieldCode = stringStringEntry.getValue();
                        String[] itsmFieldCodeAndType = itsmFieldCode.split(":");
                        String type = itsmFieldCodeAndType[1];
                        if ("GROUP".equals(type)) {
                            fillIChampValueWithGroup(ichampFieldCode, formData, itsmFieldCode, resultData, type);
                        } else {
                            fillIChampValue(ichampFieldCode, formData, itsmFieldCode, resultData);
                        }
                    }
                } else {
                    for (Map.Entry<String, String> stringStringEntry : mapFieldMapping.entrySet()) {
                        String ichampFieldCode = stringStringEntry.getKey();
                        String itsmFieldCode = stringStringEntry.getValue();
                        fillIChampValue(ichampFieldCode, formData, itsmFieldCode, resultData);
                    }
                }
            }
        }
        if (StringUtils.isNotBlank(rcaWorkOrderId)) {
            //查询审批数据
            List<ApproveInfo> rcaApproveInfoMapping = extParam.getRcaApproveInfoMapping();
            for (ApproveInfo approveInfo : rcaApproveInfoMapping) {
                //if cr Status is not New ,need query approver status
                LambdaQueryWrapper<MdlApproveRecord> wrapper = Wrappers.lambdaQuery(MdlApproveRecord.class).eq(MdlApproveRecord::getWorkOrderId, rcaWorkOrderId).eq(MdlApproveRecord::getNodeId, approveInfo.getNodeId()).orderByDesc(MdlApproveRecord::getCreatedTime);
                MdlApproveRecord mdlApproveRecord = mdlApproveRecordMapper.selectOne(wrapper, false);
                log.error("CrStatus:queryApprove:{}", mdlApproveRecord);
                if (mdlApproveRecord != null) {
                    if (mdlApproveRecord.getApproveStatus().equals(ApproveStatusEnum.FINISHED.getCode() + "")) {
                        MdlApproveRecordDetail mdlApproveRecordDetail = mdlApproveRecordDetailMapper.selectOne(Wrappers.lambdaQuery(MdlApproveRecordDetail.class).eq(MdlApproveRecordDetail::getApproveRecordId, mdlApproveRecord.getId()), false);
                        log.error("CrStatus:mdlApproveRecordDetail:{}", mdlApproveRecordDetail);
                        if (mdlApproveRecordDetail != null) {
                            String dictId = mdlApproveRecordDetail.getReason();
                            if (StringUtils.isNotBlank(dictId) && StringUtils.isNotBlank(approveInfo.getRejectionCode())) {
                                String dataDictLableById = mdlApproveRecordMapper.getDataDictLableById(dictId);
                                resultData.put(approveInfo.getRejectionCode(), dataDictLableById);
                            }
                            resultData.put(approveInfo.getReasonCode(), mdlApproveRecordDetail.getApproveMsg());
                        }
                        resultData.put(approveInfo.getStatusCode(), mdlApproveRecord.getApproveResult() == ApproveResultEnum.PASS ? "Approved" : "Rejected");
                    }
                }
            }
        }

        setRiskValue(formData, "CustomerServices", "cmrmavailablity", resultData);
        setRiskValue(formData, "InherentandResidualrisks", "cmrminherentresidualrisks", resultData);
        setRiskValue(formData, "BusinessUnitsOperations", "cmrmcontinuity", resultData);
        setRiskValue(formData, "ChangeComplexity", "cmrmcomplexity", resultData);
        setRiskValue(formData, "BackoutComplexity", "cmrmimplement", resultData);
        setRiskValue(formData, "Documentation", "cmrmtraining", resultData);
        setRiskValue(formData, "Security", "cmrmsecurity", resultData);
        setRiskValue(formData, "Interfaces", "cmrminterfaces", resultData);

        resultData.put("itsmcrnumber", mdlInstance.getBizKey());
        resultData.put("implementationplan", "https://" + dbsProperties.getCloudwiseDomain() + "/docp/dosm/dosm/orderDetails?showType=handle&id=" + workOrderId + "&isType=&isShowBack=1&type=3d409c4dc7e741539194a528cd9f0d91");
        resultData.put("reversionplan", "https://" + dbsProperties.getCloudwiseDomain() + "/docp/dosm/dosm/orderDetails?showType=handle&id=" + workOrderId + "&isType=&isShowBack=1&type=3d409c4dc7e741539194a528cd9f0d91");
        if (StringUtils.isNotBlank(crStatus)) {
            resultData.put("state", crStatus);
        }
        if (StringUtils.isNotBlank(rcaStatus)) {
            resultData.put("rcastatus", rcaStatus);
        }

        handleClarificationsection(resultData, formData, mdlInstance);

        //
        JsonNode cmcdatapatchnumberofrecord = resultData.get("cmcdatapatchnumberofrecord");
        if (cmcdatapatchnumberofrecord == null) {
            resultData.put("cmcdatapatchnumberofrecord", "NA");
        } else {
            String cmcdatapatchnumberofrecordText = cmcdatapatchnumberofrecord.asText();
            if (StringUtils.isBlank(cmcdatapatchnumberofrecordText)) {
                resultData.put("cmcdatapatchnumberofrecord", "NA");
            }

        }

        JsonNode appOwnerForDataPatch = formData.get("AppOwner_forDataPatch");
        if (appOwnerForDataPatch != null && appOwnerForDataPatch.asText().equalsIgnoreCase("SameAsCRmDApprovers")) {
            resultData.put("applicationownername", "Same as CR MD Approver");
            resultData.put("applicationownercomment", resultData.get("mdstatusreason"));
            resultData.put("applicationownerapprovalstatus", resultData.get("mdapproverstatus"));
            resultData.put("appownerdatapatchapprovalstatustime", techMdApproverTime);
        }
        //l1.5
        JsonNode needL15CMTeamApprove = formData.get(NEED_L1DOT5_CMTEAM_VALUE_CODE);
        if (!(needL15CMTeamApprove != null && "Yes".equals(needL15CMTeamApprove.asText()))) {
            resultData.put("changemanagergroup", "");
            resultData.put("changemanagerrejectioncode", "");
            resultData.put("changemanagerstatusreason", "");
            resultData.put("changemanagerapproverstatus", "");
            resultData.put("changemanagerrejectioncode", "");
        }
        JsonNode needIssApprove = formData.get(NEED_ISSAPPROVE_VALUE_CODE);
        if (!(needIssApprove != null && "Yes".equals(needIssApprove.asText()))) {
            resultData.put("issoverduepatchapprovergroup", "");
            resultData.put("issoverduepatchapprover", "");
            resultData.put("issoverduepatchapproverstatusreason", "");
            resultData.put("issoverduepatchapproverstatus", "");
            resultData.put("issoverduepatchapproverrejectioncode", "");
        }


        log.debug("Sync cr resultData:{}", resultData);
        send2Ichamp(resultData, isNew, mdlInstance, formData, mdlInstanceTableData);
    }

    private void handleSignoffUrl(DbsSignOffEntity signoffManager, SignoffConstants.SignoffItem signoffItem, ObjectNode resultData) {
        if (StringUtils.isNotBlank(signoffItem.getSignOffUrlCode())) {
            if (signoffManager.getArtifact() != null && CharSequenceUtil.isNotBlank(signoffManager.getArtifact())) {
                JsonNode node = JsonUtils.parseJsonNode(signoffManager.getArtifact());
                List<String> urls = new ArrayList<>();
                for (JsonNode jsonNode : node) {
                    JsonNode id = jsonNode.get("id");
                    UploadFileInfo fileInfoPo = uploadFileMapper.selectFileInfoListById(id.asText());
                    String filePath = fileInfoPo.getUrl();
                    String fileName = filePath.substring(filePath.indexOf(File.separator) + 1);
                    try {
                        Urler url = fileStorageService.of(fileName).setBucket(filePath.substring(0, filePath.indexOf(File.separator))).url();
                        url.setExpire(Duration.of(DEFAULT_EXPIRY_TIME, ChronoUnit.YEARS));
                        urls.add(url.get());
                    } catch (Exception e) {
                        log.error("HandleSignoffUrl Artifact error: signoffManager:{},SignoffItem:{}", signoffManager, signoffItem, e);
                    }
                }
                resultData.put(signoffItem.getSignOffUrlCode(), String.join(",", urls));
            } else {
                resultData.put(signoffItem.getSignOffUrlCode(), "");
            }
        }
    }

    private void handleClarificationsection(ObjectNode resultData, ObjectNode formData, MdlInstance mdlInstance) {
        JsonNode anyDataMigrationQuestion = formData.get("Any_data_migration_question");
        JsonNode whyCRCATisNotPatchToAppCAT = formData.get("WhyCRCATisNotPatchToAppCAT");
        ArrayNode arrayNode = JsonUtils.createArrayNode();


        if (anyDataMigrationQuestion != null && StringUtils.isNotBlank(anyDataMigrationQuestion.asText())) {
            ObjectNode objectNode = JsonUtils.createObjectNode();
            objectNode.put("seekclarification", "Any data migration/loading? What happened to the data if reversion? ");
            objectNode.put("clarification", anyDataMigrationQuestion.asText());
            arrayNode.add(objectNode);
        }
        if (whyCRCATisNotPatchToAppCAT != null && StringUtils.isNotBlank(whyCRCATisNotPatchToAppCAT.asText())) {
            ObjectNode objectNode1 = JsonUtils.createObjectNode();
            objectNode1.put("seekclarification", "Any data migration/loading? What happened to the data if reversion? ");
            objectNode1.put("clarification", whyCRCATisNotPatchToAppCAT.asText());
            arrayNode.add(objectNode1);
        }
        resultData.put("clarificationsection", arrayNode);
    }

    private void fillIChampValue(String ichampFieldCode, ObjectNode formData, String itsmFieldCode, ObjectNode resultData) {
        String[] itsmFieldCodeAndType = itsmFieldCode.split(":");
        if (formData.has(itsmFieldCodeAndType[0])) {
            JsonNode itsmValue = formData.get(itsmFieldCodeAndType[0]);
            JsonNodeType nodeType = itsmValue.getNodeType();
            switch (nodeType) {
                case NULL:
                case MISSING:
                    handleEmptyField(ichampFieldCode, resultData);
                    break;
                case POJO:
                case BINARY:
                    log.error("not support");
                    break;
                case ARRAY:
                    fillIChamValueWithArrary(ichampFieldCode, formData, itsmFieldCodeAndType[0], resultData);
                    break;
                case BOOLEAN:
                    resultData.put(ichampFieldCode, itsmValue.asBoolean());
                    break;
                case OBJECT:
                    if (itsmValue.has("startDate")) {
                        String[] startAndEnd = ichampFieldCode.split(",");
                        if (itsmValue.get("startDate").asLong() == 0) {
                            resultData.set(startAndEnd[0], null);
                            resultData.set(startAndEnd[1], null);
                        } else {
                            resultData.put(startAndEnd[0], DateUtil.format(new Date(itsmValue.get("startDate").asLong()), "yyyy-MM-dd HH:mm:ss"));
                            resultData.put(startAndEnd[1], DateUtil.format(new Date(itsmValue.get("endDate").asLong()), "yyyy-MM-dd HH:mm:ss"));
                        }
                    } else {
                        log.error("not support");
                    }
                    break;
                default:
                    if (formData.has(itsmFieldCodeAndType[0] + "_value")) {
                        String label = formData.get(itsmFieldCodeAndType[0] + "_value").asText();
                        String text = itsmValue.asText();
                        if (StringUtils.isNotBlank(text) && StringUtils.isBlank(label)) {
                            resultData.put(ichampFieldCode, queryDictLabel(Collections.singletonList(text), getDictMapping(itsmFieldCodeAndType[0])));
                        } else {
                            resultData.put(ichampFieldCode, label);
                        }

                    } else {
                        if (itsmFieldCodeAndType.length > 1) {
                            if ("DATE".equalsIgnoreCase(itsmFieldCodeAndType[1])) {
                                if (itsmValue.asLong() == 0) {
                                    resultData.set(ichampFieldCode, null);
                                } else {
                                    resultData.put(ichampFieldCode, DateUtil.format(new Date(itsmValue.asLong()), "yyyy-MM-dd HH:mm:ss"));
                                }
                            } else {
                                resultData.put(ichampFieldCode, itsmValue.asText());
                            }
                        } else {
                            resultData.put(ichampFieldCode, itsmValue.asText());
                        }

                    }
                    break;
            }
        } else {
            handleEmptyField(ichampFieldCode, resultData);
        }
    }

    private static void handleEmptyField(String ichampFieldCode, ObjectNode resultData) {
        if (ichampFieldCode.contains(",")) {
            String[] fieldCode = ichampFieldCode.split(",");
            for (String field : fieldCode) {
                resultData.put(field, "");
            }
        } else {
            resultData.put(ichampFieldCode, "");
        }
    }

    private void fillIChamValueWithArrary(String ichampFieldCode, ObjectNode formData, String itsmFieldCode, ObjectNode resultData) {
        if (formData.has(itsmFieldCode + "_value") && !formData.has(itsmFieldCode + "_search")) {
            JsonNode itsmLabel = formData.get(itsmFieldCode + "_value");
            JsonNode itsmValue = formData.get(itsmFieldCode);
            if (itsmValue.size() > 0 && itsmLabel.size() == 0) {
                List<String> values = new ArrayList<>();
                for (JsonNode jsonNode : itsmValue) {
                    values.add(jsonNode.asText());
                }
                resultData.put(ichampFieldCode, queryDictLabel(values, getDictMapping(itsmFieldCode)));
            } else {
                List<String> values = new ArrayList<>();
                for (JsonNode jsonNode : itsmLabel) {
                    values.add(jsonNode.asText());
                }
                resultData.put(ichampFieldCode, String.join(",", values));
            }
        } else {
            JsonNode itsmValue = formData.get(itsmFieldCode);
            List<String> values = new ArrayList<>();
            boolean isUser = false;
            for (JsonNode jsonNode : itsmValue) {
                if (jsonNode instanceof ObjectNode) {
                    JsonNode userId = jsonNode.get("userId");
                    JsonNode groupName = jsonNode.get("groupName");
                    if (userId != null && !(userId instanceof NullNode) && StringUtils.isNotBlank(userId.asText()) && !"null".equalsIgnoreCase(userId.asText())) {
                        isUser = true;
                        values.add(userId.asText());
                    } else if (groupName != null && !(groupName instanceof NullNode) && StringUtils.isNotBlank(groupName.asText()) && !"null".equalsIgnoreCase(groupName.asText())) {
                        values.add(groupName.asText());
                    }
                } else {
                    values.add(jsonNode.asText());
                }
            }
            if (isUser) {
                List<UserInfo> userListByIds = userSSOClient.getUserListByIds(values, "2", "110");
                List<String> userNames = userListByIds.stream().map(UserInfo::getName).collect(Collectors.toList());
                List<String> ibankIds = userListByIds.stream().map(item -> {
                    Optional<BaseExtend> first = item.getExtend().stream().filter(user -> user.getAlias().equalsIgnoreCase("1bankId")).findFirst();
                    if (first.isPresent()) {
                        return first.get().getValue();
                    } else {
                        return "";
                    }
                }).collect(Collectors.toList());
                String[] fieldCodes = ichampFieldCode.split(",");
                resultData.put(fieldCodes[0], String.join(",", ibankIds));
                if (fieldCodes.length > 1) {
                    resultData.put(fieldCodes[1], String.join(",", userNames));
                }
            } else {
                resultData.put(ichampFieldCode, String.join(",", values));
            }
        }
    }

    private DictMapping getDictMapping(String itsmFieldCode) {
        Map<String, DictMapping> dictMapping = extParam.getDictMapping();
        if (dictMapping.containsKey(itsmFieldCode)) {
            return dictMapping.get(itsmFieldCode);
        }
        return DictMapping.builder().idCode("id").tableCode("data_dict_detail").labelCode("label").build();

    }


    public String queryDictLabel(List<String> values, DictMapping dictMapping) {
        List<String> labels = dictMapper.queryLabelByValues(values, dictMapping.getTableCode(), dictMapping.getIdCode(), dictMapping.getLabelCode());
        return String.join(",", labels);
    }

    private void fillIChampValueWithMember(String ichampFieldCode, String userId, ObjectNode resultData) {
        if (CharSequenceUtil.isNotBlank(userId)) {
            List<UserInfo> userListByIds = userSSOClient.getUserListByIds(Lists.newArrayList(userId), "2", "110");
            List<String> userNames = userListByIds.stream().map(UserInfo::getName).collect(Collectors.toList());
            List<String> ibankIds = userListByIds.stream().map(item -> {
                Optional<BaseExtend> first = item.getExtend().stream().filter(user -> user.getAlias().equalsIgnoreCase("1bankId")).findFirst();
                if (first.isPresent()) {
                    return first.get().getValue();
                } else {
                    return "";
                }
            }).collect(Collectors.toList());
            String[] fieldCodes = ichampFieldCode.split(",");
            resultData.put(fieldCodes[0], String.join(",", ibankIds));
            if (fieldCodes.length > 1) {
                resultData.put(fieldCodes[1], String.join(",", userNames));
            }
        } else {
            resultData.put(ichampFieldCode, "");
        }
    }

    private void fillIChampValueWithGroup(String ichampFieldCode, ObjectNode formData, String itsmFieldCode, ObjectNode resultData, String type) {
        if ("GROUP".equals(type)) {
            JsonNode itsmValue = formData.get(itsmFieldCode);
            List<String> values = new ArrayList<>();
            boolean isUser = false;
            for (JsonNode jsonNode : itsmValue) {
                if (jsonNode instanceof ObjectNode) {
                    JsonNode userId = jsonNode.get("userId");
                    JsonNode groupName = jsonNode.get("groupName");
                    if (userId != null && !(userId instanceof NullNode) && StringUtils.isNotBlank(userId.asText()) && !"null".equalsIgnoreCase(userId.asText())) {
                        isUser = true;
                        values.add(userId.asText());
                    } else if (groupName != null && !(groupName instanceof NullNode) && StringUtils.isNotBlank(groupName.asText()) && !"null".equalsIgnoreCase(groupName.asText())) {
                        values.add(groupName.asText());
                    }
                } else {
                    values.add(jsonNode.asText());
                }
            }
            if (isUser) {
                List<UserInfo> userListByIds = userSSOClient.getUserListByIds(values, "2", "110");
                List<String> userNames = userListByIds.stream().map(UserInfo::getName).collect(Collectors.toList());
                List<String> ibankIds = userListByIds.stream().map(item -> {
                    Optional<BaseExtend> first = item.getExtend().stream().filter(user -> user.getAlias().equalsIgnoreCase("1bankId")).findFirst();
                    if (first.isPresent()) {
                        return first.get().getValue();
                    } else {
                        return "";
                    }
                }).collect(Collectors.toList());
                String[] fieldCodes = ichampFieldCode.split(",");
                resultData.put(fieldCodes[0], String.join(",", ibankIds));
                if (fieldCodes.length > 1) {
                    resultData.put(fieldCodes[1], String.join(",", userNames));
                }
            } else {
                resultData.put(ichampFieldCode, String.join(",", values));
            }
        }
    }


}
