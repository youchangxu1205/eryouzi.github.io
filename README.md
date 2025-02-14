package com.cloudwise.dosm.dbs.trigger;

import cn.hutool.core.collection.CollectionUtil;
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.text.CharSequenceUtil;
import cn.hutool.core.util.RandomUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.http.HttpRequest;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.v2.ITriggerExtV2;
import com.cloudwise.dosm.api.bean.form.FieldInfo;
import com.cloudwise.dosm.api.bean.form.RowData;
import com.cloudwise.dosm.api.bean.form.enums.FieldValueTypeEnum;
import com.cloudwise.dosm.api.bean.form.field.CheckboxField;
import com.cloudwise.dosm.api.bean.form.field.DateField;
import com.cloudwise.dosm.api.bean.form.field.DateRangeField;
import com.cloudwise.dosm.api.bean.form.field.FieldValue;
import com.cloudwise.dosm.api.bean.form.field.GroupField;
import com.cloudwise.dosm.api.bean.form.field.InputField;
import com.cloudwise.dosm.api.bean.form.field.MemberField;
import com.cloudwise.dosm.api.bean.form.field.MultiSelectField;
import com.cloudwise.dosm.api.bean.form.field.RadioField;
import com.cloudwise.dosm.api.bean.form.field.SelectField;
import com.cloudwise.dosm.api.bean.form.field.SelectManyField;
import com.cloudwise.dosm.api.bean.form.field.TableFormField;
import com.cloudwise.dosm.api.bean.form.field.TextareaField;
import com.cloudwise.dosm.api.bean.form.field.TimeField;
import com.cloudwise.dosm.api.bean.instance.entity.WorkOrderBean;
import com.cloudwise.dosm.api.bean.trigger.entity.ResultBean;
import com.cloudwise.dosm.api.bean.trigger.entity.TriggerContext;
import com.cloudwise.dosm.api.bean.utils.JsonUtils;
import com.cloudwise.dosm.biz.instance.dao.MdlInstanceMapper;
import com.cloudwise.dosm.biz.instance.entity.MdlInstance;
import com.cloudwise.dosm.bpm.base.dao.MdlApproveRecordDetailMapper;
import com.cloudwise.dosm.bpm.base.dao.MdlApproveRecordMapper;
import com.cloudwise.dosm.bpm.base.entity.MdlApproveRecord;
import com.cloudwise.dosm.bpm.base.entity.MdlApproveRecordDetail;
import com.cloudwise.dosm.bpm.base.enums.ApproveResultEnum;
import com.cloudwise.dosm.bpm.base.enums.ApproveStatusEnum;
import com.cloudwise.dosm.core.config.NacosConfigLocalCatch;
import com.cloudwise.dosm.core.constant.IDosmConstant;
import com.cloudwise.dosm.core.pojo.bo.RequestDomain;
import com.cloudwise.dosm.core.pojo.po.BasePo;
import com.cloudwise.dosm.core.utils.UserHolder;
import com.cloudwise.dosm.dict.dao.DataDictDetailMapper;
import com.cloudwise.dosm.dict.dao.DataDictMapper;
import com.cloudwise.dosm.dict.entity.DataDict;
import com.cloudwise.dosm.dict.entity.DataDictDetail;
import com.cloudwise.dosm.douc.base.Constant;
import com.cloudwise.dosm.douc.entity.BaseExtend;
import com.cloudwise.dosm.douc.entity.user.UserInfo;
import com.cloudwise.dosm.douc.service.UserService;
import com.cloudwise.dosm.douc.util.UserUtil;
import com.cloudwise.dosm.facewall.extension.base.startup.util.SpringContextUtils;
import com.cloudwise.douc.dto.v3.user.UserConditionReq;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.google.common.collect.Lists;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.jetbrains.annotations.NotNull;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.env.Environment;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;

/**
 * <p>
 *
 * </p>
 *
 * @author Norval.Xu
 * @since 2024/12/12
 */
@Slf4j
@ExtendConfig(id = "cr-status-sync-trigger", name = "CR Status Sync to IChamp", desc = "CR Status Sync to IChamp")
public class CrStatusSyncTrigger implements ITriggerExtV2 {

    private static final String CSH = "custom.service.host";
    private static final String CSHPRE = "custom.service.host.prefix";


    private static final String PATH_SEND_MSG = "/dcs/custom/api/sendMessage";

    @Autowired
    UserService userService;

