package com.cloudwise.dosm.button;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.date.DateField;
import cn.hutool.core.date.DateTime;
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.util.IdUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.db.Db;
import cn.hutool.db.DbUtil;
import cn.hutool.db.Entity;
import cn.hutool.db.sql.Condition;
import cn.hutool.http.HttpRequest;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;
import cn.hutool.json.JSONArray;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import com.baomidou.mybatisplus.core.toolkit.ObjectUtils;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.toolkit.SimpleQuery;
import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.IButtonActionExt;
import com.cloudwise.dosm.api.bean.adv.extend.BtnActionBo;
import com.cloudwise.dosm.api.bean.adv.extend.BtnCallResultBo;
import com.cloudwise.dosm.api.bean.enums.BtnTypeEnum;
import com.cloudwise.dosm.api.bean.form.GroupBean;
import com.cloudwise.dosm.api.bean.form.enums.FieldValueTypeEnum;
import com.cloudwise.dosm.api.bean.trigger.entity.FieldTriggerRuleVo;
import com.cloudwise.dosm.api.bean.utils.JsonUtils;
import com.cloudwise.dosm.biz.instance.dao.MdlInstanceMapper;
import com.cloudwise.dosm.biz.instance.dto.BtnActionParam;
import com.cloudwise.dosm.biz.instance.entity.MdlInstance;
import com.cloudwise.dosm.biz.instance.service.btn.impl.BtnStartServiceImpl;
import com.cloudwise.dosm.biz.instance.service.mdlins.order.WorkOrderLinkRecodService;
import com.cloudwise.dosm.biz.instance.service.mdlins.order.WorkOrderLinkService;
import com.cloudwise.dosm.biz.instance.vo.WorkOrderLinkParamVo;
import com.cloudwise.dosm.bpm.api.action.enums.BtnEnum;
import com.cloudwise.dosm.bpm.api.trigger.FieldLinkTriggerService;
import com.cloudwise.dosm.bpm.base.entity.ProcessInfoPo;
import com.cloudwise.dosm.bpm.base.service.IProcessService;
import com.cloudwise.dosm.bpm.utils.ActivitiUtil;
import com.cloudwise.dosm.core.exception.BaseException;
import com.cloudwise.dosm.core.pojo.bo.CommonBo;
import com.cloudwise.dosm.core.utils.SpringContextUtils;
import com.cloudwise.dosm.core.utils.UserHolder;
import com.cloudwise.dosm.dict.dao.DataDictDetailMapper;
import com.cloudwise.dosm.dict.dao.DataDictMapper;
import com.cloudwise.dosm.dict.entity.DataDict;
import com.cloudwise.dosm.dict.entity.DataDictDetail;
import com.cloudwise.dosm.dict.service.DataDictService;
import com.cloudwise.dosm.dict.vo.DictDetailVo;
import com.cloudwise.dosm.dict.vo.DictQueryVo;
import com.cloudwise.dosm.facewall.core.response.ResultDataEntity;
import com.cloudwise.dosm.service.CreateCrRelationRacWorkUtils;
import com.cloudwise.douc.facadev3.GroupFacade;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.NullNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fasterxml.jackson.databind.node.TextNode;
import com.google.common.collect.Lists;
import com.google.common.collect.Maps;
import lombok.extern.slf4j.Slf4j;
import org.activiti.bpmn.model.FlowNode;
import org.apache.commons.collections.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.env.Environment;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.PreparedStatementCallback;
import org.springframework.jdbc.core.RowMapper;

import javax.annotation.Resource;
import javax.sql.DataSource;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.text.DecimalFormat;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.function.BiFunction;
import java.util.stream.Collectors;

/**
 * <p>
 *
 * </p>
 *
 * @author Norval.Xu
 * @since 2024/12/17
 */
@Slf4j
@ExtendConfig(id = "commitButton", name = "commitButton", desc = "commitButton")
public class CommitButtonExt implements IButtonActionExt {

    public static final String DOSM_CUSTOM_SERVICE_NAME_PREFIX = "custom.service.host.prefix";
    public static final String DOSM_CUSTOM_SERVICE_NAME = "custom.service.host";
    private static final String RCA_PRE_FIX_DEFAULT = "oqeetyig";

    @Value("${dosm.web.dbs.cr.relation.rca.prenum:" + RCA_PRE_FIX_DEFAULT + "}")
    private String RCA_PRE_FIX;

    @Resource
    private JdbcTemplate jdbcTemplate;

    private static final List<String> CLOSE_STATUS_LIST = Lists.newArrayList("Closed Backoutfail", "Closed Issues");
    private static final String CR_PRE_NUM_DEFAULT = "gsbmkthk";

    private static final String CR_CLOSE_NODE_ID_DEFAULT = "UserTask_05bhasg";
    private static final String CR_SIGNOFF_NODE_ID_DEFAULT = "UserTask_0da3d8t";

    @Value("${dosm.web.dbs.cr.prenum:" + CR_PRE_NUM_DEFAULT + "}")
    private String CR_PRE_NUM;

    @Value("${dosm.web.dbs.cr.close.node.id:" + CR_CLOSE_NODE_ID_DEFAULT + "}")
    private String CR_CLOSE_NODE_ID;

    @Value("${dosm.web.dbs.cr.signoff.node.id:" + CR_SIGNOFF_NODE_ID_DEFAULT + "}")
    private String CR_SIGNOFF_NODE_ID;
    @Resource
    private BtnStartServiceImpl btnStartService;

    @Value("${dosm.l1Half.change.manger.group.name:" + JudgeCmTeamApprove.l1HalfChangeMangerGroupName + "}")
    private String l1HalfChangeMangerGroup;

    private static String APP_CODE_FIELD = "applicationImpacte_value";

    @Resource
    private MdlInstanceMapper mdlInstanceMapper;

    @Resource
    private IProcessService iProcessService;

    @Resource
    private ActivitiUtil activitiUtil;

    @Resource
    private WorkOrderLinkService workOrderLinkService;


    @Resource
    private WorkOrderLinkRecodService workOrderLinkRecodService;
    @Resource
    private DataDictService dataDictService;
    @Resource
    private GroupFacade groupFacade;


    String majorChangesCode = "MajorChanges";

    String majorChangeDictCode = "MajorChange";

    String isMajorChangeCode = "IsMajorChange";

    String isMajorChangeDictCode = "YorN";

    String crClassification = "crClassification";

    String changeNature = "changeNature";

    String dataPatch = "dataPatchNumberOfRecords";

    String changeCategory = "Change_Category";

    String changeGroup = "changeGroup";

    String otherApplication = "otherApplicationImpacted";

    String location = "location";

    String changeTypeCode = "changeType";

    String parentChangeCode = "parentChange";

    String systemIdentifiedMajor = "System Identified Major";

    List<String> defDdlDefKeyList = Lists.newArrayList("jvtbicph");

    List<String> defNodeIdList = Lists.newArrayList("UserTask_0da3d8t");

    @Autowired
    private FieldLinkTriggerService fieldLinkTriggerService;

    @Override
    public List<BtnTypeEnum> getActionList() {
        List<BtnTypeEnum> button = new ArrayList<>();
        button.add(BtnTypeEnum.COMMIT);
        return button;
    }

    @Override
    public void before(BtnActionBo btnActionBo, BtnCallResultBo resultBo) {
        String bizKey = btnActionBo.getBizKey();
        long startTime = System.currentTimeMillis();
        if (bizKey.startsWith("CR")) {
            handleCrAction(btnActionBo, resultBo, bizKey);
        }
        // validateRCA
        validateRca(btnActionBo, resultBo);
        long endTime = System.currentTimeMillis();
        log.info("CommitButtonExt.before end,time:{}", endTime - startTime);
    }

