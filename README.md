package com.cloudwise.dosm.dbs.trigger;

import cn.hutool.core.collection.CollectionUtil;
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;
import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.v2.ITriggerExtV2;
import com.cloudwise.dosm.api.bean.form.FieldInfo;
import com.cloudwise.dosm.api.bean.form.RowData;
import com.cloudwise.dosm.api.bean.form.enums.FieldValueTypeEnum;
import com.cloudwise.dosm.api.bean.form.field.*;
import com.cloudwise.dosm.api.bean.instance.entity.WorkOrderBean;
import com.cloudwise.dosm.api.bean.trigger.entity.ResultBean;
import com.cloudwise.dosm.api.bean.trigger.entity.TriggerContext;
import com.cloudwise.dosm.api.bean.utils.JsonUtils;
import com.cloudwise.dosm.biz.instance.dao.MdlInstanceMapper;
import com.cloudwise.dosm.biz.instance.entity.MdlInstance;
import com.cloudwise.dosm.core.constant.IDosmConstant;
import com.cloudwise.dosm.core.pojo.bo.RequestDomain;
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
import com.fasterxml.jackson.databind.node.ObjectNode;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.jetbrains.annotations.NotNull;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.env.Environment;

import java.util.*;

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
    @Autowired
    UserService userService;

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
        String crStatus = getCrStatus(ctx, param, workOrder, originMessageNode);
        if (crStatus != null) {
            updateCrStatus2Form(ctx, crStatus, param);
            HttpResponse response = null;
            try {
                String makeParam = makeParam(ctx, param, workOrder, crStatus, originMessageNode);
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
    }

    public static String getConfig(String configKey) {
        Environment bean = com.cloudwise.dosm.core.utils.SpringContextUtils.getBean(Environment.class);
        return bean.getProperty(configKey);
    }

    private String makeParam(TriggerContext ctx, ExtParam extParam, WorkOrderBean workOrder, String crStatus, JsonNode originMessageNode) {
        ObjectNode result = JsonUtils.createObjectNode();
        ObjectNode data = JsonUtils.createObjectNode();
        List<String> userOneBankIds = getUserOneBankIds(Collections.singletonList(Long.valueOf(workOrder.getUpdatedBy())));
        ObjectNode formData = JsonUtils.createObjectNode();
        makeFormInfo(formData, ctx, workOrder, extParam);
        JsonNode nodeIdNode = originMessageNode.get("nodeId");
        result.put("action", "New".equals(crStatus) ? "Create" : "Modify");
        if (!nodeIdNode.asText().equals(extParam.getCreateNodeId())) {
            formData.put("state", crStatus);
        }else{
            formData.put("requester", userOneBankIds.get(0));
        }
        data.set("formData", formData);
        data.put("workOrderNo", workOrder.getBizKey());

        data.put("createdBy", userOneBankIds.get(0));
        data.put("createdDate", workOrder.getCreatedTime().getTime());
        data.put("workOrderStatus", crStatus);
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

    private static String getCrStatus(TriggerContext ctx, ExtParam extParam, WorkOrderBean workOrder, JsonNode originMessageNode) {
        String crStatus = null;
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
                        crStatus = "Reopen";
                    }
                } else {
                    crStatus = "New";
                }

            } else if (nodeId.equals(extParam.getCloseNodeId())) {
                Map<String, FieldInfo> formDataMap = workOrder.getFormDataMap();
                FieldInfo fieldInfo = formDataMap.get(extParam.getCloseStatusFieldCode());
                SelectField fieldValueObj = (SelectField) fieldInfo.getFieldValueObj();
                crStatus = fieldValueObj.getLabel();
            } else if (nodeId.equals(extParam.getSignoffNodeId())) {
                crStatus = "Open";
            }
        } else if ("NODE_ENTER".equals(eventType.asText()) && nodeId.equals(extParam.getImplementationNodeId())) {
            crStatus = "Approved";
        } else if ("WORK_GET_BACK".equals(eventType.asText()) && rollbackToNodeId != null && rollbackToNodeId.asText().equals(extParam.getCreateNodeId())) {
            crStatus = "Closed Cancel";
        } else if ("WORK_ROLLBACK".equals(eventType.asText()) && rollbackToNodeId != null && rollbackToNodeId.asText().equals(extParam.getCreateNodeId())) {
            crStatus = "Rejected";
        }
        return crStatus;
    }

    /**
     * 更新crStatus至form表单
     *
     * @param ctx
     * @param crStatus
     * @param param
     */
    private void updateCrStatus2Form(TriggerContext ctx, String crStatus, ExtParam param) {
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
        Optional<DataDictDetail> first = dataDictDetails.stream().filter(item -> item.getData().equals(crStatus)).findFirst();
        if (first.isPresent()) {
            formData.put(param.getCrStatusFieldCode(), first.get().getId());
            formData.put(param.getCrStatusFieldCode() + "_value", first.get().getLabel());
            mdlInstance.setFormData(formData);
            //是否需要记录？
            mdlInstanceMapper.updateByIdSelective(mdlInstance);
        }
    }

    private void makeFormInfo(ObjectNode formData, TriggerContext ctx, WorkOrderBean workOrder, ExtParam extParam) {
        Map<String, FieldInfo> formDataMap = workOrder.getFormDataMap();
        JsonNode fieldMapping = extParam.getFieldMapping();
        Iterator<String> ichampFieldCodes = fieldMapping.fieldNames();
        while (ichampFieldCodes.hasNext()) {
            String ichampFieldCode = ichampFieldCodes.next();
            String itsmFieldCode = fieldMapping.get(ichampFieldCode).asText();
            if (formDataMap.containsKey(itsmFieldCode)) {
                FieldInfo fieldInfo = formDataMap.get(itsmFieldCode);
                log.info("CrStatusSyncTriggermakeParam: fieldCode:{},fieldInfo:{}", itsmFieldCode, fieldInfo);
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


        formData.put("itsmcrnumber", workOrder.getBizKey());
        formData.put("implementationplan", "https://" + getConfig("CloudwiseDomain") + "/docp/dosm/dosm/orderDetails?showType=handle&id=" + workOrder.getId() + "&isType=&isShowBack=1&type=3d409c4dc7e741539194a528cd9f0d91");
        formData.put("reversionplan", "https://" + getConfig("CloudwiseDomain") + "/docp/dosm/dosm/orderDetails?showType=handle&id=" + workOrder.getId() + "&isType=&isShowBack=1&type=3d409c4dc7e741539194a528cd9f0d91");

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
        RequestDomain requestDomain = UserHolder.get();
        UserConditionReq condition = new UserConditionReq();
        condition.setAccountId(Long.valueOf(requestDomain.getTopAccountId()));
        condition.setUserId(UserUtil.getUserId());
        condition.setStatus(Constant.DOUC_ENABLE_STATUS);
        condition.setRespMode(3);
        condition.setIds(memberUserIds);
        condition.setAccountScope(IDosmConstant.ACCOUNT_SCOPE);
        List<UserInfo> userByConditionV3 = userService.getUserByConditionV3(condition);
        List<String> oneBankIds = new ArrayList<>();
        userByConditionV3.forEach(item -> {
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
        private JsonNode fieldMapping;
        private Map<String, String> riskDictMapping;
    }
}