    @Autowired
    private NacosConfigLocalCatch nacosConfigLocalCatch;
    @Autowired
    private MdlApproveRecordDetailMapper mdlApproveRecordDetailMapper;
    @Autowired
    private MdlApproveRecordMapper mdlApproveRecordMapper;
    @Autowired
    private DataDictDetailMapper dataDictDetailMapper;

    @Override
    public void handler(TriggerContext ctx) {
        String extParam = ctx.getExtParam();
        ExtParam param = JsonUtils.parseObject(extParam, ExtParam.class);
        WorkOrderBean workOrder = ctx.getWorkOrder();
        String originMessage = ctx.getOriginMessage();
        JsonNode originMessageNode = JsonUtils.parseJsonNode(originMessage);
        if (originMessageNode == null) {
            log.error("origin message error:{}", ctx.getOriginMessage());
            throw new IllegalArgumentException("origin message error");
        }

        //判断是否为第一次提交
        CrStatusEnums crStatus = getCrStatus(ctx, param, workOrder);
        if (crStatus == null) {
            return;
        }
        updateCrStatus2Form(ctx, crStatus, param);
        HttpResponse response = null;
        try {
            List<UserInfo> createdBys = getUsers(Lists.newArrayList(Long.valueOf(workOrder.getCreatedBy())));

            String makeParam = makeParam(ctx, param, workOrder, createdBys, crStatus, originMessageNode);
            log.info("CrStatusSyncTriggermakeParam:{}", makeParam);
            String appCode = getConfig("dosm.custom.appCode");
            String appKey = getConfig("dosm.custom.appKey");
            boolean isNewCr = isNewCr(originMessageNode, param);
            response = HttpUtil.createPost(isNewCr ? param.getPushUrl() : param.getUpdateUrl()).header("Content-Type", "application/json").header("appCode", appCode).header("appKey", appKey).body(makeParam).execute();
            String body = response.body();
            log.info("CrStatusSyncTriggermakeResult:{}", body);
            JsonNode node = JsonUtils.parseJsonNode(body);
            if ("200".equals(node.get("code").asText())) {
                JsonNode data = node.get("data");
                JsonNode status = data.get("Status");
                JsonNode message = data.get("Message");
                if ("FAILED".equals(status.asText())) {
                    ResultBean resultBean = ctx.getResult();
                    resultBean.setError(message.asText());
                    ctx.setResult(resultBean);
                }
            } else {
                ResultBean resultBean = ctx.getResult();
                resultBean.setError(body);
                ctx.setResult(resultBean);
            }

            sendApprovalMsg(crStatus, ctx, createdBys);
        } catch (Exception e) {
            log.error("CrStatusSyncTrigger send to ichamp error:{}", e.getMessage(), e);
            ResultBean resultBean = ctx.getResult();
            resultBean.setError("Sync data to ichamp fail: data:" + e.getMessage());
            ctx.setResult(resultBean);
        } finally {
            if (response != null) {
                response.close();
            }
        }
    }

    public static String getConfig(String configKey) {
        Environment bean = com.cloudwise.dosm.core.utils.SpringContextUtils.getBean(Environment.class);
        return bean.getProperty(configKey);
    }

    private String makeParam(TriggerContext ctx, ExtParam extParam, WorkOrderBean workOrder, List<UserInfo> createdBys, CrStatusEnums crStatus, JsonNode originMessageNode) {
        ObjectNode result = JsonUtils.createObjectNode();
        ObjectNode data = JsonUtils.createObjectNode();
        List<String> userOneBankIds = getUserOneBankIdsByUser(createdBys);
        ObjectNode formData = JsonUtils.createObjectNode();
        makeFormInfo(formData, ctx, workOrder, extParam, originMessageNode);
        JsonNode nodeIdNode = originMessageNode.get("nodeId");
        result.put("action", CrStatusEnums.NEW.equals(crStatus) ? "Create" : "Modify");
        if (!nodeIdNode.asText().equals(extParam.getCreateNodeId())) {
            formData.put("state", crStatus.getLabel());
        } else {
            formData.put("requester", userOneBankIds.get(0));
        }
        formData.put("changerequestor", userOneBankIds.get(0));
        data.set("formData", formData);
        data.put("workOrderNo", workOrder.getBizKey());
        data.put("createdBy", userOneBankIds.get(0));
        data.put("createdDate", workOrder.getCreatedTime().getTime());
        data.put("workOrderStatus", crStatus.getLabel());
        result.set("data", data);
        return JsonUtils.toJsonString(result);
    }