    /**
     * when
     *
     * @param btnActionBo
     * @param resultBo
     * @param bizKey
     */
    private void handleCrAction(BtnActionBo btnActionBo, BtnCallResultBo resultBo, String bizKey) {
        log.error("start cr action");
        Object data = btnActionBo.getFormData();
        JsonNode formData = JsonUtils.parseJsonNode(data);
        log.error("cr action formData:{},{},{},{},{}", btnActionBo.getNodeId(), btnActionBo.getNodeName(), btnActionBo.getWorkOrderId(), bizKey, formData);
        if (formData == null) {
            throw new BaseException("form can not be empty!");
        }
        managedBlockSubmit(formData);
        boolean signOffNode = isSignOffNode(btnActionBo);

        validateCyberArkObjects(formData);

        if (formData.has("notNeedAiml")) {
            JsonNode notNeedAiml = formData.get("notNeedAiml");
            if (notNeedAiml.asBoolean()) {
                ((ObjectNode) formData).remove("notNeedAiml");
                btnActionBo.setFormData(formData);
                if (isSignOffNode(btnActionBo)) {
                    handleCrStatus2Open(btnActionBo, formData);
                    JudgeCmTeamApprove judgeCmTeamApprove = new JudgeCmTeamApprove(jdbcTemplate, dataDictService, groupFacade);
                    Map<String, String> dictInfoMap = judgeCmTeamApprove.queryDictLabel2Value(JudgeCmTeamApprove.YesOrNoDictCode);
                    GroupBean groupBean = judgeCmTeamApprove.getGroupBeanByName(l1HalfChangeMangerGroup);
                    handleL15CM(btnActionBo, formData, dictInfoMap, groupBean);
                }
                if (signOffNode) {
                    majorChangeAction((ObjectNode) formData, btnActionBo.getWorkOrderId(), btnActionBo);
                }
                log.error("AimlCommitButtonExt:{}", formData);
                return;
            }
        }
        if (formData.has("aiReqError")) {
            JsonNode aiReqError = formData.get("aiReqError");
            if (aiReqError.asBoolean()) {
                handleCrStatus2Open(btnActionBo, formData);
                ((ObjectNode) formData).remove("aiReqError");
                JudgeCmTeamApprove judgeCmTeamApprove = new JudgeCmTeamApprove(jdbcTemplate, dataDictService, groupFacade);
                Map<String, String> dictInfoMap = judgeCmTeamApprove.queryDictLabel2Value(JudgeCmTeamApprove.YesOrNoDictCode);
                GroupBean groupBean = judgeCmTeamApprove.getGroupBeanByName(l1HalfChangeMangerGroup);
                handleL15CM(btnActionBo, formData, dictInfoMap, groupBean);
                btnActionBo.setFormData(formData);
                return;
            }
        }

        // 判断是否为 signOff节点
        if (signOffNode) {
            ((ObjectNode) formData).put("CRSubmissionDatetime", System.currentTimeMillis());
            log.error("set CRSubmissionDatetime:{}", formData);

            JudgeCmTeamApprove judgeCmTeamApprove = new JudgeCmTeamApprove(jdbcTemplate, dataDictService, groupFacade);
            Map<String, String> dictInfoMap = judgeCmTeamApprove.queryDictLabel2Value(JudgeCmTeamApprove.YesOrNoDictCode);
            GroupBean groupBean = judgeCmTeamApprove.getGroupBeanByName(l1HalfChangeMangerGroup);

            handleSignOffNodeAction(btnActionBo, resultBo, bizKey, formData, dictInfoMap, groupBean);

            //L1.5 CM Team
//            this.handlerNeedL1HalfCmTeamApprove(judgeCmTeamApprove, btnActionBo, groupBean);
            majorChangeAction((ObjectNode) formData, btnActionBo.getWorkOrderId(), btnActionBo);
            handleCrStatus2Open(btnActionBo, formData);

            // default insert lv-signOff implamentation-check-signOff
            handlerDefaultSignOff((ObjectNode) formData, btnActionBo.getWorkOrderId());

        }

        if (isCloseNode(btnActionBo)) {
            handleCloseNodeAction(btnActionBo, resultBo, bizKey, formData);
        }

    }

    private void handlerDefaultSignOff(ObjectNode formData, String workOrderId) {

        SignOffHandler.handlerDefaultSignOff(formData, workOrderId);

    }

    private void handleCrStatus2Open(BtnActionBo btnActionBo, JsonNode formData) {
        ObjectNode formDataResult = (ObjectNode) formData;
        DataDictDetailMapper dataDictDetailMapper = SpringContextUtils.getBean(DataDictDetailMapper.class);
        DataDictMapper dataDictMapper = SpringContextUtils.getBean(DataDictMapper.class);
        DataDict dataDict = new DataDict();
        dataDict.setIsDel(0);
        dataDict.setDictCode("crStatus");
        DataDict crStatusDataDict = dataDictMapper.selectOneByParam(dataDict);
        List<DataDictDetail> dataDictDetails = dataDictDetailMapper.selectDetailByDictIdAndLevel(crStatusDataDict.getId(), 1, crStatusDataDict.getAccountId());
        Optional<DataDictDetail> first = dataDictDetails.stream().filter(item -> item.getData().equals("Open")).findFirst();
        if (first.isPresent()) {
            formDataResult.put("crStatus", first.get().getId());
            formDataResult.put("crStatus_value", "Open");
        }
        btnActionBo.setFormData(formData);
        JsonNode parentChange = formData.get("parentChange");
        updateParentCrShowChild(btnActionBo.getWorkOrderId(), parentChange == null ? "" : parentChange.asText());
    }

    private void updateParentCrShowChild(String workOrderId, String parentChangeCode) {

        MdlInstance crWorkOrderMdl = mdlInstanceMapper.selectByWorkOrderId(UserHolder.get().getTopAccountId(), workOrderId);
        Object data = crWorkOrderMdl.getFormData();
        JsonNode formData = JsonUtils.parseJsonNode(data);

        JsonNode fieldInfo = formData.get("parentChange");
        boolean oldHasParent = false;
        String oldParentCode = null;
        if (fieldInfo != null && !StringUtils.isBlank(fieldInfo.asText())) {
            oldHasParent = true;
            oldParentCode = fieldInfo.asText();
        }
        boolean newHasParent = false;
        if (StrUtil.isNotBlank(parentChangeCode)) {
            newHasParent = true;
        }

        if (newHasParent) {
            MdlInstance mdlInstance = mdlInstanceMapper.selectOne(Wrappers.lambdaQuery(MdlInstance.class)
                    .eq(MdlInstance::getBizKey, parentChangeCode), false
            );
            if (mdlInstance != null) {
                FieldTriggerRuleVo fieldTriggerRuleVo = new FieldTriggerRuleVo();
                fieldTriggerRuleVo.setKey("IfHasChildCR");
                fieldTriggerRuleVo.setType(FieldValueTypeEnum.INPUT.getVal());
                fieldTriggerRuleVo.setValue(Collections.singletonList("true"));
                List<FieldTriggerRuleVo> fieldTriggerRuleVos = new ArrayList<>();
                fieldTriggerRuleVos.add(fieldTriggerRuleVo);
                log.info("setIfHasChildCR, new:{}", true);
                fieldLinkTriggerService.triggeredUpdateFormData(mdlInstance.getProcessInstanceId(), fieldTriggerRuleVos);
            }
            if (oldHasParent && !oldParentCode.equalsIgnoreCase(parentChangeCode)) {
                mdlInstance = mdlInstanceMapper.selectOne(Wrappers.lambdaQuery(MdlInstance.class)
                        .eq(MdlInstance::getBizKey, oldParentCode), false
                );
                //查询parenChangeCode的总数
                List<Integer> count = jdbcTemplate.query("select count(1) from mdl_instance where json_value(form_data,'$.parentChange') = '" + oldParentCode + "'", new RowMapper<Integer>() {

                    @Override
                    public Integer mapRow(ResultSet rs, int rowNum) throws SQLException {
                        int anInt = rs.getInt(1);
                        return anInt;
                    }
                });
                boolean IfHasChildCR = false;
                if (!count.isEmpty() && count.get(0) - 1 > 0) {
                    IfHasChildCR = true;
                }
                FieldTriggerRuleVo fieldTriggerRuleVo = new FieldTriggerRuleVo();
                fieldTriggerRuleVo.setKey("IfHasChildCR");
                fieldTriggerRuleVo.setType(FieldValueTypeEnum.INPUT.getVal());
                fieldTriggerRuleVo.setValue(Collections.singletonList("" + IfHasChildCR));
                List<FieldTriggerRuleVo> fieldTriggerRuleVos = new ArrayList<>();
                fieldTriggerRuleVos.add(fieldTriggerRuleVo);
                log.info("setIfHasChildCR, old:{}", IfHasChildCR);
                fieldLinkTriggerService.triggeredUpdateFormData(mdlInstance.getProcessInstanceId(), fieldTriggerRuleVos);

            }

        }

    }

    //handle L1.5 CM Team approve
    private void handlerNeedL1HalfCmTeamApprove(JudgeCmTeamApprove judgeCmTeamApprove, BtnActionBo btnActionBo, GroupBean groupBean) {
        judgeCmTeamApprove.handlerNeedL1Approve(btnActionBo, groupBean);
    }

