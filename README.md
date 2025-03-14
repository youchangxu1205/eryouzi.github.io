package com.cloudwise.dosm.extend.btn;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.map.MapUtil;
import cn.hutool.core.text.CharSequenceUtil;
import cn.hutool.core.util.RandomUtil;
import cn.hutool.http.HttpRequest;
import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.IButtonActionExt;
import com.cloudwise.dosm.api.bean.adv.extend.BtnActionBo;
import com.cloudwise.dosm.api.bean.adv.extend.BtnCallResultBo;
import com.cloudwise.dosm.api.bean.core.pojo.bo.RequestDomain;
import com.cloudwise.dosm.api.bean.enums.BtnTypeEnum;
import com.cloudwise.dosm.api.bean.vo.dbs.DbsButtonConfig;
import com.cloudwise.dosm.core.config.NacosConfigLocalCatch;
import com.cloudwise.dosm.core.utils.JsonUtils;
import com.cloudwise.dosm.core.utils.SpringContextUtils;
import com.cloudwise.dosm.dict.dao.DataDictDetailMapper;
import com.cloudwise.dosm.dict.dao.DataDictMapper;
import com.cloudwise.dosm.dict.entity.DataDict;
import com.cloudwise.dosm.dict.entity.DataDictDetail;
import com.cloudwise.dosm.douc.entity.user.UserInfo;
import com.cloudwise.dosm.douc.sso.UserSSOClient;
import com.cloudwise.dosm.plugin.msgcommon.bo.VariableContext;
import com.cloudwise.dosm.plugin.msgcommon.service.WorkOrderVariableParserContextService;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.*;

/**
 * Rejection button interception function:
 * 1. Return to the current node based on the conditions (only applicable to single person approval scenarios)
 * 2. Ordinary node rejection requires filling in the rejection reason and person for the first rejection in the form
 *
 * @author ming.ma
 * @since 2025-01-15  17:04
 **/
@Slf4j
@Component
@ExtendConfig(id = "RejectBtnTrigger", name = "RejectBtnTrigger", desc = "RejectBtnTrigger")
public class RejectBtnTrigger implements IButtonActionExt {

    private static final String APP_OWNER = "ApplicationOwnerType";
    private static final String REJECTION_REASON = "RejectionReason";
    private static final String REJECTED_BY = "Rejectedby";
    private static final String APP_OWNER_VALUE = "PrimaryOrSecondaryApplicationOwner";
    private static final String CURRENT_NODE = "CURRENT_NODE";
    private static final String REJECTED = "rejected";

    private static final String PATH_SEND_MSG = "/dcs/custom/api/sendMessage";

    private static final String CSH = "custom.service.host";
    private static final String CSHPRE = "custom.service.host.prefix";



    private static final List<String> COMM_FIELD = Lists.newArrayList("title", "workOrderId", "processInstanceId", "orderId", "bizDesc", "dataStatus", "workOrderUri", "operator", "operatorId", "operatorAlias", "currentNode", "assignName", "assignId", "assignAlias");


    private static final String CR_REJECTED_SCHEDULE_DOWNTIME = "CR_REJECTED_SCHEDULE_DOWNTIME";
    private static final String CR_REJECTED_SCHEDULE_MAINTENANCE = "CR_REJECTED_SCHEDULE_MAINTENANCE";
    private static final List<String> CR_REJECTED_SCHEDULE_DOWNTIME_MAINTENANCE = Lists.newArrayList(CR_REJECTED_SCHEDULE_DOWNTIME, CR_REJECTED_SCHEDULE_MAINTENANCE);

    private static final String FK_SCHEDULE_DOWNTIME_START_END_TIME = "SCHEDULE_DOWNTIME_START_END_TIME";

    @Autowired
    private NacosConfigLocalCatch nacosConfigLocalCatch;

    @Autowired
    DbsButtonConfig dbsButtonConfig;

    @Autowired
    UserSSOClient userSSOClient;

    @Autowired
    DataDictDetailMapper dataDictDetailMapper;

    @Autowired
    WorkOrderVariableParserContextService parserContextService;

    @Override
    public List<BtnTypeEnum> getActionList() {
        return Lists.newArrayList(BtnTypeEnum.APPROVE_REJECT, BtnTypeEnum.APPROVE_PASS);
    }

    @Override
    public void before(BtnActionBo btnActionBo, BtnCallResultBo resultBo) {
        BtnTypeEnum btnType = btnActionBo.getBtnType();
        if (BtnTypeEnum.APPROVE_REJECT == btnType) {
            this.approveReject(btnActionBo, resultBo);
        }
    }