    public boolean isNewCr(JsonNode originMessageNode, ExtParam extParam) {
        JsonNode nodeIdNode = originMessageNode.get("nodeId");
        String nodeId = nodeIdNode.asText();
        JsonNode eventTypeNode = originMessageNode.get("eventType");
        String eventType = eventTypeNode.asText();
        if (nodeId.equals(extParam.getCreateNodeId()) && "WORK_COMMIT".equals(eventType)) {
            return true;
        }
        return false;
    }

    private static CrStatusEnums getCrStatus(TriggerContext ctx, ExtParam extParam, WorkOrderBean workOrder) {
        String originMessage = ctx.getOriginMessage();
        JsonNode originMessageNode = JsonUtils.parseJsonNode(originMessage);
        if (originMessageNode == null) {
            log.error("origin message error:{}", ctx.getOriginMessage());
            throw new IllegalArgumentException("origin message error");
        }
        JsonNode eventType = originMessageNode.get("eventType");
        JsonNode nodeIdNode = originMessageNode.get("nodeId");
        String nodeId = nodeIdNode.asText();
        JsonNode rollbackToNodeId = originMessageNode.get("rollbackToNodeId");
        log.info("CrStatusSyncTrigger:eventType:{},nodeId:{},rollbackNodeId:{},workOrder:{}", eventType, nodeIdNode, rollbackToNodeId, workOrder);

        /**
         * 1.Determine if the work order is submitted
         *  1.1 Determine if it is the work order creation node
         *      1.1.1 If it is determined to be Reopen, set the status to Reopen
         *      1.1.2 If it is determined to be a new work order, set the status to New
         *  1.2 Determine if it is the CR closure node, the status remains unchanged
         *  1.3 Determine if it is the signOff node, set the status to Open
         * 2.Determine if it is entering the implementation node, set the status to Approved
         * 3.Determine if it is the signOff node retrieval, set the status to Closed Cancel
         * 4.Determine if it is the signOff node rollback, set the status to Rejected
         */
        if ("WORK_COMMIT".equals(eventType.asText())) {
            if (nodeId.equals(extParam.getCreateNodeId())) {
                Map<String, FieldInfo> formDataMap = workOrder.getFormDataMap();
                if (formDataMap.containsKey(extParam.getCrStatusFieldCode())) {
                    FieldInfo fieldInfo = formDataMap.get(extParam.getCrStatusFieldCode());
                    SelectField fieldValueObj = (SelectField) fieldInfo.getFieldValueObj();
                    String label = fieldValueObj.getLabel();
                    if (label.equals("Closed Cancel") || label.equals("Rejected")) {
                        return CrStatusEnums.REOPEN;
                    }
                } else {
                    return CrStatusEnums.NEW;
                }

            } else if (nodeId.equals(extParam.getCloseNodeId())) {
                Map<String, FieldInfo> formDataMap = workOrder.getFormDataMap();
                FieldInfo fieldInfo = formDataMap.get(extParam.getCloseStatusFieldCode());
                SelectField fieldValueObj = (SelectField) fieldInfo.getFieldValueObj();
                return CrStatusEnums.getByLabel(fieldValueObj.getLabel());
            } else if (nodeId.equals(extParam.getSignoffNodeId())) {
                return CrStatusEnums.OPEN;
            }
        } else if ("NODE_ENTER".equals(eventType.asText()) && (nodeId.equals(extParam.getImplementationNodeId()) || nodeId.equals(extParam.getCloseNodeId()))) {
            return CrStatusEnums.APPROVED;
        } else if ("WORK_GET_BACK".equals(eventType.asText()) && rollbackToNodeId != null && rollbackToNodeId.asText().equals(extParam.getSignoffNodeId())) {
            return CrStatusEnums.CLOSED_CANCEL;
        } else if ("WORK_ROLLBACK".equals(eventType.asText()) && rollbackToNodeId != null && rollbackToNodeId.asText().equals(extParam.getSignoffNodeId())) {
            return CrStatusEnums.REJECTED;
        }
        return null;
    }