    private static void managedBlockSubmit(JsonNode formData) {
        Long minStartValue = null;
        Long maxEndValue = null;
        if (formData.has("crClassification_value")) {
            JsonNode classification = formData.get("crClassification_value");
            if (classification.asText().equals("Managed")) {
                try {
                    JsonNode table = formData.get("F_Reversion_Plan");

                    for (JsonNode node : table) {
                        JsonNode rowData = node.get("rowData");
                        JsonNode startTime = rowData.get("F_Start_Time");
                        if (minStartValue == null || startTime.asLong() < minStartValue) {
                            minStartValue = startTime.asLong();
                        }
                        JsonNode endTime = rowData.get("F_End_Time");
                        if (maxEndValue == null || endTime.asLong() > maxEndValue) {
                            maxEndValue = endTime.asLong();
                        }
                    }
                } catch (Exception e) {
                    log.error("managedBlockSubmit", e);
                }
                if (minStartValue != null && maxEndValue != null) {
                    long tableDuration = (maxEndValue - minStartValue) / 60 / 1000;
                    if (tableDuration > 120) {
                        throw new BaseException("Please reduce the reversion duration or change the CR to Heightened CR");
                    }
                }
            }
        }
    }

    /**
     * When a requester submits CR for approval, the status of the CyberArk Objects needs to be verified, which means CyberArk Objects in the ‘Pre-Production’ status cannot be included in the CR, except for the following circumstances:
     * LOBs = CES & Change Group when it is any of the following: Project, Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business, BAU Cutover. CR can contain Objects in the ‘Pre-Production’ state.
     * LOBs = CES & Change Group when it is NOT any of the following: Project, Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business, BAU Cutover. CR cannot contain Objects in the ‘Pre-Production’ state.
     * LOBs != CES& Change Group is any of the following: Project and Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business. CR can contain Objects in the ‘Pre-Production’ state
     * LOBs != CES& Change Group is NOT any of the following: Project and Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business. CR cannot contain Objects in the ‘Pre-Production’ state.
     *
     * @param formData
     */
    private void validateCyberArkObjects(JsonNode formData) {
        JsonNode cyberArkObject = formData.get("CyberArk_Object");
        if (cyberArkObject != null) {
            String cyberArkObjectValue = cyberArkObject.asText();
            JsonNode lobValue = formData.get("lob_value");
            JsonNode changeGroupValue = formData.get("changeGroup_value");

            if (Objects.nonNull(cyberArkObjectValue) && !"NA".equals(cyberArkObjectValue.toUpperCase()) && canNotContainPreProd(lobValue, changeGroupValue)) {
                String[] cyberArkObjects = cyberArkObjectValue.split(",");

                List<String> query = jdbcTemplate.query("select password_object from cyberark_info as ci left join ic_server_detatils_t as isdt on ci.hostname = isdt.hostname where ci.password_object in ('" + StringUtils.join(cyberArkObjects, "','") + "') and isdt.status = 'Pre-Production'", new RowMapper<String>() {
                    @Override
                    public String mapRow(ResultSet rs, int rowNum) throws SQLException {
                        String count = rs.getString(1);
                        return count;
                    }
                });
                if (query.size() > 0) {
                    String lobValueStr = lobValue.asText();
                    if ("CES".equals(lobValueStr)) {
                        throw new BaseException("LOBs = CES & Change Group when it is NOT any of the following: Project, Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business, BAU Cutover. CR cannot contain Objects in the ‘Pre-Production’ state.");
                    } else {
//                        throw new BaseException("CyberArk Objects in the ‘Pre-Production’ status cannot be included in the CR:" + String.join(",", query));
                        throw new BaseException("LOBs != CES& Change Group is NOT any of the following: Project and Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business. CR cannot contain Objects in the ‘Pre-Production’ state");
                    }
                }
            }
        }
    }

    /**
     * LOBs = CES & Change Group when it is any of the following: Project, Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business, BAU Cutover. CR can contain Objects in the ‘Pre-Production’ state.
     * LOBs = CES & Change Group when it is NOT any of the following: Project, Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business, BAU Cutover. CR cannot contain Objects in the ‘Pre-Production’ state.
     * LOBs != CES& Change Group is any of the following: Project and Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business. CR can contain Objects in the ‘Pre-Production’ state
     * LOBs != CES& Change Group is NOT any of the following: Project and Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business. CR cannot contain Objects in the ‘Pre-Production’ state.
     *
     * @param lobValue
     * @param changeGroupValue
     * @return
     */
    private boolean canNotContainPreProd(JsonNode lobValue, JsonNode changeGroupValue) {
        if (lobValue == null || changeGroupValue == null) {
            return false;
        }
        String lobValueStr = lobValue.asText();
        String changeGroupValueStr = changeGroupValue.asText();
        if (lobValueStr.equals("CES") && !changeGroupValueStr.equals("Project") &&
                !changeGroupValueStr.equals("Project Cutover Technical") &&
                !changeGroupValueStr.equals("Project Cutover Business") &&
                !changeGroupValueStr.equals("Project Cutover Technical & Business") &&
                !changeGroupValueStr.equals("BAU Cutover")) {
            return true;
        }
        if (!lobValueStr.equals("CES") &&
                !changeGroupValueStr.equals("Project") &&
                !changeGroupValueStr.equals("Project Cutover Technical") &&
                !changeGroupValueStr.equals("Project Cutover Business") &&
                !changeGroupValueStr.equals("Project Cutover Technical & Business")) {
            return true;
        }
        return false;
    }

    private static List<String> failureStatus = new ArrayList<>();

    static {
        failureStatus.add("ProdQA Deployment Failed");
        failureStatus.add("PAT Deployment Failed");
        failureStatus.add("PILOT Deployment Failed");
        failureStatus.add("PRD Deployment Failed");
        failureStatus.add("DR Deployment Failed");
        failureStatus.add("Rollback Failed");
    }

    /**
     * 5.4.Validation on Closed Successful with Drone
     * During CR Closure, ITSM will call Drone to get the result of deployment. CR cannot be allowed to set as 'Closed Successful' if DRONE ticket status is any status indicated below:
     * ProdQA Deployment Failed
     * PAT Deployment Failed
     * PILOT Deployment Failed
     * PRD Deployment Failed
     * DR Deployment Failed
     * Rollback Failed
     *
     * @param btnActionBo
     * @param resultBo
     * @param bizKey
     * @param formData
     */
    private void handleCloseNodeAction(BtnActionBo btnActionBo, BtnCallResultBo resultBo, String bizKey, JsonNode formData) {
        JsonNode closedStatusValue = formData.get("CloseStatus_value");
        if (closedStatusValue == null) {
            throw new BaseException("Close Status can not be null");
        }
        String closedStatusValueText = closedStatusValue.asText();
        if (closedStatusValueText.equals("Closed Successful")) {
            JSONObject paramBody = new JSONObject();
            JsonNode droneTicket = formData.get("DroneTicket");
            if (droneTicket == null) {
                return;
            }
            paramBody.set("releaseticketids", droneTicket.asText());
            String droneUrl = getConfig("dosm.custom.droneTickets");
            JSONObject ticketResult = getApiTenByParam(droneUrl, paramBody, bizKey);
            JSONArray releaseticketsJson = ticketResult.getJSONArray("releasetickets");
            boolean isCanContinue = true;
            StringBuilder droneTicketId = new StringBuilder();
            for (Object object : releaseticketsJson) {
                JSONObject releaseticket = (JSONObject) object;
                String status = releaseticket.getStr("status");
                if (failureStatus.contains(status)) {
                    isCanContinue = false;
                    droneTicketId.append(releaseticket.getStr("id")).append(":").append(status);
                }
            }
            if (!isCanContinue) {
                throw new BaseException("CR cannot be allowed to set as 'Closed Successful' when " + droneTicketId.toString());
            }
        }
        //Handle RCA work order
        if (CR_PRE_NUM.equals(btnActionBo.getMdlDefKey()) && CR_CLOSE_NODE_ID.equals(btnActionBo.getNodeId())) {
            if (formData.has("CloseStatus") && CLOSE_STATUS_LIST.contains(formData.get("CloseStatus_value").asText())) {
                MdlInstance crWorkOrderMdl = mdlInstanceMapper.selectByWorkOrderId(UserHolder.get().getTopAccountId(), btnActionBo.getWorkOrderId());

                // 获取RCA的最新版本 以及表单ID
                ProcessInfoPo rcaProcessInfo = iProcessService.getMaxVersion(RCA_PRE_FIX);
                String rcaFormId = rcaProcessInfo.getMdlFormId();
                String rcaProcessDefId = rcaProcessInfo.getProcessDefId();

                // cr close node handler
                BtnActionParam btnStartParam = buildRcaBtnActionParam(rcaFormId, rcaProcessDefId, crWorkOrderMdl);

                MdlInstance racMdl = new MdlInstance();
                btnStartParam.setData(racMdl);

                ObjectNode formDataJson = buildRcaFormDataJson(formData, btnActionBo.getBizKey(), crWorkOrderMdl);

                buildRcaMdl(racMdl, formDataJson, rcaProcessInfo, crWorkOrderMdl, rcaProcessDefId);

                String racWorkOrderId = createRacWorkOrder(btnStartParam);

                // 关联工单
                linkWorkOrders(btnActionBo.getWorkOrderId(), racWorkOrderId);

                // 修改RAC的 工单编号  RAC-CR-00000001  更新工单创建人
                updateRacWorkOrder(racWorkOrderId, btnActionBo.getBizKey(), crWorkOrderMdl);

                // create impacted application status relate status
                createRacWorkOrder(formDataJson.get("impactedapplication_value").asText(), btnActionBo.getWorkOrderId(), racWorkOrderId);
            }
        }
    }

