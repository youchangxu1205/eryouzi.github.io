package com.cloudwise.douc.customization.biz.service.ichampsync;

import cn.hutool.core.date.DateUtil;
import cn.hutool.core.io.FileUtil;
import cn.hutool.core.io.resource.ClassPathResource;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.cloudwise.dosm.api.bean.utils.JsonUtils;
import com.cloudwise.douc.customization.biz.dao.DbsApiLogMapper;
import com.cloudwise.douc.customization.biz.dao.DictMapper;
import com.cloudwise.douc.customization.biz.dao.MdlApproveRecordDetailMapper;
import com.cloudwise.douc.customization.biz.dao.MdlApproveRecordMapper;
import com.cloudwise.douc.customization.biz.dao.MdlInstanceMapper;
import com.cloudwise.douc.customization.biz.dao.MdlInstanceTableDataMapper;
import com.cloudwise.douc.customization.biz.dao.SignOffMapper;
import com.cloudwise.douc.customization.biz.dao.UploadFileMapper;
import com.cloudwise.douc.customization.biz.facade.UserSSOClient;
import com.cloudwise.douc.customization.biz.facade.user.BaseExtend;
import com.cloudwise.douc.customization.biz.facade.user.UserInfo;
import com.cloudwise.douc.customization.biz.model.email.dosm.ApproveResultEnum;
import com.cloudwise.douc.customization.biz.model.log.DbsApiLog;
import com.cloudwise.douc.customization.biz.model.signoff.SignOffEntity;
import com.cloudwise.douc.customization.biz.model.table.MdlApproveRecord;
import com.cloudwise.douc.customization.biz.model.table.MdlApproveRecordDetail;
import com.cloudwise.douc.customization.biz.model.table.MdlInstance;
import com.cloudwise.douc.customization.biz.model.table.MdlInstanceTableData;
import com.cloudwise.douc.customization.common.config.DbsProperties;
import com.cloudwise.storage.FileStorageService;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.JsonNodeType;
import com.fasterxml.jackson.databind.node.NullNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.stereotype.Component;

import java.nio.charset.Charset;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;

