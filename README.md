package com.cloudwise.dosm.dbs.customBtn;

import cn.hutool.core.util.IdUtil;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;
import cn.hutool.json.JSONUtil;
import com.cloudwise.dosm.api.adv.extend.CustomBtnCurrentWorkInfo;
import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.ICustomBtnExt;
import com.cloudwise.dosm.api.bean.trigger.entity.ResultBean;
import com.cloudwise.dosm.biz.instance.dao.MdlInstanceMapper;
import com.cloudwise.dosm.biz.instance.entity.MdlInstance;
import com.cloudwise.dosm.core.exception.BaseException;
import com.cloudwise.dosm.dict.dao.DataDictDetailMapper;
import com.cloudwise.dosm.dict.dao.DataDictMapper;
import com.cloudwise.dosm.dict.entity.DataDict;
import com.cloudwise.dosm.dict.entity.DataDictDetail;
import com.cloudwise.dosm.douc.entity.user.UserInfo;
import com.cloudwise.dosm.facewall.core.utils.JsonUtils;
import com.cloudwise.dosm.facewall.extension.base.startup.util.SpringContextUtils;
import com.cloudwise.dosm.facewall.extension.core.annotation.Action;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.env.Environment;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.PreparedStatementCallback;

import java.util.*;

@Slf4j
@ExtendConfig(id = "dbs_cr_status_reopen_btn_seven", name = "reopen btn", desc = "reopen btn")
public class DbsCrStatusReopenBtnSeven implements ICustomBtnExt {
    private static final String CSH = "custom.service.host";
    private static final String CSHPRE = "custom.service.host.prefix";

    @Override
    @Action(value = "dbs_cr_status_reopen_btn_seven", label = "Reopen", description = "Reopen")
    public Map<String, Object> handler(Map<String, Object> requestParamMap) {
        log.info("cr_status_reopen requestParamMap:{}", requestParamMap);
        Map<String, Object> res = new HashMap<>();
        CustomBtnCurrentWorkInfo currentOrder = getCurrentOrder(requestParamMap);

        // 获取参数
        String fieldCode = "crStatus";
        String crStatusValue = "Reopen";
        String reopenDictCode = "crStatus";
        String workOrderId = currentOrder.getWorkOrderId();

//        // Find the "CR Status" node
//        JsonNode crStatusNode = findCrStatusNode(formInfo,fieldCode);
//        if (Objects.isNull(crStatusNode)) {
//            throw new BaseException("Cannot find the cr status key");
//        }
//        String directoryValue = crStatusNode.path("x-props").path("directoryValue").asText();
//        log.error("cr_status_reopen workorderId:{},directoryValue:{}",workOrderId,JsonUtils.toJsonString(directoryValue));


        // 更新工单crStatus
        MdlInstanceMapper mdlInstanceMapper = SpringContextUtils.getBean(MdlInstanceMapper.class);
        MdlInstance mdlInstance = mdlInstanceMapper.selectById(workOrderId, null);
        log.error("cr_status_reopen workorderId:{},mdlInstance:{}", workOrderId, JsonUtils.toJsonString(mdlInstance));
        if (Objects.isNull(mdlInstance)) {
            throw new BaseException("Cannot find the current work order");
        }

        Object data = mdlInstance.getFormData();
        ObjectNode formData = (ObjectNode) com.cloudwise.dosm.api.bean.utils.JsonUtils.parseJsonNode(data);

        List<DataDictDetail> dataDictDetails = this.getDictOption(reopenDictCode);
        log.error("cr_status_reopen dataDictDetails:{}", JSONUtil.toJsonStr(dataDictDetails));
        Optional<DataDictDetail> first = dataDictDetails.stream().filter(item -> item.getData().equals(crStatusValue)).findFirst();
        if (!first.isPresent()) {
            throw new BaseException("Cannot find the reopen option");
        }
        formData.put(fieldCode, first.get().getId());
        formData.put(fieldCode + "_value", first.get().getLabel());
        mdlInstance.setFormData(formData);
        mdlInstanceMapper.updateByIdSelective(mdlInstance);
        handleSyncIChamp(mdlInstance);
        return res;
    }

    public String getConfig(String configKey, String defaultValue) {
        Environment bean = com.cloudwise.dosm.core.utils.SpringContextUtils.getBean(Environment.class);
        return bean.getProperty(configKey, defaultValue);
    }