    @Override
    public void after(BtnActionBo btnActionBo, Object sysResultData, BtnCallResultBo resultBo) {
        sendRejectedEmail(btnActionBo, resultBo);
    }

    @Override
    public void rollbackBefore(BtnActionBo btnActionBo, Object sysResultData, Object rollbackParam) {

    }

    private UserInfo getUserInfoById(String userId) {
        List<UserInfo> userInfos = userSSOClient.getUserListByIds(Lists.newArrayList(Long.valueOf(userId)));
        if (CollUtil.isNotEmpty(userInfos)) {
            return userInfos.get(0);
        }
        return null;
    }

    public void approveReject(BtnActionBo btnActionBo, BtnCallResultBo resultBo) {

        String mdlDefKey = btnActionBo.getMdlDefKey();
        String nodeId = btnActionBo.getNodeId();
        Object formData = btnActionBo.getFormData();
        JsonNode formDataNode = JsonUtils.parseJsonNode(formData);
        ObjectNode formDataNodeOb = (ObjectNode) formDataNode;
        RequestDomain requestDomain = btnActionBo.getRequestDomain();
        Map<String, List<DbsButtonConfig.NodeConfig>> processConfig = dbsButtonConfig.getProcessConfig();
        if (processConfig == null) {
            log.error("RejectBtnTrigger get processConfig is empty");
            return;
        }
        log.error("RejectBtnTrigger get processConfig is:{}", processConfig);
        List<DbsButtonConfig.NodeConfig> nodeConfigs = processConfig.get(mdlDefKey);
        if (nodeConfigs == null) {
            log.error("RejectBtnTrigger get nodeConfigs is empty,mdlDefKey:{}", mdlDefKey);
            return;
        }
        List<String> nodes = new ArrayList<>();
        boolean flag = false;
        DbsButtonConfig.NodeConfig currNodeConfig = null;
        for (DbsButtonConfig.NodeConfig nodeConfig : nodeConfigs) {
            String configNodeId = nodeConfig.getNodeId();
            nodes.add(configNodeId);
            String rejectKey = nodeConfig.getRejectKey();
            boolean judge = nodeConfig.isJudge();
            if (CharSequenceUtil.equals(nodeId, configNodeId)) {
                currNodeConfig = nodeConfig;
                if (judge) {
                    log.error("RejectBtnTrigger judge:{},configNodeId:{}", judge, configNodeId);
                    JsonNode appOwner = formDataNodeOb.get(APP_OWNER);
                    JsonNode appOwnerValue = formDataNodeOb.get(APP_OWNER + "_value");
                    if (appOwnerValue != null && CharSequenceUtil.isNotBlank(appOwnerValue.asText())) {
                        String value = appOwnerValue.asText();
                        List<String> judgeValue = nodeConfig.getJudgeValue();
                        if (CollUtil.isNotEmpty(judgeValue)) {
                            if (judgeValue.contains(value)) {
                                //驳回到当前节点
                                btnActionBo.setRejectTo(CURRENT_NODE);
                                formDataNodeOb.put(rejectKey, REJECTED);
                            } else {
                                flag = true;
                            }
                        } else {
                            if (APP_OWNER_VALUE.equals(value)) {
                                //驳回到当前节点
                                btnActionBo.setRejectTo(CURRENT_NODE);
                                formDataNodeOb.put(rejectKey, REJECTED);
                            } else {
                                flag = true;
                            }
                        }

                    } else {
                        if (appOwner != null && CharSequenceUtil.isNotBlank(appOwner.asText())) {
                            List<String> judgeValue = nodeConfig.getJudgeValue();
                            String value = appOwner.asText();
                            if (CollUtil.isNotEmpty(judgeValue)) {
                                if (judgeValue.contains(value)) {
                                    //驳回到当前节点
                                    btnActionBo.setRejectTo(CURRENT_NODE);
                                    formDataNodeOb.put(rejectKey, REJECTED);
                                } else {
                                    flag = true;
                                }
                            } else {
                                if (APP_OWNER_VALUE.equals(value)) {
                                    //驳回到当前节点
                                    btnActionBo.setRejectTo(CURRENT_NODE);
                                    formDataNodeOb.put(rejectKey, REJECTED);
                                } else {
                                    flag = true;
                                }
                            }

                        }
                    }
                } else {
                    //驳回到当前节点
                    btnActionBo.setRejectTo(CURRENT_NODE);
                    formDataNodeOb.put(rejectKey, REJECTED);
                }
                break;
            }
        }
        String reason = btnActionBo.getReason();
        JsonNode rejectionReasonNode = formDataNodeOb.get(REJECTION_REASON);
        JsonNode rejectedByNode = formDataNodeOb.get(REJECTED_BY);
        log.error("RejectBtnTrigger start set reason and rejected by:rejectionReasonNode:{},rejectedByNode:{}", rejectionReasonNode, rejectedByNode);
        // 驳回节点首次驳回 设置驳回理由和驳回人
        if ((rejectionReasonNode == null || rejectionReasonNode.isNull() || CharSequenceUtil.isBlank(rejectionReasonNode.asText())) && (rejectedByNode == null || rejectedByNode.isNull() || CharSequenceUtil.isBlank(rejectedByNode.asText()))) {
            log.error("RejectBtnTrigger set reason and rejected by flag:{}", flag);
            //驳回设置字段
            if (!nodes.contains(nodeId) || (nodes.contains(nodeId) && flag)) {
                List<DataDictDetail> dataDictDetails = dataDictDetailMapper.getDetailsByIds(Lists.newArrayList(reason), null, null);
                if (CollUtil.isNotEmpty(dataDictDetails)) {
                    String label = dataDictDetails.get(0).getLabel();
                    log.error("RejectBtnTrigger dataDictDetailsy label:{}", label);
                    formDataNodeOb.put(REJECTION_REASON, label);
                }
                ArrayNode arrayNode = JsonUtils.createArrayNode();
                ObjectNode objectNode = JsonUtils.createObjectNode();
                String userId = requestDomain.getUserId();
                UserInfo userInfo = getUserInfoById(userId);
                if (userInfo != null) {
                    objectNode.put("userId", userId);
                    String userAlias = userInfo.getUserAlias();
                    String userName = userInfo.getName();
                    objectNode.put("userName", userName + "(" + userAlias + ")");
                    arrayNode.add(objectNode);
                    log.error("RejectBtnTrigger userInfo :{}", arrayNode);
                    formDataNodeOb.put(REJECTED_BY, arrayNode);
                }

                resultBo.setResultData(SendRejectedEmailResult.builder().send(Boolean.TRUE).currNodeConfig(currNodeConfig).formDataNode(formDataNodeOb).build());
            }
        }
        if(!CURRENT_NODE.equals(btnActionBo.getRejectTo())) {
            DataDictDetailMapper dataDictDetailMapper = SpringContextUtils.getBean(DataDictDetailMapper.class);
            DataDictMapper dataDictMapper = SpringContextUtils.getBean(DataDictMapper.class);
            DataDict dataDict = new DataDict();
            dataDict.setIsDel(0);
            dataDict.setDictCode("crStatus");
            DataDict crStatusDataDict = dataDictMapper.selectOneByParam(dataDict);
            List<DataDictDetail> dataDictDetails = dataDictDetailMapper.selectDetailByDictIdAndLevel(crStatusDataDict.getId(), 1, crStatusDataDict.getAccountId());
            Optional<DataDictDetail> first = dataDictDetails.stream().filter(item -> item.getData().equals("Rejected")).findFirst();
            if (first.isPresent()) {
                formDataNodeOb.put("crStatus", first.get().getId());
                formDataNodeOb.put("crStatus_value", "Rejected");
            }
            btnActionBo.setFormData(formDataNodeOb);
        }
    }