import static com.cloudwise.douc.customization.biz.service.ichampsync.SignoffConstants.SignoffItem.IDR_SIGNOFF_PROCUTOVER;

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
    private final SignOffMapper signOffMapper;

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

    public DataSynchronizer(MdlInstanceMapper mdlInstanceMapper, MdlInstanceTableDataMapper mdlInstanceTableDataMapper, SignOffMapper signOffMapper, MdlApproveRecordMapper mdlApproveRecordMapper, MdlApproveRecordDetailMapper mdlApproveRecordDetailMapper, UserSSOClient userSSOClient, DbsApiLogMapper dbsApiLogMapper, DbsProperties dbsProperties, DictMapper dictMapper, FileStorageService fileStorageService, UploadFileMapper uploadFileMapper) {
        this.mdlInstanceMapper = mdlInstanceMapper;
        this.mdlInstanceTableDataMapper = mdlInstanceTableDataMapper;
        this.signOffMapper = signOffMapper;
        this.mdlApproveRecordMapper = mdlApproveRecordMapper;
        this.mdlApproveRecordDetailMapper = mdlApproveRecordDetailMapper;
        this.userSSOClient = userSSOClient;
        this.dbsApiLogMapper = dbsApiLogMapper;
        this.dbsProperties = dbsProperties;
        this.dictMapper = dictMapper;
        this.fileStorageService = fileStorageService;
        this.uploadFileMapper = uploadFileMapper;
    }


    public void send2Ichamp(ObjectNode resultData, boolean isNew, MdlInstance mdlInstance, ObjectNode formData) {
        JsonNode lastRequestor = formData.get("LastRequestor");
        String userId = mdlInstance.getCreatedBy();
        if (!lastRequestor.isEmpty()) {
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
        HttpResponse response = HttpUtil.createPost(url).header("Content-Type", "application/json").header("appCode", dbsProperties.getAppCode()).header("appKey", dbsProperties.getAppKey()).body(jsonString).execute();
        String body = response.body();
        log.info("CrStatusSyncTriggermakeResult:{}", body);
        JsonNode node = JsonUtils.parseJsonNode(body);
        DbsApiLog dbsApiLog = new DbsApiLog();
        dbsApiLog.setCreatedTime(new Date());
        dbsApiLog.setRequestStatus(true);
        dbsApiLog.setRequestUrl(url);
        dbsApiLog.setRequestBody(jsonString);
        dbsApiLog.setResponseBody(body);
        dbsApiLog.setWorkOrderNo(mdlInstance.getBizKey());
        dbsApiLog.setApiModule("crStatusSync");
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


        List<MdlInstanceTableData> mdlInstanceTableData = mdlInstanceTableDataMapper.selectList(Wrappers.lambdaQuery(MdlInstanceTableData.class).eq(MdlInstanceTableData::getId, workOrderId));
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
                cmcrollbackduration = (int) ((maxDate - minDate) / (1000 * 60 * 60 * 24));
            }

            formData.put("cmcrollbackduration", cmcrollbackduration);
        } else {
            log.error("CrStatusSyncTrigger: implementInfo not found");
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


        List<SignOffEntity> signOffEntities = signOffMapper.selectList(Wrappers.lambdaQuery(SignOffEntity.class).eq(SignOffEntity::getWorkOrderId, workOrderId));
        for (SignOffEntity signoffManager : signOffEntities) {
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
                                caseIdsResult.add(caseIdsResult);
                            }
                        }
                        resultData.put("idrsignoffcaseid", caseIdsResult);
                    }
                    JsonNode signOffUser = JsonUtils.parseJsonNode(signoffManager.getSignOffUser());
                    if (signOffUser != null && !signOffUser.isEmpty()) {
                        if (signoffItem.getSignOffApproverLoginCode() != null) {
                            String userId = signOffUser.get(0).get("userId").asText();
                            UserInfo userInfo = userSSOClient.getUserById(userId);
                            List<BaseExtend> extend = userInfo.getExtend();
                            extend.stream().filter(item -> item.getAlias().equalsIgnoreCase("1bankid")).findFirst().ifPresent(item -> {
                                String ibankId = item.getValue();
                                resultData.put(signoffItem.getSignOffApproverLoginCode(), ibankId);
                            });
                        }
                        SignOffStatusEnum signOffStatusEnum = SignOffStatusEnum.valueOf(signoffManager.getStatus());
                        if (signOffStatusEnum == SignOffStatusEnum.REJECTED || signOffStatusEnum == SignOffStatusEnum.APPROVED) {
                            if (StringUtils.isNotBlank(signoffItem.getSignOffStatusCode())) {
                                resultData.put(signoffItem.getSignOffStatusCode(), signOffStatusEnum.getDesc());
                            }
                            if (StringUtils.isNotBlank(signoffItem.getSignOffRejectionReasonCode())) {
                                resultData.put(signoffItem.getSignOffRejectionReasonCode(), signoffManager.getRejectionReason() == null ? "" : signoffManager.getRejectionReason());
                            }
                            if (StringUtils.isNotBlank(signoffItem.getSignOffUrlCode())) {
                                JsonNode node = JsonUtils.parseJsonNode(signoffManager.getArtifact());
                                List<String> urls = new ArrayList<>();
                                for (JsonNode jsonNode : node) {
                                    JsonNode id = jsonNode.get("id");

                                }
                                formData.put(signoffItem.getSignOffUrlCode(), String.join(",", urls));
                            }
                        } else {
                            if (StringUtils.isNotBlank(signoffItem.getSignOffStatusCode())) {
                                handleEmptyField(signoffItem.getSignOffStatusCode(), resultData);
                            }
                            if (StringUtils.isNotBlank(signoffItem.getSignOffRejectionReasonCode())) {
                                handleEmptyField(signoffItem.getSignOffRejectionReasonCode(), resultData);
                            }
                            if (StringUtils.isNotBlank(signoffItem.getSignOffUrlCode())) {
                                JsonNode node = JsonUtils.parseJsonNode(signoffManager.getArtifact());
                                List<String> urls = new ArrayList<>();
                                for (JsonNode jsonNode : node) {
                                    JsonNode id = jsonNode.get("id");
                                    //
                                }
                                formData.put(signoffItem.getSignOffUrlCode(), String.join(",", urls));
                            }
                        }

                    }
                }
            }
        }
        //查询审批数据
        List<ApproveInfo> approveInfoMapping = extParam.getApproveInfoMapping();
        for (ApproveInfo approveInfo : approveInfoMapping) {
            //if cr Status is not New ,need query approver status
            LambdaQueryWrapper<MdlApproveRecord> wrapper = Wrappers.lambdaQuery(MdlApproveRecord.class).eq(MdlApproveRecord::getWorkOrderId, workOrderId).eq(MdlApproveRecord::getNodeId, approveInfo.getNodeId()).orderByDesc(MdlApproveRecord::getCreatedTime);
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
                            formData.put(approveInfo.getRejectionCode(), dataDictLableById);
                        }

                        if (StringUtils.isNotBlank(approveInfo.getStatusTimeCode())) {
                            formData.put(approveInfo.getStatusTimeCode(), DateUtil.formatDateTime(mdlApproveRecord.getUpdatedTime()));
                        }
                        resultData.put(approveInfo.getReasonCode(), mdlApproveRecordDetail.getApproveMsg());
                    }
                    resultData.put(approveInfo.getStatusCode(), mdlApproveRecord.getApproveResult() == ApproveResultEnum.PASS ? "Approved" : "Rejected");
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
                            if (StringUtils.isNotBlank(dictId) && StringUtils.isNotBlank(approveInfo.getReasonCode())) {
                                String dataDictLableById = mdlApproveRecordMapper.getDataDictLableById(dictId);
                                formData.put(approveInfo.getRejectionCode(), dataDictLableById);
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

        send2Ichamp(resultData, isNew, mdlInstance, formData);
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
        if (formData.has(itsmFieldCode)) {
            JsonNode itsmValue = formData.get(itsmFieldCode);
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
                    fillIChamValueWithArrary(ichampFieldCode, formData, itsmFieldCode, resultData);
                    break;
                case BOOLEAN:
                    resultData.put(ichampFieldCode, itsmValue.asBoolean());
                    break;
                case OBJECT:
                    if (itsmValue.has("startDate")) {
                        String[] startAndEnd = ichampFieldCode.split(",");
                        resultData.put(startAndEnd[0], DateUtil.format(new Date(itsmValue.get("startDate").asLong()), "yyyy-MM-dd HH:mm:ss"));
                        resultData.put(startAndEnd[1], DateUtil.format(new Date(itsmValue.get("endDate").asLong()), "yyyy-MM-dd HH:mm:ss"));
                    } else {
                        log.error("not support");
                    }
                    break;
                default:
                    if (formData.has(itsmFieldCode + "_value")) {
                        String label = formData.get(itsmFieldCode + "_value").asText();
                        String text = itsmValue.asText();
                        if (StringUtils.isNotBlank(text) && StringUtils.isBlank(label)) {
                            resultData.put(ichampFieldCode, queryDictLabel(Collections.singletonList(text), getDictMapping(itsmFieldCode)));
                        } else {
                            resultData.put(ichampFieldCode, label);
                        }

                    } else {
                        resultData.put(ichampFieldCode, itsmValue.asText());
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
        if (formData.has(itsmFieldCode + "_value")) {
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

}
