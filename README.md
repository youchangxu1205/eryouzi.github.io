package com.cloudwise.dosm.button;

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
import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.IButtonActionExt;
import com.cloudwise.dosm.api.bean.adv.extend.BtnActionBo;
import com.cloudwise.dosm.api.bean.adv.extend.BtnCallResultBo;
import com.cloudwise.dosm.api.bean.enums.BtnTypeEnum;
import com.cloudwise.dosm.api.bean.form.GroupBean;
import com.cloudwise.dosm.api.bean.utils.JsonUtils;
import com.cloudwise.dosm.biz.instance.dao.MdlInstanceMapper;
import com.cloudwise.dosm.biz.instance.dto.BtnActionParam;
import com.cloudwise.dosm.biz.instance.entity.MdlInstance;
import com.cloudwise.dosm.biz.instance.service.btn.impl.BtnStartServiceImpl;
import com.cloudwise.dosm.biz.instance.service.mdlins.order.WorkOrderLinkRecodService;
import com.cloudwise.dosm.biz.instance.service.mdlins.order.WorkOrderLinkService;
import com.cloudwise.dosm.biz.instance.vo.WorkOrderLinkParamVo;
import com.cloudwise.dosm.bpm.api.action.enums.BtnEnum;
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
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.env.Environment;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.PreparedStatementCallback;
import org.springframework.jdbc.core.RowMapper;