    private void createRacWorkOrder(String applicationCode, String crWorkOrderId, String rcaWorkOrderId) {

        Date lastDay = CreateCrRelationRacWorkUtils.getLastDay(jdbcTemplate);

        jdbcTemplate.update("insert into dbs_rca_application_pending(created_time,last_pending_time,application_name,cr_work_order_id,rca_work_order_id,is_pending) values(?,?,?,?,?,?)", new Date(), lastDay, applicationCode, crWorkOrderId, rcaWorkOrderId, false);

    }

    private void updateRacWorkOrder(String racWorkOrderId, String crBizKey, MdlInstance crWorkOrderMdl) {
        String racBizKey = "RCA-" + crBizKey;
        if (StringUtils.isNotEmpty(crBizKey)) {
            mdlInstanceMapper.update(Wrappers.lambdaUpdate(MdlInstance.class)
                    .eq(MdlInstance::getId, racWorkOrderId)
                    .set(StringUtils.isNotEmpty(crBizKey), MdlInstance::getBizKey, racBizKey)
            );
        }
    }

    private void linkWorkOrders(String crWorkOrderId, String racWorkOrderId) {
        CommonBo commonBo = CommonBo.buildWith(UserHolder.get());
        WorkOrderLinkParamVo workOrderLinkParamVo = new WorkOrderLinkParamVo();
        workOrderLinkParamVo.setOrgOrderId(crWorkOrderId);
        workOrderLinkParamVo.setTarOrderList(Lists.newArrayList(racWorkOrderId));
        workOrderLinkService.batchRelevance(commonBo, workOrderLinkParamVo);
        workOrderLinkRecodService.batchRelevance(commonBo, workOrderLinkParamVo);
    }

    private String createRacWorkOrder(BtnActionParam btnStartParam) {
        try {
            return btnStartService.exec(btnStartParam);
        } catch (Exception e) {
            log.error("create rac workOrder error", e);
            throw new RuntimeException("create rac workOrder error", e);
        }
    }

    private void buildRcaMdl(MdlInstance racMdl, ObjectNode formDataJson, ProcessInfoPo rcaProcessInfo, MdlInstance crWorkOrderMdl, String rcaProcessDefId) {
        racMdl.setFormData(formDataJson);
        racMdl.setOrderType(rcaProcessInfo.getProcessType());
        racMdl.setSourceId("WEB_FORM");
        racMdl.setTopAccountId(crWorkOrderMdl.getTopAccountId());

        racMdl.setAccountId(crWorkOrderMdl.getAccountId());

        racMdl.setIsTest(0L);

        racMdl.setCreatedBy(crWorkOrderMdl.getCreatedBy());

        racMdl.setDataStatus(0);

        racMdl.setMdlDefCode(rcaProcessDefId);
        racMdl.setMdlDefKey(RCA_PRE_FIX);
    }

    private ObjectNode buildRcaFormDataJson(JsonNode crFormDataJson, String crBizKey, MdlInstance crWorkOrderMdl) {
        ObjectNode formDataJson = JsonUtils.createObjectNode();
        formDataJson.put("CRTicketNumber", crBizKey);

        // 暂存数据 cr 下拉选项范围数据
        List<String> applicationImpacteList = JsonUtils.convertList(crFormDataJson.get("applicationImpacte"), String.class);
        List<String> applicationImpacteValueList = JsonUtils.convertList(crFormDataJson.get("applicationImpacte_value"), String.class);
        Map<String, String> applicationImpacteMap = Maps.newHashMap();
        for (int i = 0; i < applicationImpacteValueList.size(); i++) {
            applicationImpacteMap.put(applicationImpacteList.get(i), applicationImpacteValueList.get(i));
        }
        formDataJson.put("saveImpactedApplication", JsonUtils.parseJsonNode(applicationImpacteMap).toString());

        if (Objects.nonNull(crFormDataJson.get("changeSummary"))) {
            formDataJson.put("changeSummary", crFormDataJson.get("changeSummary").asText());
        }

        // 配置rca配置的下拉选项 取第一个选项数据
        formDataJson.put("impactedapplication", applicationImpacteList.get(0));
        formDataJson.put("impactedapplication_value", applicationImpacteValueList.get(0));

        String relatedIncident = Optional.ofNullable(crFormDataJson.get("relatedIncident")).map(JsonNode::asText).orElse(null);
        log.error("current RCA relatedIncident value is:{}", relatedIncident);
        formDataJson.put("relatedIncident", relatedIncident);

        formDataJson.put("L1Risk_Manager", crFormDataJson.get("L1Risk_Manager"));
        return formDataJson;
    }

    private BtnActionParam buildRcaBtnActionParam(String rcaFormId, String rcaProcessDefId, MdlInstance crWorkOrderMdl) {
        BtnActionParam btnStartParam = new BtnActionParam();
        btnStartParam.setSubProcess(true);
        btnStartParam.setBtnType(BtnEnum.COMMIT);
        btnStartParam.setFormId(rcaFormId);
        FlowNode firstActivityByDefId = activitiUtil.findFirstActivityByDefId(rcaProcessDefId);
        btnStartParam.setNodeId(firstActivityByDefId.getId());
        btnStartParam.setUserId(UserHolder.get().getUserId());
        btnStartParam.setTopAccountId(UserHolder.get().getTopAccountId());
        btnStartParam.setAccountId(UserHolder.get().getAccountId());
        return btnStartParam;
    }

    public JSONObject getApiTenByParam(String url, JSONObject body, String bizKey) {
        String appCode = getConfig("dosm.custom.appCode");
        String appKey = getConfig("dosm.custom.appKey");
        String resultHttpDetail = null;
        String jsonStr = JSONUtil.toJsonStr(body);
        try {
            resultHttpDetail = HttpUtil
                    .createPost(url)
                    .header("appCode", appCode)
                    .header("appKey", appKey)
                    .timeout(20000)
                    .body(jsonStr)
                    .execute().body();

            DosmCommonResponse dosmCommonResponse = JSONUtil.toBean(resultHttpDetail, DosmCommonResponse.class);
            if (ObjectUtils.isNotEmpty(dosmCommonResponse) && dosmCommonResponse.getCode().equals("200")) {
                String data = dosmCommonResponse.getData();
                saveApiLog(url, jsonStr, resultHttpDetail, bizKey, "droneTicket", true);
                return JSONUtil.parseObj(data);
            } else {
                saveApiLog(url, jsonStr, resultHttpDetail, bizKey, "droneTicket", false);
            }
        } catch (Exception e) {
            log.error("请求api失败{}", body);
            log.error("请求api失败{}", e.getMessage());
            saveApiLog(url, jsonStr, resultHttpDetail, bizKey, "droneTicket", false);
        }
        return null;
    }

    private boolean isCloseNode(BtnActionBo btnActionBo) {
        if (btnActionBo.getNodeId().equals(CR_CLOSE_NODE_ID)) {
            return true;
        }
        return false;
    }