    /**
     * 更新crStatus至form表单
     *
     * @param ctx
     * @param crStatus
     * @param param
     */
    private void updateCrStatus2Form(TriggerContext ctx, CrStatusEnums crStatus, ExtParam param) {
        MdlInstanceMapper mdlInstanceMapper = SpringContextUtils.getBean(MdlInstanceMapper.class);
        MdlInstance mdlInstance = mdlInstanceMapper.selectById(ctx.getWorkOrder().getId(), ctx.getAccountId());
        Object data = mdlInstance.getFormData();
        ObjectNode formData = (ObjectNode) JsonUtils.parseJsonNode(data);

        DataDictDetailMapper dataDictDetailMapper = SpringContextUtils.getBean(DataDictDetailMapper.class);
        DataDictMapper dataDictMapper = SpringContextUtils.getBean(DataDictMapper.class);
        DataDict dataDict = new DataDict();
        dataDict.setIsDel(0);
        dataDict.setDictCode(param.getCrStatusDictCode());
        DataDict crStatusDataDict = dataDictMapper.selectOneByParam(dataDict);
        List<DataDictDetail> dataDictDetails = dataDictDetailMapper.selectDetailByDictIdAndLevel(crStatusDataDict.getId(), 1, crStatusDataDict.getAccountId());
        Optional<DataDictDetail> first = dataDictDetails.stream().filter(item -> item.getData().equals(crStatus.getLabel())).findFirst();
        if (first.isPresent()) {
            formData.put(param.getCrStatusFieldCode(), first.get().getId());
            formData.put(param.getCrStatusFieldCode() + "_value", first.get().getLabel());
            mdlInstance.setFormData(formData);
            //是否需要记录？
            mdlInstanceMapper.updateByIdSelective(mdlInstance);
        }
    }