    private String getHost(String ips) {
        String[] split = ips.split(",");
        int idx = RandomUtil.randomInt(0, split.length);
        return split[idx];
    }


    private void sendRejectedEmail(BtnActionBo btnActionBo, BtnCallResultBo resultBo) {
        SendRejectedEmailResult sendRejectedEmail = null;
        if(resultBo == null || (sendRejectedEmail = (SendRejectedEmailResult) resultBo.getResultData())== null || !Boolean.TRUE.equals(sendRejectedEmail.getSend())) {
            log.info("not send rejected email, work workOrderId: {}, nodeId: {}", btnActionBo.getWorkOrderId(), btnActionBo.getNodeId());
            return;
        }



        if (sendRejectedEmail==null|| sendRejectedEmail.getCurrNodeConfig() == null) {
            log.info("not config notifyScences for workOrderId: {}, nodeId: {}", btnActionBo.getWorkOrderId(), btnActionBo.getNodeId());
            return;
        }
        List<String> notifyScences = sendRejectedEmail.getCurrNodeConfig().getNotifyScences();
        if(CollectionUtils.isEmpty(notifyScences)) {
            log.info("not config notifyScences for workOrderId: {}, nodeId: {}", btnActionBo.getWorkOrderId(), btnActionBo.getNodeId());
            return;
        }

        String notifyScence = CR_REJECTED_SCHEDULE_DOWNTIME;
        if(notifyScences.containsAll(CR_REJECTED_SCHEDULE_DOWNTIME_MAINTENANCE) && MapUtil.isNotEmpty(dbsButtonConfig.getFormFieldRefs())) {
            String formFieldCode = dbsButtonConfig.getFormFieldRefs().get(FK_SCHEDULE_DOWNTIME_START_END_TIME);
            JsonNode jsonNode = sendRejectedEmail.getFormDataNode().get(formFieldCode);
            notifyScence = JsonUtils.isNotEmpty(jsonNode)? CR_REJECTED_SCHEDULE_DOWNTIME: CR_REJECTED_SCHEDULE_MAINTENANCE;
        }

        Map properties = nacosConfigLocalCatch.getProperties();
        if (null == properties) {
            log.info("RejectBtnTrigger get properties is null");
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
            Map<String, Object> sendMsgContextMap = getSendMsgContext(notifyScence, btnActionBo, resultBo);
            log.info("RejectBtnTrigger call  url :{},param:{}", url, sendMsgContextMap);
            String body = HttpRequest.post(url).header("Content-Type", "application/json")
                    .body(JsonUtils.toJsonString(sendMsgContextMap))
                    .timeout(5000).execute().body();
            log.info("RejectBtnTrigger call  success:{}", body);
        } catch (Exception e) {
            log.error("RejectBtnTrigger call  fail:", e);
        }

    }