    public void majorChangeAction(ObjectNode formData, String workOrderId, BtnActionBo btnActionBo) {
        BiFunction<JsonNode, String, String> extractSelectLabelValueFunc = (jsonNode, fieldCode) -> Optional.ofNullable(jsonNode)
                .filter(com.cloudwise.dosm.core.utils.JsonUtils::isNotEmpty)
                .map(node -> {
                    log.error("MajorChangeButtonEvent workOrderId:{},The code of the current dropdown field of the MajorChangeButton is: {}, and the value information of the field is: {}", workOrderId, fieldCode, node);
                    JsonNode currentSelectLabel = node.get(fieldCode + "_value");
                    log.error("MajorChangeButtonEvent workOrderId:{},For the MajorChangeButton, the code of the current dropdown field is: {}, and the label information of the field value is: {}", workOrderId, fieldCode, currentSelectLabel);
                    return currentSelectLabel;
                })
                .filter(com.cloudwise.dosm.core.utils.JsonUtils::isNotEmpty)
                .map(JsonNode::asText)
                .orElse(null);

        //Single - selection type component
        String crClassifyValue = extractSelectLabelValueFunc.apply(formData, crClassification);
        String changeNatureValue = extractSelectLabelValueFunc.apply(formData, changeNature);
        String locationValue = extractSelectLabelValueFunc.apply(formData, location);
        String changeGroupValue = extractSelectLabelValueFunc.apply(formData, changeGroup);
        String changeTypeValue = extractSelectLabelValueFunc.apply(formData, changeTypeCode);

        //Multiple - selection type component
        String otherAppLabelCode = otherApplication + "_value";
        List<String> otherAppValue = Optional.ofNullable(formData.get(otherAppLabelCode))
                .filter(com.cloudwise.dosm.core.utils.JsonUtils::isNotEmpty)
                .map(otherAppLabelNode -> com.cloudwise.dosm.core.utils.JsonUtils.parseArray(otherAppLabelNode, String.class))
                .orElseGet(ArrayList::new);

        boolean existRaf = otherAppValue.stream().anyMatch(otherApp -> StringUtils.contains(otherApp, "RAF"));

        boolean rafFlag;
        if (existRaf) {
            rafFlag = otherAppValue.size() > 6;
        } else {
            rafFlag = otherAppValue.size() > 5;
        }

        log.error("MajorChangeButtonEvent workOrderId:{},The current Raf judgment information of the MajorChangeButton is: {}", workOrderId, rafFlag);

        //Text type component
        String changeCategoryValue = Optional.ofNullable(formData.get(changeCategory)).filter(com.cloudwise.dosm.core.utils.JsonUtils::isNotEmpty).map(JsonNode::asText).orElse(null);
        String parentChangeValue = Optional.ofNullable(formData.get(parentChangeCode)).filter(com.cloudwise.dosm.core.utils.JsonUtils::isNotEmpty).map(JsonNode::asText).orElse(null);

        //Number type component
        long dataPatchNum = Optional.ofNullable(formData.get(dataPatch)).filter(com.cloudwise.dosm.core.utils.JsonUtils::isNotEmpty).map(JsonNode::asLong).orElse(0L);

        log.error("MajorChangeButtonEvent workOrderId:{},The current form field information of the MajorChangeButton is as follows: " +
                        "crClassification: {}, " +
                        "changeNature: {}, " +
                        "location: {}, " +
                        "changeGroup: {}, " +
                        "changeType: {}, " +
                        "otherApplicationImpacted: {}, " +
                        "Change_Category: {}, " +
                        "parentChange: {}, " +
                        "dataPatchNumberOfRecords: {}"
                , workOrderId
                , crClassifyValue
                , changeNatureValue
                , locationValue
                , changeGroupValue
                , changeTypeValue
                , otherAppValue
                , changeCategoryValue
                , parentChangeValue
                , dataPatchNum);

        String majorChangeLabelCode = majorChangesCode + "_value";
        List<String> majorChanges = Optional.ofNullable(formData.get(majorChangeLabelCode))
                .filter(com.cloudwise.dosm.core.utils.JsonUtils::isNotEmpty)
                .map(otherAppLabelNode -> com.cloudwise.dosm.core.utils.JsonUtils.parseArray(otherAppLabelNode, String.class))
                .orElseGet(ArrayList::new);

        List<String> newProduct = Arrays.asList("With upstream/downstream impact and complex system dependencies greater than 5 other apps impacted",
                "Service/System/Data migration", "Heighten/Heighten App", "Program or code change to more than 1 system (more than 3 application codes)"
                , "Major Release/Upgrade", "DB/Middleware/Clustered software patching or upgrades", "New Product");

        boolean majorChangesBoolean = newProduct.stream().anyMatch(majorChanges::contains);

        boolean excludeLocationFlag = !StringUtils.equalsAny(locationValue, "PAT", "Pre-prod", "QA");
        boolean changeGroupFlag = StringUtils.equalsAny(changeGroupValue, "Project Cutover Technical", "Project CCutover Business", "Project Cutover Technical & Business");

        boolean regularA = "Heightened".equals(crClassifyValue) && "Compliance with Industry Standards".equals(changeNatureValue) && excludeLocationFlag;
        boolean regularB = "Heightened".equals(crClassifyValue) && dataPatchNum > 1000L;
        boolean regularC = "Heightened".equals(crClassifyValue) && "High".equals(changeCategoryValue);
        boolean regularD = "Heightened".equals(crClassifyValue) && changeGroupFlag;
        boolean regularE = "Heightened".equals(crClassifyValue) && "Technology Re-Platform".equals(changeNatureValue) && excludeLocationFlag;
        boolean regularF = "Heightened".equals(crClassifyValue) && StringUtils.equalsAny(changeTypeValue, "Heighten", "Heighten App") && excludeLocationFlag;
        boolean regularG = "Heightened".equals(crClassifyValue) && majorChangesBoolean && excludeLocationFlag;
        boolean regularH = "Heightened".equals(crClassifyValue) && rafFlag && majorChangesBoolean;

        log.error("MajorChangeButtonEvent workOrderId:{},The results of rule matching for the MajorChangeButton are: {}, {}, {}, {}, {}, {}, {}", workOrderId, regularA, regularB, regularC, regularD, regularE, regularF, regularG);

        if (regularA
                || regularB
                || regularC
                || regularD
                || regularE
                || regularF
                || regularG
                || regularH) {
            log.error("MajorChangeButtonEvent workOrderId:{},The MajorChangeButton has matched the system judgment. Assign values to the system judgment options of the dropdown field for Major Change.", workOrderId);


            //not exist
            if (!majorChanges.contains(systemIdentifiedMajor)) {

                DictQueryVo dictQueryVo = dataDictService.getDictByCode(majorChangeDictCode, UserHolder.getCommonBo());

                Map<String, String> dictLabel2IdMap = Optional.ofNullable(dictQueryVo)
                        .map(DictQueryVo::getFieldContent)
                        .filter(CollectionUtils::isNotEmpty)
                        .map(fieldContent -> SimpleQuery.list2Map(fieldContent, DictDetailVo::getLabel, DictDetailVo::getId))
                        .orElseGet(HashMap::new);

                log.error("MajorChangeButtonEvent workOrderId:{},current major change dict info: {}", workOrderId, dictLabel2IdMap);

                String dictId = dictLabel2IdMap.get(systemIdentifiedMajor);

                log.error("MajorChangeButtonEvent workOrderId:{},The formData information before updating form data: {}", workOrderId, formData);

                JsonNode majorChangeNode = org.apache.commons.lang3.ObjectUtils.defaultIfNull(formData.get(majorChangesCode), com.cloudwise.dosm.core.utils.JsonUtils.createArrayNode());

                ((ArrayNode) majorChangeNode).add(dictId);

                ((ObjectNode) formData).set(majorChangesCode, majorChangeNode);

                //_value
                majorChanges.add(systemIdentifiedMajor);
                ((ObjectNode) formData).putPOJO(majorChangeLabelCode, majorChanges);

                log.error("MajorChangeButtonEvent workOrderId:{}, The formData information after updating the form data: {}", workOrderId, formData);
                btnActionBo.setFormData(formData);
            }
        }

        //fill IsMajorChange selector component
        List<String> majorChangesValueList = Optional.ofNullable(formData.get(majorChangesCode))
                .filter(com.cloudwise.dosm.core.utils.JsonUtils::isNotEmpty)
                .map(otherAppLabelNode -> com.cloudwise.dosm.core.utils.JsonUtils.parseArray(otherAppLabelNode, String.class))
                .orElseGet(ArrayList::new);

        if (CollectionUtils.isNotEmpty(majorChangesValueList)) {
            DictQueryVo dictQueryVo = dataDictService.getDictByCode(isMajorChangeDictCode, UserHolder.getCommonBo());

            Map<String, String> dictLabel2IdMap = Optional.ofNullable(dictQueryVo)
                    .map(DictQueryVo::getFieldContent)
                    .filter(CollectionUtils::isNotEmpty)
                    .map(fieldContent -> SimpleQuery.list2Map(fieldContent, DictDetailVo::getLabel, DictDetailVo::getId))
                    .orElseGet(HashMap::new);

            log.error("MajorChangeButtonEvent workOrderId:{}, current is major change dict info: {}", workOrderId, dictLabel2IdMap);

            String isYes = "Yes";

            ((ObjectNode) formData).put(isMajorChangeCode, dictLabel2IdMap.get(isYes));
            ((ObjectNode) formData).put(isMajorChangeCode + "_value", isYes);
        }
    }