    private void makeFormInfo(ObjectNode formData, TriggerContext ctx, WorkOrderBean workOrder, ExtParam extParam, JsonNode originMessageNode) {
        Map<String, FieldInfo> formDataMap = workOrder.getFormDataMap();
        Map<String, String> fieldMapping = extParam.getFieldMapping();
        Set<String> ichampFieldCodes = fieldMapping.keySet();
        for (String ichampFieldCode : ichampFieldCodes) {
            String itsmFieldCode = fieldMapping.get(ichampFieldCode);
            if (formDataMap.containsKey(itsmFieldCode)) {
                FieldInfo fieldInfo = formDataMap.get(itsmFieldCode);
                log.info("CrStatusSyncTriggermakeParam: fieldCode:{},fieldInfo:{}", itsmFieldCode, fieldInfo);
                fillIchampValue(formData, fieldInfo, ichampFieldCode);
            } else {
                log.error("CrStatusSyncTriggermakeParam: fieldCode:{},noFieldInfo", itsmFieldCode);
                String[] startAndEndCode = ichampFieldCode.split(",");
                for (String fieldCode : startAndEndCode) {
                    formData.put(fieldCode, "");
                }
            }
        }
        //Special handling
        if (formDataMap.containsKey("F_Reversion_Plan")) {
            FieldInfo implementInfo = formDataMap.get("F_Reversion_Plan");
            TableFormField implementInfoFieldValueObj = (TableFormField) implementInfo.getFieldValueObj();
            Collection<RowData> implementInfoFieldValueObjValue = implementInfoFieldValueObj.getValue();
            int cmcrollbackduration = 0;
            Long minDate = null;
            Long maxDate = null;
            for (RowData rowData : implementInfoFieldValueObjValue) {
                Map<String, FieldInfo> columnDataMap = rowData.getColumnDataMap();
                FieldInfo startTimeFieldInfo = columnDataMap.get("F_Start_Time");
                DateField startTime = (DateField) startTimeFieldInfo.getFieldValueObj();
                if (minDate == null) {
                    minDate = startTime.getValue();
                } else {
                    minDate = Math.min(minDate, startTime.getValue());
                }
                FieldInfo endTimeFieldInfo = columnDataMap.get("F_End_Time");
                DateField endTime = (DateField) endTimeFieldInfo.getFieldValueObj();
                if (maxDate == null) {
                    maxDate = endTime.getValue();
                } else {
                    maxDate = Math.max(maxDate, endTime.getValue());
                }
            }
            if (minDate != null && maxDate != null) {
                cmcrollbackduration = (int) ((maxDate - minDate) / (1000 * 60 * 60 * 24));
            }

            formData.put("cmcrollbackduration", cmcrollbackduration);
        } else {
            log.error("CrStatusSyncTrigger: implementInfo not found");
        }
        Map<String, String> riskDictMapping = extParam.getRiskDictMapping();
        /**
         * {
         *     "CustomerServices":"d7b6d17cf02c401a93144a6323915745",
         *     "InherentandResidualrisks":"f831cb01fd0f43b297a93ead68abc562",
         *     "BusinessUnitsOperations":"2008e9b96d5d413798d119b829648535",
         *     "ChangeComplexity":"845a783c49524348bd5804cfbc0ac06e",
         *     "BackoutComplexity":"c7199dacb70e4a07ad29dd898cf56860",
         *     "Documentation":"ff30d63471b14bc3870b90c849c66f7d",
         *     "Security":"3240ef33f1bd4dbcbb1ad5ac86bb0b53",
         *     "Interfaces":"aea98f47781147f8a8cb8eae078f7299",
         * }
         */
        setRiskValue(formData, formDataMap, "CustomerServices", riskDictMapping.get("CustomerServices"), "cmrmavailablity");
        setRiskValue(formData, formDataMap, "InherentandResidualrisks", riskDictMapping.get("InherentandResidualrisks"), "cmrminherentresidualrisks");
        setRiskValue(formData, formDataMap, "BusinessUnitsOperations", riskDictMapping.get("BusinessUnitsOperations"), "cmrmcontinuity");
        setRiskValue(formData, formDataMap, "ChangeComplexity", riskDictMapping.get("ChangeComplexity"), "cmrmcomplexity");
        setRiskValue(formData, formDataMap, "BackoutComplexity", riskDictMapping.get("BackoutComplexity"), "cmrmimplement");
        setRiskValue(formData, formDataMap, "Documentation", riskDictMapping.get("Documentation"), "cmrmtraining");
        setRiskValue(formData, formDataMap, "Security", riskDictMapping.get("Security"), "cmrmsecurity");
        setRiskValue(formData, formDataMap, "Interfaces", riskDictMapping.get("Interfaces"), "cmrminterfaces");

        //mainframedata
        Map<String, List<TableFieldMapping>> tableFieldMappings = extParam.getTableFieldMapping();
        for (Map.Entry<String, List<TableFieldMapping>> item : tableFieldMappings.entrySet()) {
            String key = item.getKey();
            ObjectNode itemResult = JsonUtils.createObjectNode();

            List<TableFieldMapping> itemValue = item.getValue();
            for (TableFieldMapping tableFieldMapping : itemValue) {
                String itsmTableFieldInfo = tableFieldMapping.getItsmTableFieldInfo();
                FieldInfo jobDetailsSection = formDataMap.get(itsmTableFieldInfo);
                if (jobDetailsSection != null) {
                    ArrayNode jobDetailResult = JsonUtils.createArrayNode();
                    TableFormField jobDetailsSectionField = new TableFormField();
                    Collection<RowData> jobDetailsSectionFieldValue = jobDetailsSectionField.getValue();
                    Map<String, String> fieldMapping1 = tableFieldMapping.getFieldMapping();
                    for (RowData rowData : jobDetailsSectionFieldValue) {
                        ObjectNode jobDetailItem = JsonUtils.createObjectNode();
                        Map<String, FieldInfo> columnDataMap = rowData.getColumnDataMap();
                        for (Map.Entry<String, String> stringStringEntry : fieldMapping1.entrySet()) {
                            FieldInfo fieldValueInfo = columnDataMap.get(stringStringEntry.getValue());
                            fillIchampValue(jobDetailItem, fieldValueInfo, stringStringEntry.getKey());
                            jobDetailResult.add(jobDetailItem);
                        }
                    }
                    itemResult.put(tableFieldMapping.getIchampTableFieldInfo(), jobDetailResult);
                }
            }
            if (!itemResult.isEmpty()) {
                formData.put(key, itemResult);
            }
        }
        List<ApproveInfo> approveInfoMapping = extParam.getApproveInfoMapping();
        for (ApproveInfo approveInfo : approveInfoMapping) {
            //if cr Status is not New ,need query approver status
            LambdaQueryWrapper<MdlApproveRecord> wrapper = Wrappers.lambdaQuery(MdlApproveRecord.class)
                    .eq(MdlApproveRecord::getWorkOrderId, workOrder.getId())
                    .eq(MdlApproveRecord::getNodeId, approveInfo.getNodeId()).orderByDesc(BasePo::getCreatedTime);
            MdlApproveRecord mdlApproveRecord = mdlApproveRecordMapper.selectOne(wrapper, false);
            log.error("CrStatus:queryApprove:{}", mdlApproveRecord);
            if (mdlApproveRecord != null) {
                if (mdlApproveRecord.getApproveStatus().equals(ApproveStatusEnum.FINISHED.getCode() + "")) {
                    MdlApproveRecordDetail mdlApproveRecordDetail = mdlApproveRecordDetailMapper.selectOne(Wrappers.lambdaQuery(MdlApproveRecordDetail.class)
                            .eq(MdlApproveRecordDetail::getApproveRecordId, mdlApproveRecord.getId()), false);
                    log.error("CrStatus:mdlApproveRecordDetail:{}", mdlApproveRecordDetail);
                    if (mdlApproveRecordDetail != null) {
                        String dictId = mdlApproveRecordDetail.getReason();
                        if (StringUtils.isNotBlank(dictId) && StringUtils.isNotBlank(approveInfo.getReasonCode())) {
                            DataDictDetail dataDictDetail = dataDictDetailMapper.selectById(dictId);
                            String label = dataDictDetail.getLabel();
                            formData.put(approveInfo.getRejectionCode(), label);
                        }
                        formData.put(approveInfo.getReasonCode(), mdlApproveRecordDetail.getApproveMsg());
                    }
                    formData.put(approveInfo.getStatusCode(), mdlApproveRecord.getApproveResult() == ApproveResultEnum.PASS ? "Approved" : "Rejected");
                }
            }
        }


        formData.put("itsmcrnumber", workOrder.getBizKey());
        formData.put("implementationplan", "https://" + getConfig("CloudwiseDomain") + "/docp/dosm/dosm/orderDetails?showType=handle&id=" + workOrder.getId() + "&isType=&isShowBack=1&type=3d409c4dc7e741539194a528cd9f0d91");
        formData.put("reversionplan", "https://" + getConfig("CloudwiseDomain") + "/docp/dosm/dosm/orderDetails?showType=handle&id=" + workOrder.getId() + "&isType=&isShowBack=1&type=3d409c4dc7e741539194a528cd9f0d91");

    }