    private Map<String, Object> getSendMsgContext(String notifyScence, BtnActionBo btnActionBo, BtnCallResultBo resultBo) {
        Map<String, Object> sendMsgContextMap = new HashMap<>();

        Map<String, Object> notifyMap = new HashMap<>();
        sendMsgContextMap.put("notify", notifyMap);
        notifyMap.put("channelType", "EMAIL");
        notifyMap.put("notifyScene", notifyScence);

        RequestDomain requestDomain = btnActionBo.getRequestDomain();

        Map<String, String> publicFieldsMap = new HashMap<>();
        sendMsgContextMap.put("publicFields", publicFieldsMap);
        // return fieldKey: "title, workkkOrderId, processInstanceId, orderId, bizDesc, dataStatus, workOrderUri, operator, workOrderUri, operator, operatorId, operatorAlias, currentNode, assignName, assignId, assignAlias"
        Map<String, Object> workOrderBindingFieldMap = parserContextService.buildWorkOrderVariableContext(btnActionBo.getProcessInstanceId(), requestDomain.getUserId(), btnActionBo.getNodeId(), new VariableContext());
        COMM_FIELD.forEach(fieldKey -> publicFieldsMap.put(fieldKey, workOrderBindingFieldMap.getOrDefault(fieldKey, "").toString()));

        UserInfo createdUserInfo = getUserInfoById(btnActionBo.getCreatedBy());
        publicFieldsMap.put("createdById", btnActionBo.getCreatedBy());
        publicFieldsMap.put("createdByName", createdUserInfo == null? "": createdUserInfo.getName());
        publicFieldsMap.put("createdByEmail", createdUserInfo == null? "": createdUserInfo.getEmail());
        publicFieldsMap.put("workOrderId", btnActionBo.getWorkOrderId());
        publicFieldsMap.put("mdlDefKey", btnActionBo.getMdlDefKey());
        publicFieldsMap.put("bizKey", StringUtils.isBlank(btnActionBo.getBizKey())? btnActionBo.getWorkOrderId(): btnActionBo.getBizKey());

        sendMsgContextMap.put("workOrderId", btnActionBo.getWorkOrderId());
        sendMsgContextMap.put("nodeId", btnActionBo.getNodeId());
        sendMsgContextMap.put("createdBy", requestDomain.getUserId());
        sendMsgContextMap.put("topAccountId", requestDomain.getTopAccountId());
        sendMsgContextMap.put("accountId", requestDomain.getAccountId());
        sendMsgContextMap.put("userId", requestDomain.getUserId());

        return sendMsgContextMap;
    }

}