    private void handleSignOffNodeAction(BtnActionBo btnActionBo, BtnCallResultBo resultBo, String bizKey, JsonNode formData, Map<String, String> dictInfoMap, GroupBean groupBean) {
        // signOff节点 提交需要判断 审批结果
        this.calculateSignOffResult(btnActionBo.getWorkOrderId());
        Boolean isNeedAiml = checkNeedAiml(btnActionBo, formData);
        if (isNeedAiml) {
            try {
                JsonNode aimlResult = queryAimlResult(btnActionBo, formData);
                if (aimlResult == null) {
                    aimlErrorHandler(resultBo);
                    return;
                }
                Boolean isDiff = compareChangeCategory(aimlResult, btnActionBo, formData, resultBo);
                if (isDiff) {
                    ObjectNode objectNode = JsonUtils.createObjectNode();
                    objectNode.put("needDialog", true);
                    objectNode.put("aiReqError", false);
                    objectNode.put("aiReqErrorMsg", "Not able to get Risk Score from AI. Please retry CR submission.");
                    objectNode.put("workOrderNo", bizKey);
                    ObjectNode formDataResult = JsonUtils.createObjectNode();
                    JsonNode aimlData = aimlResult.get("data");
                    JsonNode aimlRowData = aimlData.get("rowData");
                    JsonNode explainability = aimlRowData.get("explainability");
                    String explainabilityText = explainability.asText();
                    String[] explainabilitys = explainabilityText.split(",");
                    formDataResult.set("riskScore", aimlRowData.get("score"));
                    formDataResult.set("riskThreshold", aimlRowData.get("risk_threshold"));
                    JsonNode explainabilityScore = aimlRowData.get("explainability_scores");
                    List<Double> explainabilityScoreList = new ArrayList<>();
                    for (JsonNode featureValue : explainabilityScore) {
                        DecimalFormat decimalFormat = new DecimalFormat("#.###");
                        double aDouble = Double.parseDouble(decimalFormat.format(featureValue.asDouble()));
                        explainabilityScoreList.add(aDouble);
                    }
                    formDataResult.set("explainabilityScores", JsonUtils.parseJsonNode(explainabilityScoreList.stream().limit(5).collect(Collectors.toList())));
                    formDataResult.put("explainability", Arrays.stream(explainabilitys).limit(5).collect(Collectors.joining(",")));
                    formDataResult.set("explainabilityBase", aimlRowData.get("explainability_base"));
                    formDataResult.set("recommendedRisk", aimlRowData.get("risk"));
                    JsonNode featureValues = aimlRowData.get("feature_values");
                    List<Double> featureValueList = new ArrayList<>();
                    for (JsonNode featureValue : featureValues) {
                        featureValueList.add(featureValue.asDouble());
                    }
                    formDataResult.set("featureValues", JsonUtils.parseJsonNode(featureValueList.stream().limit(5).collect(Collectors.toList())));
                    objectNode.set("formData", formDataResult);
                    objectNode.setAll((ObjectNode) aimlResult.get("data"));
                    resultBo.setResultData(objectNode);
                    resultBo.setSkipSysExecutor(true);
                } else {
                    JsonNode aimlData = aimlResult.get("data");
                    JsonNode aimlRowData = aimlData.get("rowData");
                    ObjectNode formDataResult = (ObjectNode) formData;
                    formDataResult.set("riskScore", aimlRowData.get("score"));
                    formDataResult.set("riskThreshold", aimlRowData.get("risk_threshold"));
                    JsonNode explainabilityScore = aimlRowData.get("explainability_scores");
                    formDataResult.set("explainabilityScores", explainabilityScore);
                    formDataResult.set("explainability", aimlRowData.get("explainability"));
                    formDataResult.set("explainabilityBase", aimlRowData.get("explainability_base"));
                    formDataResult.set("recommendedRisk", aimlRowData.get("risk"));
                    formDataResult.set("featureValues", aimlRowData.get("feature_values"));
                    resultBo.setSkipSysExecutor(false);
                }
            } catch (Exception e) {
                log.error("Not able to get Risk Score from AI. Please retry CR submission.", e);
//                throw new BaseException("Not able to get Risk Score from AI. Please retry CR submission.");
                aimlErrorHandler(resultBo);
                return;
            }
        } else {
            resultBo.setSkipSysExecutor(false);
        }

        if (!resultBo.isSkipSysExecutor()) {
            handleL15CM(btnActionBo, formData, dictInfoMap, groupBean);
        }


        btnActionBo.setFormData(formData);


    }

    private void handleL15CM(BtnActionBo btnActionBo, JsonNode formData, Map<String, String> dictInfoMap, GroupBean groupBean) {
        DateTime currentTime = DateUtil.date();
        if (currentTime.getField(DateField.HOUR_OF_DAY) >= 17) {
            DateTime nextCutoffTime = getNextCutoffTime();
            log.error("CommitButton:nextCutoffTime:{}", DateUtil.formatDateTime(nextCutoffTime));
            JsonNode fieldInfo = formData.get("ChangeSchedule_StartEndTime");
            if (fieldInfo == null || fieldInfo instanceof NullNode) {
                throw new BaseException("Please select Change Schedule Start & End Time");
            }
            JsonNode startDateNode = fieldInfo.get("startDate");
            Long startDateLong = startDateNode.asLong();
            DateTime startDate = DateUtil.date(startDateLong);
            if (startDate.before(nextCutoffTime)) {
                ObjectNode formDataResult = (ObjectNode) formData;

                ArrayNode arrayNode = JsonUtils.createArrayNode();
                arrayNode.add(JsonUtils.createObjectNode()
                        .put("groupId", groupBean.getGroupId())
                        .put("groupName", groupBean.getGroupName()));
                if (formDataResult.get(JudgeCmTeamApprove.l1HalfChangeMangerGroupCode) == null) {
                    formDataResult.set(JudgeCmTeamApprove.l1HalfChangeMangerGroupCode, arrayNode);
                }
                formDataResult.put(JudgeCmTeamApprove.needL1Dot5CmTeamValueCode, "Yes");
                formDataResult.put(JudgeCmTeamApprove.needL1Dot5CmTeamCode, dictInfoMap.getOrDefault("Yes", JudgeCmTeamApprove.defaultYesDictId));
                btnActionBo.setFormData(formDataResult);
            }
        }
    }

    private static void aimlErrorHandler(BtnCallResultBo resultBo) {
        ObjectNode objectNode = JsonUtils.createObjectNode();
        objectNode.put("needDialog", false);
        objectNode.put("aiReqError", true);
        objectNode.put("aiReqErrorMsg", "Not able to get Risk Score from AI. Please retry CR submission.");
        resultBo.setSkipSysExecutor(true);
        resultBo.setResultData(objectNode);
        resultBo.setMsg("Not able to get Risk Score from AI. Please retry CR submission.");
    }


    private DateTime getNextCutoffTime() {
        int count = 1;
        int nextFlag = 1;
        String nextDayDate = null;
        while (nextFlag > 0) {
            String nextSql = "SELECT count(*) as total,ADDDATE(CURDATE(), INTERVAL " + count + " DAY) as dateStr,DAYNAME(ADDDATE(CURDATE(), INTERVAL " + count + " DAY)) as dayName  from freezed_holidays where  freeze_date = (CURDATE()+ INTERVAL " + count + " DAY) and upper(freeze_type) = upper('Public Holiday')";
            Map<String, Object> nextResult = jdbcTemplate.queryForMap(nextSql);
            nextFlag = Integer.parseInt(nextResult.get("total").toString());
            nextDayDate = nextResult.get("dateStr").toString();
            String nextDayName = nextResult.get("dayName").toString();
            if (("Friday").equals(nextDayName) && nextFlag == 1) {
                count++;
            }
            if (("Sunday").equals(nextDayName) || ("Saturday").equals(nextDayName)) {
                nextFlag = 1;
            }
            count++;
        }
        return DateUtil.parseDate(nextDayDate).setField(DateField.HOUR_OF_DAY, 13).setField(DateField.MINUTE, 59).setField(DateField.SECOND, 59).setField(DateField.MILLISECOND, 999);
    }

    /**
     * @desc 提交获取审批结果
     * @author cade.liu
     * @since 2025/1/05
     */
    private void calculateSignOffResult(String workOrderId) {
        Boolean result = false;
        String url = null;
        String response = null;
        try {
            // 调用定制服务接口
            String dosmCustomDomainName = getConfig(DOSM_CUSTOM_SERVICE_NAME_PREFIX) + getConfig(DOSM_CUSTOM_SERVICE_NAME);
            log.info("dosmCustomDomainName:{}", dosmCustomDomainName);
            url = dosmCustomDomainName + "/dcs/dosm/signOff/calculateSignOffResult";
            response = this.get(url + "?workOrderId=" + workOrderId, null);
            log.info("calculateSignOffResult response:{}", response);
            // 解析返回结果
            if (StringUtils.isBlank(response)) {
                log.error("response is null");
                throw new BaseException("Signoff verification failure");
            }
            ResultDataEntity<Boolean> resultDataEntity = JsonUtils.parseObject(response, ResultDataEntity.class);

            if (resultDataEntity.isSuccess() && resultDataEntity.getData() != null) {
                result = resultDataEntity.getData();
            }
            saveApiLog(url, workOrderId, response, workOrderId, "SIGNOFF_RESULT", true);
        } catch (Exception e) {
            log.error("get signOff approval result error:{}", e);
            saveApiLog(url, workOrderId, response, workOrderId, "SIGNOFF_RESULT", false);
            throw new BaseException("Signoff verification failure");
        }

        // 判断是否通过
        if (!result) {
            throw new BaseException("You still have pending Signoff to submit CR.");
        }
    }