    private void fillIchampValue(ObjectNode formData, FieldInfo fieldInfo, String ichampFieldCode) {
        FieldValueTypeEnum fieldType = fieldInfo.getFieldType();
        switch (fieldType) {
            case RADIO:
                RadioField radioField = (RadioField) fieldInfo.getFieldValueObj();
                String radioFieldLabel = radioField.getLabel();
                formData.put(ichampFieldCode, StrUtil.isBlank(radioFieldLabel) ? "" : radioFieldLabel);
                break;
            case SELECT:
                SelectField fieldValueObj = (SelectField) fieldInfo.getFieldValueObj();
                String label = fieldValueObj.getLabel();
                formData.put(ichampFieldCode, StrUtil.isBlank(label) ? "" : label);
                break;
            case CHECKBOX:
                CheckboxField checkboxField = (CheckboxField) fieldInfo.getFieldValueObj();
                Collection<String> checkboxFieldLabel = checkboxField.getLabel();
                formData.put(ichampFieldCode, CollectionUtil.isEmpty(checkboxFieldLabel) ? "" : String.join(",", checkboxFieldLabel));
                break;
            case INPUT:
                InputField inputField = (InputField) fieldInfo.getFieldValueObj();
                formData.put(ichampFieldCode, StrUtil.isBlank(inputField.getValue()) ? "" : inputField.getValue());
                break;
            case TEXTAREA:
                TextareaField textareaField = (TextareaField) fieldInfo.getFieldValueObj();
                formData.put(ichampFieldCode, StrUtil.isBlank(textareaField.getValue()) ? "" : textareaField.getValue());
                break;
            case SELECT_MANY:
                SelectManyField selectManyField = (SelectManyField) fieldInfo.getFieldValueObj();
                Collection<String> labels = selectManyField.getLabel();
                formData.put(ichampFieldCode, CollectionUtil.isEmpty(labels) ? "" : String.join(",", labels));
                break;
            case MULTI_SELECT:
                MultiSelectField multiSelectField = (MultiSelectField) fieldInfo.getFieldValueObj();
                Collection<String> labelsM = multiSelectField.getLabel();
                formData.put(ichampFieldCode, CollectionUtil.isEmpty(labelsM) ? "" : String.join(",", labelsM));
                break;
            case GROUP:
                GroupField groupField = (GroupField) fieldInfo.getFieldValueObj();
                List<String> arrayNode1 = new ArrayList<>();
                groupField.getValue().forEach(item -> {
                    arrayNode1.add(item.getGroupName());
                });
                formData.put(ichampFieldCode, CollectionUtil.isEmpty(arrayNode1) ? "" : String.join(",", arrayNode1));
                break;
            case MEMBER:
                MemberField memberField = (MemberField) fieldInfo.getFieldValueObj();
                List<String> arrayNode2 = new ArrayList<>();
                List<Long> memberUserIds = new ArrayList<>();
                memberField.getValue().forEach(item -> {
                    arrayNode2.add(item.getUserName());
                    memberUserIds.add(Long.valueOf(item.getUserId()));
                });
                String[] userFieldCodes = ichampFieldCode.split(",");
                List<String> oneBankIds = getUserOneBankIds(memberUserIds);
                formData.put(userFieldCodes[0], CollectionUtil.isEmpty(oneBankIds) ? "" : String.join(",", oneBankIds));
                if (userFieldCodes.length > 1) {
                    formData.put(userFieldCodes[1], CollectionUtil.isEmpty(arrayNode2) ? "" : String.join(",", arrayNode2));
                }
                break;
            case DATERANGE:
                DateRangeField dateRangeField = (DateRangeField) fieldInfo.getFieldValueObj();
                String[] startAndEndCode = ichampFieldCode.split(",");
                if (dateRangeField.getValue() == null) {
                    formData.put(startAndEndCode[0], "");
                    formData.put(startAndEndCode[1], "");
                } else {
                    formData.put(startAndEndCode[0], DateUtil.format(new Date(dateRangeField.getValue().getStartDate()), "yyyy-MM-dd HH:mm:ss"));
                    formData.put(startAndEndCode[1], DateUtil.format(new Date(dateRangeField.getValue().getEndDate()), "yyyy-MM-dd HH:mm:ss"));
                }
                break;
            case DATE:
                DateField dateField = (DateField) fieldInfo.getFieldValueObj();
                formData.put(ichampFieldCode, dateField.getValue() == null ? "" : dateField.getValue().toString());
                break;
            case TIME:
                TimeField timeField = (TimeField) fieldInfo.getFieldValueObj();
                formData.put(ichampFieldCode, timeField.getValue() == null ? "" : timeField.getValue().toString());
                break;
        }
    }

