package com.cloudwise.dosm.dbs.trigger;

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
import com.cloudwise.dosm.biz.instance.service.mdlins.MdlInstanceAddService;
import com.cloudwise.dosm.biz.instance.service.mdlins.MdlInstanceService;
import com.cloudwise.dosm.bpm.api.instance.MdlInstanceApiService;
import com.cloudwise.dosm.core.utils.SpringContextUtils;
import com.cloudwise.dosm.douc.entity.user.UserInfo;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.List;
import java.util.Map;

@Slf4j
@ExtendConfig(id = "rca-status-sync-trigger", name = "RCA Status Sync to IChamp", desc = "RCA Status Sync to IChamp")
public class RcaStatusSyncTrigger implements ITriggerExtV2 {
    @Autowired
    MdlInstanceApiService mdlInstanceApiService;
    @Autowired
    MdlInstanceMapper mdlInstanceMapper;

    @Override
    public void handler(TriggerContext ctx) {
        WorkOrderBean rcaWorkOrderInfo = ctx.getWorkOrder();
        Map<String, FieldInfo> rcaFormData = rcaWorkOrderInfo.getFormDataMap();
        FieldInfo crTicketNumber = rcaFormData.get("CRTicketNumber");
        FieldValue fieldValueObj = crTicketNumber.getFieldValueObj();
        MdlInstance mdlInstance = mdlInstanceMapper.selectOne(Wrappers.lambdaQuery(MdlInstance.class).eq(MdlInstance::getBizKey, fieldValueObj.getValue()), false);
        if (mdlInstance == null) {
            return;
        }
        CrStatusSyncTrigger.ExtParam param = JsonUtils.parseObject(ctx.getExtParam(), CrStatusSyncTrigger.ExtParam.class);
        WorkOrderBean workOrder = mdlInstanceApiService.getAuthorizeWorkOrder(ctx.getTopAccountId(), ctx.getAccountId(), ctx.getUserId(), mdlInstance.getProcessInstanceId());
        CrStatusSyncTrigger crStatusSyncTrigger = SpringContextUtils.getBean(CrStatusSyncTrigger.class);
        List<UserInfo> createdBys = crStatusSyncTrigger.getUsers(Lists.newArrayList(Long.valueOf(workOrder.getCreatedBy())));
        Map<String, FieldInfo> formDataMap = workOrder.getFormDataMap();
        FieldInfo fieldInfo = formDataMap.get(param.getCloseStatusFieldCode());
        SelectField crStatusValue = (SelectField) fieldInfo.getFieldValueObj();
        String appCode = crStatusSyncTrigger.getConfig("dosm.custom.appCode");
        String appKey = crStatusSyncTrigger.getConfig("dosm.custom.appKey");
        JsonNode params = crStatusSyncTrigger.makeParam(param, workOrder, createdBys, CrStatusEnums.getByLabel(crStatusValue.getLabel()), false);
        ObjectNode formData = (ObjectNode) params.get("formData");
        crStatusSyncTrigger.fillFieldMapping(formData, param.getRcaFieldMapping(), rcaFormData);
        log.info("RcaStatusSyncTriggermakeParam:{}", params);
        HttpResponse response = null;
        try {
            response = HttpUtil.createPost(param.getUpdateUrl()).header("Content-Type", "application/json")
                    .header("appCode", appCode).header("appKey", appKey).body(JsonUtils.toJsonString(params)).execute();
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
            log.error("RcaStatusSyncTrigger send to ichamp error:{}", e.getMessage(), e);
            ResultBean resultBean = ctx.getResult();
            resultBean.setError("Sync data to ichamp fail: data:" + e.getMessage());
            ctx.setResult(resultBean);
        }

    }
}