    private Boolean compareChangeCategory(JsonNode aimlResult, BtnActionBo btnActionBo, JsonNode formData, BtnCallResultBo resultBo) {
        JsonNode changeCategoryNode = formData.get("Change_Category");
        String changeCategory = null;
        if (changeCategoryNode instanceof TextNode) {
            changeCategory = changeCategoryNode.asText();
        }
        if (changeCategory == null) {
            throw new BaseException("please input change Category");
        }
//        return true;
        return changeCategory.equals("Low") && isHighAiml(aimlResult);
    }

    private boolean isHighAiml(JsonNode aimlResult) {
        JsonNode data = aimlResult.get("data");
        JsonNode rowData = data.get("rowData");
        JsonNode risk = rowData.get("risk");
        String text = risk.asText();
        return text.equalsIgnoreCase("high");
    }


    public void saveApiLog(String requestUrl, String requestBody, String responseBody, String workOrderNo, String apiModule, boolean requestStatus) {
        try {
            jdbcTemplate.execute("insert into dbs_api_log (id,request_url,request_body,response_body,work_order_no,api_module,request_status) values(?,?,?,?,?,?,?)",
                    (PreparedStatementCallback<Integer>) ps -> {
                        ps.setLong(1, IdUtil.getSnowflakeNextId());
                        ps.setString(2, requestUrl);
                        ps.setString(3, requestBody);
                        ps.setString(4, responseBody);
                        ps.setString(5, workOrderNo);
                        ps.setString(6, apiModule);
                        ps.setBoolean(7, requestStatus);
                        return ps.executeUpdate();
                    });
        } catch (Exception ignore) {

        }
    }


    private JsonNode queryAimlResult(BtnActionBo btnActionBo, JsonNode formData) {
        AimlReq aimlReq = new AimlReq();
        List<String> columns = new ArrayList<>();
        List<Object> dataItem = new ArrayList<>();
        columns.add("ChangeSchedStartDateTime");
        columns.add("ChangeSchedEndDateTime");
        JsonNode changeScheduleStartEndTime = formData.get("ChangeSchedule_StartEndTime");
        if (changeScheduleStartEndTime != null) {
            dataItem.add(DateUtil.format(new Date(changeScheduleStartEndTime.get("startDate").asLong()), "MM/dd/yyyy HH:mm"));
            dataItem.add(DateUtil.format(new Date(changeScheduleStartEndTime.get("endDate").asLong()), "MM/dd/yyyy HH:mm"));
        } else {
            throw new BaseException("changeScheduleStartEndTime can not be null");
        }

        columns.add("ApplicationImpacted");
        List<String> applicationImpactedCodes = getMultiValues(formData, "applicationImpacte_value");
        dataItem.add(String.join(",", applicationImpactedCodes));

        columns.add("OtherApplicationImpacted");
        List<String> otherApplicationImpactedValue = getMultiValues(formData, "OtherApplicationImpacted_value");
        dataItem.add(String.join(",", otherApplicationImpactedValue));

        DataSource dataSource = SpringContextUtils.getBean(DataSource.class);
        List<Entity> entities = getAppCodeList(applicationImpactedCodes, dataSource);

        columns.add("CyberarkObjects");
        JsonNode cyberObjects = formData.get("CyberArk_Object");
        dataItem.add(cyberObjects.asText());

        columns.add("ApproverGroup3");
        String approverGroup3 = getApproverGroupName(formData, "ApproverGroup3");
        dataItem.add(approverGroup3);

        columns.add("ApproverGroup2");
        String approverGroup2 = getApproverGroupName(formData, "ApproverGroup2");
        dataItem.add(approverGroup2);

        columns.add("Critical business services impacted");
        String impactedBusinessServices = getSingleValue(formData, "Listtheimpactedbythechange");
        dataItem.add(impactedBusinessServices);

        columns.add("RegressionTesting");
        String regressiontesting = getSingleValue(formData, "regressiontesting_value");
        dataItem.add(regressiontesting);

        columns.add("UAT");
        String uat = getSingleValue(formData, "uat_value");
        dataItem.add(uat);

        columns.add("RPCCB_Number");
        dataItem.add("");

        columns.add("MASAppcodeFlag");
        long count = entities.stream().filter(item -> item.get("mas").equals("Y")).count();
        dataItem.add(count > 0 ? "Yes" : "No");

        columns.add("Live Verification (LV)");
        String lv = getSingleValue(formData, "liveVerificationAfterImplementat_value");
        dataItem.add(LvEnum.ofItsmValue(lv).getAimlValue());

        columns.add("CountryImpacted");
        List<String> countryImpacteValue = getMultiValues(formData, "CountryImpacte_value");
        dataItem.add(String.join(",", countryImpacteValue));

        columns.add("CountryOfOrigin");
        String countryOfOrigin = getSingleValue(formData, "countryOfOrigin_value");
        dataItem.add(countryOfOrigin);

        columns.add("ChangeType");
        String changeTypeValue = getSingleValue(formData, "changeType_value");
        dataItem.add(changeTypeValue);

        columns.add("changeGroup");
        String changeGroupValue = getSingleValue(formData, "changeGroup_value");
        ChangeGroupEnum changeGroupEnum = ChangeGroupEnum.ofItsmValue(changeGroupValue);
        dataItem.add(changeGroupEnum.getAimlValue());


        columns.add("Location");
        String locationValue = getSingleValue(formData, "location_value");
        dataItem.add(locationValue);

        columns.add("InfraOnStandby");
        // https://yunzhihui.feishu.cn/record/HjgcriiBIeJGWCcadFicgHxRnwf
        dataItem.add("No");

        columns.add("ApplicationCategory");
        Optional<Integer> appCat = entities.stream().map(entity -> entity.getInt("app_cat_level")).max(Integer::compareTo);
        if (appCat.isPresent()) {
            dataItem.add(appCat.get());
        } else {
            dataItem.add(1);
        }

        columns.add("ChangeCategory");
        String chCategoryValue = getSingleValue(formData, "Change_Category");
        if (chCategoryValue.equals("High")) {
            dataItem.add(1);
        } else {
            dataItem.add(3);
        }


        columns.add("ChangeDescription");
        String changeDescribtion = getSingleValue(formData, "changeDescribtion");
        dataItem.add(changeDescribtion);

        columns.add("ImplementationPlan");
        List<String> implementInfos = new ArrayList<>();
        String aPreImplementationTasksActivi = "A_Pre_Implementation_Tasks_Activi";
        String aTaskActivityDescription = "A_Task_Activity_Description";
        addTableFieldInfo(formData, implementInfos, aPreImplementationTasksActivi, aTaskActivityDescription);

        String bImplementationTasksActiv = "B_Implementation_Tasks_Activ";
        String bTaskActivityDescription = "B_Task_Activity_Description";
        addTableFieldInfo(formData, implementInfos, bImplementationTasksActiv, bTaskActivityDescription);

        String cPostImplementationPlan = "C_Post_Implementation_Plan";
        String cPlanDescription = "C_Plan_Description";
        addTableFieldInfo(formData, implementInfos, cPostImplementationPlan, cPlanDescription);
        if (implementInfos.isEmpty()) {
            dataItem.add("NA");
        } else {
            dataItem.add(String.join(",", implementInfos));
        }

        columns.add("ChangeRequestorGroups");
        dataItem.add(getApproverGroupName(formData, "changeRequestorGroups"));

        columns.add("ImplementerGroup");
        List<String> implementerGroups = getMultiGroupName(formData, "implementInfo", "implementerGroupPerformingTaskAc");
        dataItem.add(String.join(",", implementerGroups));

        columns.add("Data Patch - Number of records");
        if (formData.has("dataPatchNumberOfRecords")) {
            JsonNode dataPatchNumberOfRecords = formData.get("dataPatchNumberOfRecords");
            if (dataPatchNumberOfRecords != null && !(dataPatchNumberOfRecords instanceof NullNode)) {
                dataItem.add(dataPatchNumberOfRecords.asInt());
            } else {
                dataItem.add(0);
            }
        } else {
            dataItem.add(0);
        }

        columns.add("ChangeSummary");
        String changeSummary = getSingleValue(formData, "changeSummary");
        dataItem.add(changeSummary);

        columns.add("Rollback Testing");
        String reversionBackoutRollbackTestingValue = getSingleValue(formData, "ReversionBackoutRollbackTesting_value");
        dataItem.add(reversionBackoutRollbackTestingValue);

        aimlReq.setColumns(columns);
        aimlReq.setData(Collections.singletonList(dataItem));
        String crRiskScoring = getConfig("dosm.custom.crRiskScoring");
        String aimlReqBody = JsonUtils.toJsonString(aimlReq);
        log.error("CommitButtonExt:crRiskScoring:{},body:{}", crRiskScoring, aimlReqBody);
        HttpRequest post = HttpUtil.createPost(crRiskScoring).timeout(60000);
        post.body(aimlReqBody);
        String appCode = getConfig("dosm.custom.appCode");
        String appKey = getConfig("dosm.custom.appKey");
        post.header("appCode", appCode);
        post.header("appKey", appKey);
        post.header("Content-Type", "application/json");
        try (HttpResponse execute = post.execute()) {
            String body = execute.body();
            log.error("CommitButtonExt:result:{}", body);
            saveApiLog(crRiskScoring, aimlReqBody, body, btnActionBo.getWorkOrderId(), "CR_RISK_SCORING", true);
            return JsonUtils.parseJsonNode(body);
        } catch (Exception e) {
            log.error("CommitButtonExt:req error", e);
            saveApiLog(crRiskScoring, aimlReqBody, null, btnActionBo.getWorkOrderId(), "CR_RISK_SCORING", false);
            return null;
        }
    }