import javax.annotation.Resource;
import javax.sql.DataSource;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.*;

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


    @Override
    public List<BtnTypeEnum> getActionList() {
        List<BtnTypeEnum> button = new ArrayList<>();
        button.add(BtnTypeEnum.COMMIT);
        return button;
    }

    @Override
    public void before(BtnActionBo btnActionBo, BtnCallResultBo resultBo) {
        String bizKey = btnActionBo.getBizKey();
        if (bizKey.startsWith("CR")) {
            handleCrAction(btnActionBo, resultBo, bizKey);
        }
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
        log.error("cr action formData:{},{},{},{}", btnActionBo.getNodeName(), btnActionBo.getWorkOrderId(), bizKey, formData);
        if (formData == null) {
            throw new BaseException("form can not be empty!");
        }

        validateCyberArkObjects(formData);

        if (formData.has("notNeedAiml")) {
            JsonNode notNeedAiml = formData.get("notNeedAiml");
            if (notNeedAiml.asBoolean()) {
                ((ObjectNode) formData).remove("notNeedAiml");
                btnActionBo.setFormData(formData);
                log.error("AimlCommitButtonExt:{}", formData);
                return;
            }
        }
        if (formData.has("aiReqError")) {
            JsonNode aiReqError = formData.get("aiReqError");
            if (aiReqError.asBoolean()) {
                ((ObjectNode) formData).remove("aiReqError");
                btnActionBo.setFormData(formData);
                return;
            }
        }

        // 判断是否为 signOff节点
        if (isSignOffNode(btnActionBo)) {
            ((ObjectNode) formData).put("CRSubmissionDatetime", System.currentTimeMillis());
            log.error("set CRSubmissionDatetime:{}", formData);

            JudgeCmTeamApprove judgeCmTeamApprove = new JudgeCmTeamApprove(jdbcTemplate, dataDictService, groupFacade);
            Map<String, String> dictInfoMap = judgeCmTeamApprove.queryDictLabel2Value(JudgeCmTeamApprove.YesOrNoDictCode);
            GroupBean groupBean = judgeCmTeamApprove.getGroupBeanByName(l1HalfChangeMangerGroup);

            handleSignOffNodeAction(btnActionBo, resultBo, bizKey, formData, dictInfoMap, groupBean);

            //L1.5 CM Team
            this.handlerNeedL1HalfCmTeamApprove(judgeCmTeamApprove, btnActionBo, groupBean);
            handleCrStatus2Open(btnActionBo,formData);
        }

        if (isCloseNode(btnActionBo)) {
            handleCloseNodeAction(btnActionBo, resultBo, bizKey, formData);
        }

        managedBlockSubmit(formData);

    }

    private void handleCrStatus2Open(BtnActionBo btnActionBo,JsonNode formData) {
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
    }

    //handle L1.5 CM Team approve
    private void handlerNeedL1HalfCmTeamApprove(JudgeCmTeamApprove judgeCmTeamApprove, BtnActionBo btnActionBo, GroupBean groupBean) {
        judgeCmTeamApprove.handlerNeedL1Approve(btnActionBo, groupBean);
    }

    private static void managedBlockSubmit(JsonNode formData) {
        if (formData.has("CRClassification_value")) {
            JsonNode countryOfOriginJson = formData.get("CRClassification_value");
            JsonNode table = formData.get("Reversion_Plan");
            Long minStartValue = null;
            Long maxEndValue = null;
            if (countryOfOriginJson.asText().equals("Managed")) {
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
                    throw new BaseException("CyberArk Objects in the ‘Pre-Production’ status cannot be included in the CR:" + String.join(",", query));
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
        JsonNode crStatus = formData.get("crStatus_value");
        if (crStatus == null) {
            throw new BaseException("crStatus can not be null");
        }
        String crStatusValue = crStatus.asText();
        if (crStatusValue.equals("Closed Successful")) {
            JSONObject paramBody = new JSONObject();
            JsonNode droneTicket = formData.get("DroneTicket");
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
            if (formData.has("crStatus") && CLOSE_STATUS_LIST.contains(formData.get("crStatus_value").asText())) {
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
            log.info("请求数据:{}", JSONUtil.toJsonStr(resultHttpDetail));
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

    private void handleSignOffNodeAction(BtnActionBo btnActionBo, BtnCallResultBo resultBo, String bizKey, JsonNode formData, Map<String, String> dictInfoMap, GroupBean groupBean) {
        // signOff节点 提交需要判断 审批结果
        this.calculateSignOffResult(btnActionBo.getWorkOrderId());
        Boolean isNeedAiml = checkNeedAiml(btnActionBo, formData);
        if (isNeedAiml) {
            try {
                JsonNode aimlResult = queryAimlResult(btnActionBo, formData);
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
                    formDataResult.set("riskScore", aimlRowData.get("score"));
                    formDataResult.set("riskThreshold", aimlRowData.get("risk_threshold"));
                    JsonNode explainabilityScore = aimlRowData.get("explainability_scores");
                    formDataResult.set("explainabilityScores", explainabilityScore);
                    formDataResult.set("explainability", aimlRowData.get("explainability"));
                    formDataResult.set("explainabilityBase", aimlRowData.get("explainability_base"));
                    formDataResult.put("recommendedRisk", "high");
//                    formDataResult.set("recommendedRisk", aimlRowData.get("risk"));
                    formDataResult.set("featureValues", aimlRowData.get("feature_values"));
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
            }
        } else {
            resultBo.setSkipSysExecutor(false);
        }

        if (!resultBo.isSkipSysExecutor()) {
            DateTime currentTime = DateUtil.date();
            if (currentTime.getField(DateField.HOUR_OF_DAY) >= 17) {
                DateTime nextCutoffTime = getNextCutoffTime();
                log.error("CommitButton:nextCutoffTime:{}", DateUtil.formatDateTime(nextCutoffTime));
                JsonNode fieldInfo = formData.get("ChangeSchedule_StartEndTime");
                JsonNode startDateNode = fieldInfo.get("startDate");
                Long startDateLong = startDateNode.asLong();
                DateTime startDate = DateUtil.date(startDateLong);
                if (startDate.before(nextCutoffTime)) {
                    ObjectNode formDataResult = (ObjectNode) formData;
                    ArrayNode arrayNode = JsonUtils.createArrayNode();
                    arrayNode.add(JsonUtils.createObjectNode()
                            .put("groupId", groupBean.getGroupId())
                            .put("groupName", groupBean.getGroupName()));
                    formDataResult.set(JudgeCmTeamApprove.l1HalfChangeMangerGroupCode, arrayNode);
                    formDataResult.put(JudgeCmTeamApprove.needL1Dot5CmTeamValueCode, "Yes");
                    formDataResult.put(JudgeCmTeamApprove.needL1Dot5CmTeamCode, dictInfoMap.getOrDefault("Yes", JudgeCmTeamApprove.defaultYesDictId));
                    btnActionBo.setFormData(formDataResult);
                }
            }
        }


        btnActionBo.setFormData(formData);


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
        //TODO just for test to set this true
        return true;
//        return changeCategory.equals("Low") && isHighAiml(aimlResult);
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
            dataItem.add(DateUtil.formatDateTime(new Date(changeScheduleStartEndTime.get("startDate").asLong())));
            dataItem.add(DateUtil.formatDateTime(new Date(changeScheduleStartEndTime.get("endDate").asLong())));
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
        dataItem.add(null);

        columns.add("MASAppcodeFlag");
        long count = entities.stream().filter(item -> item.get("mas").equals("Y")).count();
        dataItem.add(count > 0 ? "Yes" : "No");

        columns.add("Live Verification (LV)");
        String lv = getSingleValue(formData, "liveVerificationAfterImplementat_value");
        dataItem.add(lv);

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
        dataItem.add(changeGroupValue);

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
        dataItem.add(chCategoryValue);

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
        HttpRequest post = HttpUtil.createPost(crRiskScoring);
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


    public static String get(String url, Map<String, String> header) {
        log.info("请求参数,url:【{}】,请求头:【{}】", url, JSONUtil.toJsonStr(header));
        String response = HttpRequest.post(url)
                .headerMap(header, true)
                .execute()
                .body();
        log.info("返回结果:【{}】", response);
        return response;
    }

}




package com.cloudwise.dosm.dbs.trigger;

import cn.hutool.core.collection.CollectionUtil;
import cn.hutool.core.date.DateTime;
import cn.hutool.core.date.DateUnit;
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.text.CharSequenceUtil;
import cn.hutool.core.util.IdUtil;
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
import com.cloudwise.dosm.api.bean.form.field.NumberField;
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
import com.cloudwise.dosm.biz.instance.table.dao.MdlInstanceTableDataMapper;
import com.cloudwise.dosm.biz.instance.table.service.MdlInstanceTableDataService;
import com.cloudwise.dosm.biz.instance.table.vo.MdlInstanceTableRowVo;
import com.cloudwise.dosm.bpm.api.action.enums.EventTypeEnum;
import com.cloudwise.dosm.bpm.api.instance.MdlInstanceApiService;
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
import com.cloudwise.dosm.core.utils.RedisUtil;
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
import com.cloudwise.dosm.file.entity.FileInfoPo;
import com.cloudwise.dosm.file.service.IFileInfoService;
import com.cloudwise.dosm.form.biz.service.FormFieldService;
import com.cloudwise.douc.dto.v3.user.UserConditionReq;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.NullNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.google.common.collect.Lists;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.jetbrains.annotations.NotNull;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.env.Environment;
import org.springframework.dao.DataAccessException;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.PreparedStatementCallback;
import org.springframework.jdbc.core.PreparedStatementSetter;
import org.springframework.jdbc.core.ResultSetExtractor;
import org.springframework.jdbc.core.RowMapper;

import java.io.File;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;

import static com.cloudwise.dosm.dbs.trigger.SignoffConstants.SignoffItem.DATA_CENTER_OPS_BATCH_SIGNOFF;
import static com.cloudwise.dosm.dbs.trigger.SignoffConstants.SignoffItem.IDR_SIGNOFF_PROCUTOVER;

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
    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Autowired
    private MdlInstanceTableDataService mdlInstanceTableDataService;
    @Autowired
    private FormFieldService formFieldService;
    @Autowired
    private IFileInfoService fileInfoService;


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
        if (crStatus != null) {
            updateCrStatus2Form(ctx, crStatus, param);
        }
        HttpResponse response = null;
        JsonNode nodeIdNode = originMessageNode.get("nodeId");
        String nodeId = nodeIdNode.asText();
        JsonNode eventTypeNode = originMessageNode.get("eventType");
        String eventType = eventTypeNode.asText();
        if (nodeId.equals(param.getCreateNodeId()) && EventTypeEnum.NODE_EXIT.getCode().equals(eventType)) {
            return;
        }

        boolean isNewCr = isNewCr(originMessageNode, param, workOrder);
        String url = isNewCr ? param.getPushUrl() : param.getUpdateUrl();
        String jsonString = null;
        Map<String, Object> datas = new HashMap<>();
        datas.put("workOrder", workOrder);
        datas.put("originMessage", originMessageNode);
        try {
            if (crStatus == null) {
                Map<String, FieldInfo> formDataMap = workOrder.getFormDataMap();
                FieldInfo fieldInfo = formDataMap.get(param.getCrStatusFieldCode());
                SelectField fieldValueObj = (SelectField) fieldInfo.getFieldValueObj();
                crStatus = CrStatusEnums.getByLabel(fieldValueObj.getLabel());
            }
            String appCode = getConfig("dosm.custom.appCode");
            String appKey = getConfig("dosm.custom.appKey");
            List<UserInfo> createdBys = getUsers(Lists.newArrayList(Long.valueOf(workOrder.getCreatedBy())));
            Map<String, List<MdlInstanceTableRowVo>> tableMap = mdlInstanceTableDataService.getTableRowVoMapByWorkOrderId(workOrder.getId());
            ObjectNode tables = com.cloudwise.dosm.core.utils.JsonUtils.parseOrCreateObjectNode(tableMap);
            //将表格数据回填到表单展示
            ObjectNode formData = JsonUtils.createObjectNode();
            for (Map.Entry<String, JsonNode> tableE : tables.properties()) {
                formData.set(tableE.getKey(), tableE.getValue());
            }
            Map<String, FieldInfo> formDataMap = formFieldService.getFormData(workOrder.getFormId(), null, formData);
            workOrder.getFormDataMap().putAll(formDataMap);
            JsonNode makeParam = makeParam(param, workOrder, createdBys, crStatus, isNewCr);
            log.info("CrStatusSyncTriggermakeParam:{}", makeParam);
            jsonString = JsonUtils.toJsonString(makeParam);
            response = HttpUtil.createPost(url).header("Content-Type", "application/json").header("appCode", appCode).header("appKey", appKey).body(jsonString).execute();
            String body = response.body();
            log.info("CrStatusSyncTriggermakeResult:{}", body);
            JsonNode node = JsonUtils.parseJsonNode(body);

            datas.put("reqParam", makeParam);
            datas.put("workOrder", workOrder);
            String jsonString1 = com.cloudwise.dosm.core.utils.JsonUtils.toJsonString(datas);
            if ("200".equals(node.get("code").asText())) {
                JsonNode data = node.get("data");
                JsonNode status = data.get("Status");
                JsonNode message = data.get("Message");
                if ("FAILED".equals(status.asText())) {
                    ResultBean resultBean = ctx.getResult();
                    resultBean.setError(message.asText());
                    ctx.setResult(resultBean);

                    saveApiLog(url, jsonString1, body, workOrder.getBizKey(), "crStatusSync", false);
                } else {
                    saveApiLog(url, jsonString1, body, workOrder.getBizKey(), "crStatusSync", true);
                }
            } else {
                ResultBean resultBean = ctx.getResult();
                resultBean.setError(body);
                ctx.setResult(resultBean);
                saveApiLog(url, jsonString1, body, workOrder.getBizKey(), "crStatusSync", false);
            }

            sendApprovalMsg(crStatus, ctx, createdBys);
        } catch (Exception e) {
            log.error("CrStatusSyncTrigger send to ichamp error:{}", e.getMessage(), e);
            ResultBean resultBean = ctx.getResult();
            resultBean.setError("Sync data to ichamp fail: data:" + e.getMessage());
            ctx.setResult(resultBean);
            saveApiLog(url, jsonString, e.getMessage(), workOrder.getBizKey(), "crStatusSync", false);
        } finally {
            if (response != null) {
                response.close();
            }
        }
    }

    public String getConfig(String configKey) {
        Environment bean = com.cloudwise.dosm.core.utils.SpringContextUtils.getBean(Environment.class);
        return bean.getProperty(configKey);
    }

    public JsonNode makeParam(ExtParam extParam, WorkOrderBean workOrder, List<UserInfo> createdBys, CrStatusEnums crStatus, Boolean isNew) {
        ObjectNode result = JsonUtils.createObjectNode();
        ObjectNode data = JsonUtils.createObjectNode();
        List<String> userOneBankIds = getUserOneBankIdsByUser(createdBys);
        ObjectNode formData = JsonUtils.createObjectNode();
        makeFormInfo(formData, workOrder, extParam);
        result.put("action", isNew ? "Create" : "Modify");
        if (!isNew) {
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
        return result;
    }

    public boolean isNewCr(JsonNode originMessageNode, ExtParam extParam, WorkOrderBean workOrderBean) {
        JsonNode nodeIdNode = originMessageNode.get("nodeId");
        String nodeId = nodeIdNode.asText();
        JsonNode eventTypeNode = originMessageNode.get("eventType");
        String eventType = eventTypeNode.asText();
        if (nodeId.equals(extParam.getCreateNodeId()) && "WORK_COMMIT".equals(eventType)) {
            return true;
        }
        return false;
    }

    private CrStatusEnums getCrStatus(TriggerContext ctx, ExtParam extParam, WorkOrderBean workOrder) {
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

    public void saveApiLog(String requestUrl, String requestBody, String responseBody, String workOrderNo, String apiModule, boolean requestStatus) {
        try {
            jdbcTemplate.execute("insert into dbs_api_log (id,request_url,request_body,response_body,work_order_no,api_module,request_status) values(?,?,?,?,?,?,?)", (PreparedStatementCallback<Integer>) ps -> {
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

    public void makeFormInfo(ObjectNode formData, WorkOrderBean workOrder, ExtParam extParam) {
        Map<String, FieldInfo> formDataMap = workOrder.getFormDataMap();
        fillFieldMapping(formData, extParam.getFieldMapping(), formDataMap);
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
                    log.error("CrStatusSyncTrigger find table info:{}", itsmTableFieldInfo);
                    ArrayNode jobDetailResult = JsonUtils.createArrayNode();
                    TableFormField jobDetailsSectionField = (TableFormField) jobDetailsSection.getFieldValueObj();
                    Collection<RowData> jobDetailsSectionFieldValue = jobDetailsSectionField.getValue();
                    Map<String, String> fieldMapping1 = tableFieldMapping.getFieldMapping();
                    for (RowData rowData : jobDetailsSectionFieldValue) {
                        ObjectNode jobDetailItem = JsonUtils.createObjectNode();
                        Map<String, FieldInfo> columnDataMap = rowData.getColumnDataMap();
                        for (Map.Entry<String, String> stringStringEntry : fieldMapping1.entrySet()) {
                            FieldInfo fieldValueInfo = columnDataMap.get(stringStringEntry.getValue());
                            fillIchampValue(jobDetailItem, fieldValueInfo, stringStringEntry.getKey());
                        }
                        jobDetailResult.add(jobDetailItem);
                    }
                    itemResult.put(tableFieldMapping.getIchampTableFieldInfo(), jobDetailResult);
                } else {

                    //try to query db?

                    log.error("CrStatusSyncTrigger not find table info:{}", itsmTableFieldInfo);
                }
            }
            if (!itemResult.isEmpty()) {
                formData.put(key, itemResult);
            }
        }
        List<ApproveInfo> approveInfoMapping = extParam.getApproveInfoMapping();
        for (ApproveInfo approveInfo : approveInfoMapping) {
            //if cr Status is not New ,need query approver status
            LambdaQueryWrapper<MdlApproveRecord> wrapper = Wrappers.lambdaQuery(MdlApproveRecord.class).eq(MdlApproveRecord::getWorkOrderId, workOrder.getId()).eq(MdlApproveRecord::getNodeId, approveInfo.getNodeId()).orderByDesc(BasePo::getCreatedTime);
            MdlApproveRecord mdlApproveRecord = mdlApproveRecordMapper.selectOne(wrapper, false);
            log.error("CrStatus:queryApprove:{}", mdlApproveRecord);
            if (mdlApproveRecord != null) {
                if (mdlApproveRecord.getApproveStatus().equals(ApproveStatusEnum.FINISHED.getCode() + "")) {
                    MdlApproveRecordDetail mdlApproveRecordDetail = mdlApproveRecordDetailMapper.selectOne(Wrappers.lambdaQuery(MdlApproveRecordDetail.class).eq(MdlApproveRecordDetail::getApproveRecordId, mdlApproveRecord.getId()), false);
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

        //TODO SIGNOFF
        handleSignoffField(formData, workOrder, extParam);


        formData.put("itsmcrnumber", workOrder.getBizKey());
        formData.put("implementationplan", "https://" + getConfig("CloudwiseDomain") + "/docp/dosm/dosm/orderDetails?showType=handle&id=" + workOrder.getId() + "&isType=&isShowBack=1&type=3d409c4dc7e741539194a528cd9f0d91");
        formData.put("reversionplan", "https://" + getConfig("CloudwiseDomain") + "/docp/dosm/dosm/orderDetails?showType=handle&id=" + workOrder.getId() + "&isType=&isShowBack=1&type=3d409c4dc7e741539194a528cd9f0d91");

    }

    private void handleSignoffField(ObjectNode formData, WorkOrderBean workOrder, ExtParam param) {

        List<SignoffManager> signoffManagers = jdbcTemplate.query("select * from sign_off_manager where work_order_id = ?", new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps) throws SQLException {
                ps.setString(1, workOrder.getBizKey());
            }
        }, new RowMapper<SignoffManager>() {

            @Override
            public SignoffManager mapRow(ResultSet rs, int rowNum) throws SQLException {
                SignoffManager signoffManager = new SignoffManager();
                signoffManager.setArtifact(rs.getString("artifact"));
                signoffManager.setCaseId(rs.getString("case_id"));
                signoffManager.setSignOffType(rs.getString("sign_off_type"));
                signoffManager.setSignOffGroup(rs.getString("sign_off_group"));
                signoffManager.setSignOffUser(rs.getString("sign_off_user"));
                signoffManager.setStatus(rs.getString("status"));
                signoffManager.setRejectionReason(rs.getString("rejection_reason"));
                return signoffManager;
            }
        });

        //IDRSIGNOFFCASEID
        /**
         * query https://confluence.sgp.dbs.com:8443/dcifcnfl/display/IN/27-+INAA-805+%5BInvensys%5D+API+for+IDR+result
         */


        //datacentersignoff
        for (SignoffManager signoffManager : signoffManagers) {
            String signOffType = signoffManager.getSignOffType();
            List<String> signOffTypes = JsonUtils.parseArray(signOffType, String.class);
            for (String offType : signOffTypes) {
                SignoffConstants.SignoffItem signoffItem = SignoffConstants.SignoffItem.fromString(offType);
                if (signoffItem != null) {
                    if (signoffItem == IDR_SIGNOFF_PROCUTOVER) {
                        ArrayNode idrSignoff = JsonUtils.createArrayNode();
                        String appCode = getConfig("dosm.custom.appCode");
                        String appKey = getConfig("dosm.custom.appKey");
                        if (StringUtils.isNotBlank(signoffManager.getCaseId())) {
                            String caseId = signoffManager.getCaseId();
                            String[] caseIds = caseId.split(",");
                            ObjectNode objectNode = JsonUtils.createObjectNode();
                            objectNode.put("CASEID", caseId);
                            HttpResponse response = HttpUtil.createPost(param.getIdrUrl()).header("Content-Type", "application/json")
                                    .header("appCode", appCode).header("appKey", appKey).body(JsonUtils.toJsonString(objectNode)).execute();
                            String body = response.body();
                            JsonNode caseIdResp = JsonUtils.parseJsonNode(body);
                            JsonNode data = caseIdResp.get("data");
                            Map<String, JsonNode> caseIdMap = new HashMap<>();
                            for (JsonNode node : data) {
                                caseIdMap.put(node.get("CASEID").asText(), node);
                            }
                            for (String id : caseIds) {
                                ObjectNode caseIdResult = JsonUtils.createObjectNode();
                                caseIdResult.put("caseid", id);
                                if (caseIdMap.containsKey(id)) {
                                    JsonNode node = caseIdMap.get(id);
                                    JsonNode node1 = node.get("COMPLETIONDATE");
                                    if (node1 != null && !(node1 instanceof NullNode)) {
                                        String text = node1.asText();
                                        DateTime parse = DateUtil.parse(text);
                                        long between = DateUtil.between(parse, new Date(), DateUnit.DAY);
                                        if (between >= 180) {
                                            caseIdResult.put("remarks", "Signoff Exceed 6 months");
                                        } else {
                                            caseIdResult.put("remarks", "Signoff not find");
                                        }
                                    } else {
                                        caseIdResult.put("remarks", "Not approved");
                                    }
                                } else {
                                    caseIdResult.put("remarks", "Signoff not find");
                                }
                                idrSignoff.add(caseIdResult);
                            }
                            formData.put("idrsignoffcaseid", idrSignoff);
                        }
                    }
                    JsonNode signOffUser = JsonUtils.parseJsonNode(signoffManager.getSignOffUser());
                    if (signOffUser != null && !signOffUser.isEmpty()) {
                        if (signoffItem.getSignOffApproverLoginCode() != null) {
                            String userName = signOffUser.get(0).get("userName").asText();
                            formData.put(signoffItem.getSignOffApproverLoginCode(), userName.substring(0, userName.indexOf("(")));
                        }
                        SignOffStatusEnum signOffStatusEnum = SignOffStatusEnum.valueOf(signoffManager.getStatus());
                        if (signOffStatusEnum == SignOffStatusEnum.REJECTED || signOffStatusEnum == SignOffStatusEnum.APPROVED) {
                            if (signoffItem.getSignOffStatusCode() != null) {
                                formData.put(signoffItem.getSignOffStatusCode(), signOffStatusEnum.getDesc());
                            }
                            if (signoffItem.getSignOffRejectionReasonCode() != null) {
                                formData.put(signoffItem.getSignOffRejectionReasonCode(), signoffManager.getRejectionReason() == null ? "" : signoffManager.getRejectionReason());
                            }
                            if (signoffItem.getSignOffUrlCode() != null) {
                                JsonNode node = JsonUtils.parseJsonNode(signoffManager.getArtifact());
                                List<String> urls = new ArrayList<>();
                                for (JsonNode jsonNode : node) {
                                    JsonNode id = jsonNode.get("id");
                                    FileInfoPo fileInfoPo = fileInfoService.getById(workOrder.getAccountId(), id.asText());
                                    String filePath = fileInfoPo.getFilePath();
                                    String fileName = filePath.substring(filePath.indexOf(File.separator) + 1);
                                    try {
                                        String presignedObjectUrl = fileInfoService.getPresignedObjectUrl(fileName.substring(0, filePath.indexOf(File.separator)), fileName, 604800);
                                        urls.add(presignedObjectUrl);
                                    } catch (Exception e) {

                                    }
                                }
                                formData.put(signoffItem.getSignOffUrlCode(), String.join(",", urls));
                            }
                        } else {
                            if (signoffItem.getSignOffStatusCode() != null) {
                                formData.put(signoffItem.getSignOffStatusCode(), "");
                            }
                            if (signoffItem.getSignOffRejectionReasonCode() != null) {
                                formData.put(signoffItem.getSignOffRejectionReasonCode(), "");
                            }
                            if (signoffItem.getSignOffUrlCode() != null) {
                                formData.put(signoffItem.getSignOffUrlCode(), "");
                            }
                        }

                    }
                }
            }
        }
    }

    @Data
    static class SignoffManager {
        private String signOffType;
        private String signOffGroup;
        private String signOffUser;
        private String caseId;
        private String artifact;
        private String rejectionReason;
        private String status;
    }

    public void fillFieldMapping(ObjectNode formData, Map<String, String> fieldMapping, Map<String, FieldInfo> formDataMap) {
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
    }

    public void fillIchampValue(ObjectNode formData, FieldInfo fieldInfo, String ichampFieldCode) {
        //codechecker error

        FieldValueTypeEnum fieldType = fieldInfo.getFieldType();
        switch (fieldType) {
            case NUMBER:
                NumberField numberField = (NumberField) fieldInfo.getFieldValueObj();
                String numberFieldIntValue = numberField.getValue();
                formData.put(ichampFieldCode, StrUtil.isBlank(numberFieldIntValue) ? "" : numberFieldIntValue);
                break;
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
                formData.put(ichampFieldCode, dateField.getValue() == null ? "" : DateUtil.format(new Date(dateField.getValue()), "yyyy-MM-dd HH:mm:ss"));
                break;
            case TIME:
                TimeField timeField = (TimeField) fieldInfo.getFieldValueObj();
                formData.put(ichampFieldCode, timeField.getValue() == null ? "" : DateUtil.format(new Date(timeField.getValue()), "yyyy-MM-dd HH:mm:ss"));
                break;
        }
    }

    private void setRiskValue(ObjectNode formData, Map<String, FieldInfo> formDataMap, String itsmFieldCode, String lowDictId, String ichampFieldCode) {
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
    public List<String> getUserOneBankIdsByUser(List<UserInfo> userList) {
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
    public List<UserInfo> getUsers(List<Long> userIdList) {
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
    public static class ApproveInfo {

        private String nodeId;
        private String rejectionCode;
        private String reasonCode;
        private String statusCode;

    }


    @Data
    public static class TableFieldMapping {
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
            String body = HttpRequest.post(url).header("Content-Type", "application/json").body(com.cloudwise.dosm.core.utils.JsonUtils.toJsonString(sendMsgContextMap)).timeout(5000).execute().body();
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
        publicFieldsMap.put("createdByEmail", createdUserInfo == null ? "" : createdUserInfo.getEmail());
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


package com.cloudwise.dosm.dbs.trigger;

import lombok.Data;

import java.util.List;
import java.util.Map;

@Data
public class ExtParam {
    private String pushUrl;
    private String updateUrl;
    private String idrUrl;
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
    private Map<String, List<CrStatusSyncTrigger.TableFieldMapping>> tableFieldMapping;
    private List<CrStatusSyncTrigger.ApproveInfo> approveInfoMapping;
    private Map<String, String> rcaFieldMapping;
}




package com.cloudwise.dosm.dbs.trigger;


public interface SignoffConstants {


    enum SignoffType {
        TESTING_SIGNOFF("TestingSignoff"),
        PROCUTOVER_SIGNOFF("ProCutoverSignoff"),
        OTHER_SIGNOFFS("OtherSignoffs"),
        OPTIONAL_ARTEFACTS("OptionalArtefacts"),
        HEIGHTENED_SIGNOFF("HeightenedSignoff");

        private final String type;

        SignoffType(String type) {
            this.type = type;
        }

        public String getType() {
            return type;
        }

        public static SignoffType fromString(String text) {
            for (SignoffType signoffType : SignoffType.values()) {
                if (signoffType.getType().equalsIgnoreCase(text)) {
                    return signoffType;
                }
            }
            throw new IllegalArgumentException("No constant with text " + text + " found");
        }
    }


    enum SignoffItem {
//        USER_ACCEPTANCE_TESTING("UAT", , , , ),
//        REGRESSION_TESTING("Regression Testing", , , , ),
//        REVERSION_BACKOUT_ROLLBACK_TESTING("Rollback Testing", , , , ),
//        PERFORMANCE_TESTING("Performance Testing", , , , ),
//        PRODUCTION_ASSURANCE_TESTING("PAT", , , , ),
//        CHAOS_TESTING("Chaos Testing", , , , ),


        HA_DR_FLIP_SIGNOFF("HA & DR Flip Signoff", "hasignoffstatus","hasignoffapproverlogin" , "hasignoffrejectionreason", null),
        MD_DELEGATE_SIGNOFF("MD Delegate Signoff","mddelegatesignoffstatus", "mddelegatesignoffapproverlogin","mddelegatesignoffrejectionreason" , null),
        BU_APPLICATION_OWNER_SIGNOFF("BU/Application Owner Signoff", "busignoffstatus",null ,"busignoffrejectionreason" ,null ),
        DATA_CENTER_OPS_BATCH_SIGNOFF("Data Center OPS (Batch) Signoff","datacentersignoffstatus" ,"datacentersignoffapproverlogin" , "datacentersignoffrejectionreason",null ),
        IMPACT_TO_MAINFRAME_SIGNOFF("Impact To Mainframe Signoff", "impacttomainframestatus", "impacttomainframeapproverlogin", "impacttomainframerejectionreason",null ),
        DESIGN_FOR_DATA_D4D_SIGNOFF("Design For Data (D4D) Signoff","d4dsignoffstatus" , "d4dsignoffapproverlogin", "d4dsignoffrejectionreason", null),
        /**
         * "servicemonitoringsnoctype" : "Approval Required",
         * 		"servicemonitoringsnocappservicelist" : "text",
         * 		"servicemonitoringsnocurllist" : "text",
         */
//        SERVICE_MONITORING_AT_SNOC("Service Monitoring at SNOC",null ,null , null,null ),
        CUS_SIGNOFF("CUS Signoff",null ,null , null, "cussignoffurl"),
        IDR_SIGNOFF_PROCUTOVER("IDR Signoff", null,null ,null , "idrsignoffurl");

//        ISS_SIGNOFF("ISS Signoff", , , , ),
//        CODE_CHECKER_SIGNOFF("Code Checker Signoff", , , , ),
//        DR_TEAM_SIGNOFF("DR team Signoff", , , , ),
//        STORAGE_TEAM_SIGNOFF("Storage team Signoff", , , , ),
//        DCON_SIGNOFF("DCON Signoff", , , , ),
//        TECHNICAL_LIVE_VERIFICATIONLV_SIGNOFF("Technical Live Verification (LV) Signoff", , , , ),
//        BUSINESS_LIVE_VERIFICATIONLV_SIGNOFF("Business Live Verification (LV) Signoff", , , , ),
//        IMPLEMENTATION_CHECKER_SIGNOFF("Implementation Checker Signoff", , , , ),


//        HEIGHTENED_HEIGHTENED_PERIOD_RELATED_ARTEFACTS("Heightened/Heightened Period Related Artefacts", , , , ),
//        COMPLIANCE_RELATED_ARTEFACTS("Compliance Related Artefacts", , , , ),
//        UNAUTHORIZED_CHANGE_REPORT_DOCS("Unauthorized Change Report/ Docs", , , , ),
//
//        ARC_SIGNOFF("ARC Signoff", , , , ),
//        LOBT_CHANGE_MANAGERS_CONCURRENCE("LOBT Change managers' concurrence", , , , ),
//        CTO_S_APPROVALS_INFRA_AND_APP("CTO's approvals (infra and app)", , , , );

        private final String name;

        private final String signOffStatusCode;
        private final String signOffApproverLoginCode;
        private final String signOffRejectionReasonCode;
        private final String signOffUrlCode;



        SignoffItem(String name, String signOffStatusCode, String signOffApproverLoginCode, String signOffRejectionReasonCode, String signOffUrlCode) {
            this.name = name;
            this.signOffStatusCode = signOffStatusCode;
            this.signOffApproverLoginCode = signOffApproverLoginCode;
            this.signOffRejectionReasonCode = signOffRejectionReasonCode;
            this.signOffUrlCode = signOffUrlCode;
        }

        public String getName() {
            return name;
        }

        public String getSignOffApproverLoginCode() {
            return signOffApproverLoginCode;
        }

        public String getSignOffRejectionReasonCode() {
            return signOffRejectionReasonCode;
        }

        public String getSignOffStatusCode() {
            return signOffStatusCode;
        }

        public String getSignOffUrlCode() {
            return signOffUrlCode;
        }

        public static SignoffItem fromString(String signOffType) {
            for (SignoffItem signoffItem : SignoffItem.values()) {
                if (signoffItem.getName().equalsIgnoreCase(signOffType)) {
                    return signoffItem;
                }
            }
            return null;
        }


    }
}


package com.cloudwise.dosm.dbs.trigger;

/**
 * @author abell.wu
 */

public enum SignOffStatusEnum {


    WAITSEND("WAITSEND", "Waiting for Send"),

    PENDING("PENDING", "Pending Approval"),


    APPROVED("APPROVED", "Approved"),


    REJECTED("REJECTED", "Rejected"),


    ;


    private String code;


    private String desc;


    SignOffStatusEnum(String code, String desc) {
        this.code = code;
        this.desc = desc;
    }

    public String getDesc() {
        return desc;
    }
}