    private static void setRiskValue(ObjectNode formData, Map<String, FieldInfo> formDataMap, String itsmFieldCode, String lowDictId, String ichampFieldCode) {
        FieldInfo fieldInfo = formDataMap.get(itsmFieldCode);
        if (fieldInfo == null) {
            formData.put(ichampFieldCode, "3");
            return;
        }
        FieldValue fieldValueObj = fieldInfo.getFieldValueObj();
        if (fieldValueObj == null || fieldValueObj.getValue() == null) {
            formData.put(ichampFieldCode, "3");
        } else {
            String value = fieldValueObj.getValue().toString();
            if (value.equals(lowDictId)) {
                formData.put(ichampFieldCode, "3");
            } else {
                formData.put(ichampFieldCode, "1");
            }
        }
    }

    @NotNull
    private List<String> getUserOneBankIds(List<Long> memberUserIds) {
        List<UserInfo> userList = getUsers(memberUserIds);
        return getUserOneBankIdsByUser(userList);
    }

    @NotNull
    private List<String> getUserOneBankIdsByUser(List<UserInfo> userList) {
        List<String> oneBankIds = new ArrayList<>();
        userList.forEach(item -> {
            String onebankId = "";
            for (BaseExtend baseExtend : item.getExtend()) {
                if (baseExtend.getAlias().equals("1bankId")) {
                    onebankId = baseExtend.getValue();
                }
            }
            oneBankIds.add(onebankId);
        });
        return oneBankIds;
    }


    @NotNull
    private List<UserInfo> getUsers(List<Long> userIdList) {
        RequestDomain requestDomain = UserHolder.get();
        UserConditionReq condition = new UserConditionReq();
        condition.setAccountId(Long.valueOf(requestDomain.getTopAccountId()));
        condition.setUserId(UserUtil.getUserId());
        condition.setStatus(Constant.DOUC_ENABLE_STATUS);
        condition.setRespMode(3);
        condition.setIds(userIdList);
        condition.setAccountScope(IDosmConstant.ACCOUNT_SCOPE);
        return userService.getUserByConditionV3(condition);
    }

    @Data
    static class ExtParam {
        private String pushUrl;
        private String updateUrl;
        private String pushMethod;
        private String createNodeId;
        private String closeNodeId;
        private String crStatusFieldCode;
        private String crStatusDictCode = "crStatus";
        private String closeStatusFieldCode;
        private String signoffNodeId;
        private String implementationNodeId;
        private Map<String, String> fieldMapping;
        private Map<String, String> riskDictMapping;
        private Map<String, List<TableFieldMapping>> tableFieldMapping;
        private List<ApproveInfo> approveInfoMapping;
    }

    @Data
    static class ApproveInfo {

        private String nodeId;
        private String rejectionCode;
        private String reasonCode;
        private String statusCode;

    }