    private static void addTableFieldInfo(JsonNode formData, List<String> implementInfos, String implementInfoTableName, String implementInfoTableFieldName) {
        if (formData.has(implementInfoTableName)) {
            JsonNode implementInfo = formData.get(implementInfoTableName);
            if (implementInfo != null && implementInfo instanceof ArrayNode) {
                for (JsonNode implementInfoItem : implementInfo) {
                    JsonNode rowData = implementInfoItem.get("rowData");
                    JsonNode taskTypeDescription = rowData.get(implementInfoTableFieldName);
                    implementInfos.add(taskTypeDescription.asText());
                }
            }
        }
    }

    private static List<Entity> getAppCodeList(List<String> applicationImpactedCodes, DataSource dataSource) {
        List<Entity> entities = new ArrayList<>();
        Db db = DbUtil.use(dataSource);
        Condition condition = new Condition();
        condition.setField("app_id");
        condition.setValue(applicationImpactedCodes, true);
        try {
            entities = db.find(Entity.create().setTableName("ic_appcode").setFieldNames("mas", "app_cat_level").set("app_id", condition));

        } catch (SQLException e) {

        }
        return entities;
    }

    private List<String> getMultiGroupName(JsonNode formData, String implementInfo, String implementerGroupPerformingTaskAc) {
        if (!formData.has(implementInfo)) {
            return Collections.emptyList();
        }
        JsonNode jsonNode = formData.get(implementInfo);
        if (jsonNode == null || jsonNode instanceof NullNode) {
            return Collections.emptyList();
        }
        List<String> groups = new ArrayList<>();
        for (JsonNode node : jsonNode) {
            JsonNode row = node.get("rowData");
            groups.add(getApproverGroupName(row, implementerGroupPerformingTaskAc));
        }
        return groups;
    }

    private String getSingleValue(JsonNode formData, String uat) {
        if (formData.has(uat)) {
            return formData.get(uat).asText();
        }
        return null;
    }

    private String getApproverGroupName(JsonNode formData, String approverGroup3) {
        if (!formData.has(approverGroup3)) {
            return null;
        }
        JsonNode approveGroupInfo = formData.get(approverGroup3);
        if (approveGroupInfo == null || approveGroupInfo instanceof NullNode) {
            return null;
        }
        if (approveGroupInfo instanceof ArrayNode && approveGroupInfo.size() > 0) {
            return approveGroupInfo.get(0).get("groupName").asText();
        }
        return approveGroupInfo.get("groupName").asText();
    }

    private static List<String> getMultiValues(JsonNode formData, String applicationImpacte_value) {
        if (!formData.has(applicationImpacte_value)) {
            return Collections.emptyList();
        }
        JsonNode applicationImpacte = formData.get(applicationImpacte_value);
        if (applicationImpacte == null || applicationImpacte instanceof NullNode) {
            return Collections.emptyList();
        }
        List<String> applicationImpactedCodes = new ArrayList<>();
        for (JsonNode applicationCode : applicationImpacte) {
            applicationImpactedCodes.add(applicationCode.asText());
        }
        return applicationImpactedCodes;
    }

    private Boolean checkNeedAiml(BtnActionBo btnActionBo, JsonNode formData) {
        //1. 是否为normal cr
        boolean cr = btnActionBo.getBizKey().startsWith("CR");
        if (!cr) {
            return false;
        }
        if (isSignOffNode(btnActionBo)) return true;

        //2. 是否第一次提交
        return checkIsFirstCommit(btnActionBo, formData);
    }

    private boolean isSignOffNode(BtnActionBo btnActionBo) {
        String nodeId = btnActionBo.getNodeId();
        //检查是否需要是审批节点之前的节点
        if (StrUtil.startWithIgnoreCase(nodeId, CR_SIGNOFF_NODE_ID)) {
            return true;
        }
        return false;
    }

    private Boolean checkIsFirstCommit(BtnActionBo btnActionBo, JsonNode formData) {
        if (formData.has("riskScore")) {
            return false;
        }
        if (formData.has("aiReqError")) {
            return false;
        }
        return true;
    }

    @Override
    public void after(BtnActionBo btnActionBo, Object sysResultData, BtnCallResultBo resultBo) {
        if (RCA_PRE_FIX.equals(btnActionBo.getMdlDefKey())) {
            log.debug("rca 工单修改表单数据,同步修改阻塞表,rcaId:{}", btnActionBo.getWorkOrderId());
            JsonNode rcaFormDataJson = JsonUtils.parseJsonNode(btnActionBo.getFormData());
            if (rcaFormDataJson.has("impactedapplication_value")) {
                String impactedapplicationValue = rcaFormDataJson.get("impactedapplication_value").asText();
                String rca_work_order_id = btnActionBo.getWorkOrderId();
                jdbcTemplate.update("update dbs_rca_application_pending set application_name = ?   where rca_work_order_id = ?", impactedapplicationValue, rca_work_order_id);
                log.debug("rca 工单修改表单数据,同步修改阻塞表成功,rcaId:{},impactedapplication:{}", btnActionBo.getWorkOrderId(), impactedapplicationValue);
            }
        }
    }

    @Override
    public void rollbackBefore(BtnActionBo btnActionBo, Object sysResultData, Object rollbackParam) {

    }

    public static String getConfig(String configKey) {
        Environment bean = SpringContextUtils.getBean(Environment.class);
        return bean.getProperty(configKey);
    }

    public void validateRca(BtnActionBo btnActionBo, BtnCallResultBo resultBo) {
        BtnTypeEnum btnType = btnActionBo.getBtnType();
        if (BtnTypeEnum.COMMIT == btnType && isSignOffNode(btnActionBo)) {
            Object formData = btnActionBo.getFormData();
            JsonNode formDataNode = com.cloudwise.dosm.core.utils.JsonUtils.parseJsonNode(formData);
            List<String> appCodes = getMultiValues(formDataNode, APP_CODE_FIELD);
            if (CollUtil.isEmpty(appCodes)) {
                throw new BaseException("Application(s) Impacted can not be empty");
            }
            String appCode = appCodes.get(0);
            String sql = "select rca_work_order_id from dbs_rca_application_pending where application_name='{}' and is_pending =true";
            sql = StrUtil.format(sql, appCode);
            List<String> query = jdbcTemplate.query(sql, new RowMapper<String>() {
                @Override
                public String mapRow(ResultSet rs, int rowNum) throws SQLException {
                    String rca_work_order_id = rs.getString("rca_work_order_id");
                    return rca_work_order_id;
                }
            });
            if (CollUtil.isNotEmpty(query)) {
                String rcaWorkOrderId = query.get(0);
                MdlInstance mdlInstance = mdlInstanceMapper.selectById(rcaWorkOrderId, btnActionBo.getTopAccountId());
                if (mdlInstance != null) {
                    throw new BaseException(StrUtil.format("The selected application App code {} has a peending RCA. Please complete the RCA({}) for this application App code before creating a CR.", appCode, mdlInstance.getBizKey()));
                }
            }
        }
    }

    public static String get(String url, Map<String, String> header) {

        String response = HttpRequest.post(url)
                .headerMap(header, true)
                .execute()
                .body();
        log.info("返回结果:【{}】", response);
        return response;
    }

}