    public void handleSyncIChamp(MdlInstance mdlInstance) {
        Map<String, Object> datas = new HashMap<>();
        datas.put("workOrder", mdlInstance);
        datas.put("originMessage", "");
        String syncUrl = getConfig(CSH, "");
        String prefix = getConfig(CSHPRE, "http://");
        String workOrderId = mdlInstance.getId();
        String finalUrl = prefix + syncUrl + "dcs/api/ichamp/sync?workOrderId=" + workOrderId;
        try {
            HttpResponse execute = HttpUtil.createPost(finalUrl).execute();
            String body = execute.body();
            log.info("body:{}", body);
        } catch (Exception e) {
            log.error("CrStatusSyncTrigger send to ichamp error:{}", e.getMessage(), e);
            saveApiLog(finalUrl, "", e.getMessage(), mdlInstance.getBizKey(), "crStatusSync", false);
        }
    }

    public void saveApiLog(String requestUrl, String requestBody, String responseBody, String workOrderNo, String apiModule, boolean requestStatus) {
        JdbcTemplate jdbcTemplate = SpringContextUtils.getBean(JdbcTemplate.class);
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


    public List<DataDictDetail> getDictOption(String dictCode) {
        DataDictDetailMapper dataDictDetailMapper = SpringContextUtils.getBean(DataDictDetailMapper.class);
        DataDictMapper dataDictMapper = SpringContextUtils.getBean(DataDictMapper.class);
        DataDict dataDict = new DataDict();
        dataDict.setIsDel(0);
        dataDict.setDictCode(dictCode);
        DataDict crStatusDataDict = dataDictMapper.selectOneByParam(dataDict);
        List<DataDictDetail> dataDictDetails = dataDictDetailMapper.selectDetailByDictIdAndLevel(crStatusDataDict.getId(), 1, crStatusDataDict.getAccountId());
        return dataDictDetails;
    }

//    /**
//     * 获取formInfo 中的 fieldCode 的规则
//     */
//    public static JsonNode findCrStatusNode(JsonNode node,String fieldCode) {
//        if (node.isObject()) {
//            // Check if the current node is the "cr_status" node
//            if (fieldCode.equals(node.path("key").asText())) {
//                return node;
//            }
//
//            // Recursively search in the properties of the current node
//            JsonNode propertiesNode = node.path("properties");
//            if (propertiesNode.isObject()) {
//                for (JsonNode propertyNode : propertiesNode) {
//                    JsonNode result = findCrStatusNode(propertyNode,fieldCode);
//                    if (result != null) {
//                        return result;
//                    }
//                }
//            }
//        }
//        return null;
//    }

}





package com.cloudwise.dosm.dbs.trigger;

import cn.hutool.core.util.IdUtil;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.v2.ITriggerExtV2;
import com.cloudwise.dosm.api.bean.form.FieldInfo;
import com.cloudwise.dosm.api.bean.form.field.FieldValue;
import com.cloudwise.dosm.api.bean.form.field.SelectField;
import com.cloudwise.dosm.api.bean.instance.entity.WorkOrderBean;
import com.cloudwise.dosm.api.bean.trigger.entity.ResultBean;
import com.cloudwise.dosm.api.bean.trigger.entity.TriggerContext;
import com.cloudwise.dosm.api.bean.utils.JsonUtils;
import com.cloudwise.dosm.biz.instance.dao.MdlInstanceMapper;
import com.cloudwise.dosm.biz.instance.entity.MdlInstance;
import com.cloudwise.dosm.bpm.api.instance.MdlInstanceApiService;
import com.cloudwise.dosm.core.utils.SpringContextUtils;
import com.cloudwise.dosm.douc.entity.user.UserInfo;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.env.Environment;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.PreparedStatementCallback;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Slf4j
@ExtendConfig(id = "rca-status-sync-trigger", name = "RCA Status Sync to IChamp", desc = "RCA Status Sync to IChamp")
public class RcaStatusSyncTrigger implements ITriggerExtV2 {
    private static final String CSH = "custom.service.host";
    private static final String CSHPRE = "custom.service.host.prefix";
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public void handler(TriggerContext ctx) {
        callSyncIChampApi(ctx);
    }