    @Data
    static class TableFieldMapping {
        private String ichampTableFieldInfo;
        private String itsmTableFieldInfo;
        private Map<String, String> fieldMapping;
    }

    private void sendApprovalMsg(CrStatusEnums crStatus, TriggerContext ctx, List<UserInfo> createdBys) {
        if (crStatus == null || crStatus.getNotifyScene() == null) {
            log.info("workOrderId: {}, crStatus: {}", ctx.getWorkOrder(), crStatus);
            return;
        }

        Map properties = nacosConfigLocalCatch.getProperties();
        if (null == properties) {
            log.info("RejectBtnTrigger get properties is null, workOrderId: {}, crStatus: {}", ctx.getWorkOrder(), crStatus);
            return;
        }
        Object host = properties.get(CSH);
        Object prefix = properties.get(CSHPRE);
        String serviceHost = Optional.ofNullable(host).map(String::valueOf).orElse("");
        String serviceHostPrefix = Optional.ofNullable(prefix).map(String::valueOf).orElse("http://");
        log.info("RejectBtnTrigger custom.service.host:{}", serviceHost);
        if (CharSequenceUtil.isBlank(serviceHost)) {
            log.info("RejectBtnTrigger get properties [custom.service.host] is null");
            return;
        }

        try {
            String url = serviceHostPrefix + getHost(serviceHost) + PATH_SEND_MSG;
            Map<String, Object> sendMsgContextMap = getSendMsgContext(crStatus.getNotifyScene(), ctx, createdBys);
            log.info("RejectBtnTrigger call  url :{},param:{}", url, sendMsgContextMap);
            String body = HttpRequest.post(url).header("Content-Type", "application/json")
                    .body(com.cloudwise.dosm.core.utils.JsonUtils.toJsonString(sendMsgContextMap))
                    .timeout(5000).execute().body();
            log.info("RejectBtnTrigger call  success:{}", body);
        } catch (Exception e) {
            log.error("RejectBtnTrigger call fail:", e);
        }
    }


    private Map<String, Object> getSendMsgContext(String notifyScence, TriggerContext ctx, List<UserInfo> createdBys) {
        Map<String, Object> sendMsgContextMap = new HashMap<>();

        Map<String, Object> notifyMap = new HashMap<>();
        sendMsgContextMap.put("notify", notifyMap);
        notifyMap.put("channelType", "EMAIL");
        notifyMap.put("notifyScene", notifyScence);

        Map<String, String> publicFieldsMap = new HashMap<>();
        sendMsgContextMap.put("publicFields", publicFieldsMap);
        WorkOrderBean workOrder = ctx.getWorkOrder();
        UserInfo createdUserInfo = CollectionUtils.isEmpty(createdBys) ? null : createdBys.get(0);
        // return fieldKey: "title, workOrderId, processInstanceId, orderId, bizDesc, dataStatus, workOrderUri, operator, operatorId, operatorAlias, currentNode, assignName, assignId, assignAlias"
        publicFieldsMap.put("title", workOrder.getTitle());
        publicFieldsMap.put("createdById", workOrder.getCreatedBy());
        publicFieldsMap.put("createdByName", createdUserInfo == null ? "" : createdUserInfo.getName());
        publicFieldsMap.put("createdByEmail", createdUserInfo == null ? "" : createdUserInfo.getName());
        publicFieldsMap.put("workOrderId", workOrder.getId());
        publicFieldsMap.put("processInstanceId", workOrder.getProcessInstanceId());
        publicFieldsMap.put("orderId", workOrder.getId());
        publicFieldsMap.put("bizDesc", workOrder.getBizDesc());
        publicFieldsMap.put("mdlDefKey", workOrder.getMdlDefKey());
        publicFieldsMap.put("bizKey", StringUtils.isBlank(workOrder.getBizKey()) ? workOrder.getId() : workOrder.getBizKey());
        publicFieldsMap.put("currentNode", workOrder.getCurrentNodeId());


        sendMsgContextMap.put("workOrderId", workOrder.getId());
        sendMsgContextMap.put("nodeId", workOrder.getCurrentNodeId());
        sendMsgContextMap.put("createdBy", workOrder.getCreatedBy());
        sendMsgContextMap.put("topAccountId", ctx.getTopAccountId());
        sendMsgContextMap.put("accountId", ctx.getAccountId());
        sendMsgContextMap.put("userId", ctx.getUserId());

        return sendMsgContextMap;
    }


    private String getHost(String ips) {
        String[] split = ips.split(",");
        int idx = RandomUtil.randomInt(0, split.length);
        return split[idx];
    }
}