    private void callSyncIChampApi(TriggerContext ctx) {
        WorkOrderBean workOrder = ctx.getWorkOrder();
        Map<String, Object> datas = new HashMap<>();
        datas.put("workOrder", workOrder);
        datas.put("originMessage", ctx.getOriginMessage());
        String syncUrl = getConfig(CSH, "");
        String prefix = getConfig(CSHPRE, "http://");
        String workOrderId = workOrder.getId();
        String finalUrl = prefix + syncUrl + "dcs/api/ichamp/sync?rcaWorkOrderId=" + workOrderId;
        try {
            HttpResponse execute = HttpUtil.createPost(finalUrl).execute();
            String body = execute.body();
            log.info("body:{}", body);
        } catch (Exception e) {
            log.error("CrStatusSyncTrigger send to ichamp error:{}", e.getMessage(), e);
            ResultBean resultBean = ctx.getResult();
            resultBean.setError("Sync data to ichamp fail: data:" + e.getMessage());
            ctx.setResult(resultBean);
            saveApiLog(finalUrl, "", e.getMessage(), workOrder.getBizKey(), "crStatusSync", false);
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

    public static String getConfig(String configKey, String defaultValue) {
        Environment bean = SpringContextUtils.getBean(Environment.class);
        return bean.getProperty(configKey, defaultValue);
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
import com.cloudwise.dosm.api.bean.form.field.NullFieldValue;
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
        JsonNode nodeIdNode = originMessageNode.get("nodeId");
        String nodeId = nodeIdNode.asText();
        JsonNode eventTypeNode = originMessageNode.get("eventType");
        String eventType = eventTypeNode.asText();
        if (nodeId.equals(param.getCreateNodeId()) && EventTypeEnum.NODE_EXIT.getCode().equals(eventType)) {
            return;
        }

        Map<String, Object> datas = new HashMap<>();
        datas.put("workOrder", workOrder);
        datas.put("originMessage", originMessageNode);
        String syncUrl = getConfig(CSH, "");
        String prefix = getConfig(CSHPRE, "http://");
        String workOrderId = workOrder.getId();
        String finalUrl = prefix + syncUrl + "dcs/api/ichamp/sync?workOrderId=" + workOrderId;
        try {
            HttpResponse execute = HttpUtil.createPost(finalUrl).execute();
            String body = execute.body();
            log.info("body:{}", body);
            saveApiLog(finalUrl, "", body, workOrder.getBizKey(), "crStatusSync", false);
            List<UserInfo> createdBys = getUsers(Lists.newArrayList(Long.valueOf(workOrder.getCreatedBy())));
            sendApprovalMsg(crStatus, ctx, createdBys);
        } catch (Exception e) {
            log.error("CrStatusSyncTrigger send to ichamp error:{}", e.getMessage(), e);
            ResultBean resultBean = ctx.getResult();
            resultBean.setError("Sync data to ichamp fail: data:" + e.getMessage());
            ctx.setResult(resultBean);
            saveApiLog(finalUrl, "", e.getMessage(), workOrder.getBizKey(), "crStatusSync", false);
        }
    }

    public String getConfig(String configKey) {
        Environment bean = com.cloudwise.dosm.core.utils.SpringContextUtils.getBean(Environment.class);
        return bean.getProperty(configKey);
    }

    public String getConfig(String configKey, String defaultValue) {
        Environment bean = com.cloudwise.dosm.core.utils.SpringContextUtils.getBean(Environment.class);
        return bean.getProperty(configKey, defaultValue);
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
        } else if ("NODE_EXIT".equals(eventType.asText()) && nodeId.equals(extParam.getSignoffNodeId())) {
            return CrStatusEnums.OPEN;
        } else if ("NODE_ENTER".equals(eventType.asText()) && (nodeId.equals(extParam.getImplementationNodeId()) || nodeId.equals(extParam.getCloseNodeId()))) {
            return CrStatusEnums.APPROVED;
        } else if ("WORK_GET_BACK".equals(eventType.asText()) && rollbackToNodeId != null && rollbackToNodeId.asText().equals(extParam.getSignoffNodeId())) {
            return CrStatusEnums.CLOSED_CANCEL;
        } else if ("WORK_ROLLBACK".equals(eventType.asText()) && rollbackToNodeId != null && rollbackToNodeId.asText().equals(extParam.getSignoffNodeId())) {
            if ("2".equals(ctx.getUserId())) {
                return CrStatusEnums.CLOSED_CANCEL;
            }
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
            log.info("Send email success:{}", body);

            ((Map<String, Object>) sendMsgContextMap.get("notify")).put("channelType", "SMS");
            body = HttpRequest.post(url).header("Content-Type", "application/json").body(com.cloudwise.dosm.core.utils.JsonUtils.toJsonString(sendMsgContextMap)).timeout(5000).execute().body();
            log.info("Send SMS success:{}", body);
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

