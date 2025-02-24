package com.cloudwise.dosm.field;

import cn.hutool.core.date.DateUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.v2.IFieldLinkExtV2;
import com.cloudwise.dosm.api.bean.form.FieldInfo;
import com.cloudwise.dosm.api.bean.form.FieldTriggerRule;
import com.cloudwise.dosm.api.bean.form.field.FieldValue;
import com.cloudwise.dosm.api.bean.form.field.SelectField;
import com.cloudwise.dosm.api.bean.form.field.SelectManyField;
import com.cloudwise.dosm.api.bean.instance.entity.FieldLinkContext;
import com.cloudwise.dosm.constant.CommonFieldEnums;
import com.cloudwise.dosm.core.utils.JsonUtils;
import com.cloudwise.dosm.model.FieldInfoEnum;
import com.cloudwise.dosm.util.CommonUtil;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;

import javax.annotation.Resource;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Rule1: NA is allowed.
 * Rule2: Multi values are allowed in the ‘CyberArk Objects’ field and multi values must be separated by ‘,’.
 * Rule3: All the CyberArk Objects must belong to applications (The range of applications are chosen in ‘Application impacted’). If not, the warning message will be shown as: 'The following CyberArk objects are Invalid for this CR: [‘objectname1’,’objectname2’]'.
 * Rule4: When a requester submits CR for approval, the status of the CyberArk Objects needs to be verified, which means CyberArk Objects in the ‘Pre-Production’ status cannot be included in the CR, except for the following circumstances:
 * LOBs = CES & Change Group when it is any of the following: Project, Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business, BAU Cutover. CR can contain Objects in the ‘Pre-Production’ state.
 * LOBs = CES & Change Group when it is NOT any of the following: Project, Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business, BAU Cutover. CR cannot contain Objects in the ‘Pre-Production’ state.
 * LOBs != CES& Change Group is any of the following: Project and Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business. CR can contain Objects in the ‘Pre-Production’ state
 * LOBs != CES& Change Group is NOT any of the following: Project and Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business. CR cannot contain Objects in the ‘Pre-Production’ state.
 */
@Slf4j
@ExtendConfig(id = "CyberObjectsFieldLink", name = "CyberObjectsFieldLink", desc = "CyberObjectsFieldLink")
public class CyberObjectsFieldLink implements IFieldLinkExtV2 {

    String descDefault = "<div style=\"color: red;\"> Important: Please ensure only required CyberArk Objects are requested. Requesting Unit is accountable for all CyberArk Objects requested in CR.</div>";
    String canContainDesc = "<div style=\"color: black;\"> CR can contain Objects in the ‘Pre-Production’ state. </div>";
    @Override
    public void handler(FieldLinkContext ctx) {
        Long time = System.currentTimeMillis();
        log.error("CyberObjectsFieldLink:time:{},ctx:{}", time, JSONUtil.toJsonStr(ctx));
        Map<String, FieldInfo> formData = ctx.getFormData();
        Map<String, FieldTriggerRule> resultMap = ctx.getResultMap();
        String extParam = ctx.getExtParam();
        /**
         * 1)Cyberark objects are pre-prod status
         * 2) Change group != Project and Project Cutover Technical or Project Cutover Business or Project Cutover Technical & Business
         * 3) LOBs !=CES
         * 4) Change Type != ECR,
         * CR can contain Objects in the ‘Pre-Production’ state.
         */

        FieldInfo changeGroup = formData.get("changeGroup");
        FieldInfo lob = formData.get("lob");
//        FieldInfo changeType = formData.get("changeType");
        boolean isEqCes = false;
        if(lob !=null && lob.getFieldValueObj()!=null && lob.getFieldValueObj().getValue()!=null){
            SelectField lobFieldValueObj = (SelectField) lob.getFieldValueObj();
            isEqCes =  lobFieldValueObj.getLabel().equals("CES");
        }
        boolean isProjectCutover = false;
        boolean isBauCutover = false;
        if(changeGroup!=null && changeGroup.getFieldValueObj()!=null && changeGroup.getFieldValueObj().getValue()!=null){
            if(changeGroup.getFieldValueObj() instanceof SelectField) {
                SelectField changeGroupFieldValueObj = (SelectField) changeGroup.getFieldValueObj();
                isProjectCutover = changeGroupFieldValueObj.getLabel().startsWith("Project");
                isBauCutover = changeGroupFieldValueObj.getLabel().equals("BAU Cutover");
            }
        }
        String finalNodeMessage = descDefault;
        if(isEqCes && (isBauCutover|| isProjectCutover )){
            finalNodeMessage += canContainDesc;

        }
        if (!isEqCes && isProjectCutover){
            finalNodeMessage += canContainDesc;
        }
        CommonUtil.setNeedSetNodeRuleByMessage(resultMap, null, FieldInfoEnum.CyberArk_Object.name(), finalNodeMessage);

        FieldInfo fieldInfo = formData.get(CommonFieldEnums.CYBERARK_OBJECTS.getFieldCode());
        Object value = Objects.nonNull(fieldInfo) ? fieldInfo.getFieldValueObj().getValue() : null;
        log.error("CyberObjectsFieldLink:time:{},value:{}", time, value);
        if (Objects.isNull(value) || "NA".equals(String.valueOf(value).toUpperCase())) {
            CommonUtil.setResultMapRuleNoMessage(resultMap, null, FieldInfoEnum.CyberArk_Object.name(), null);
            return;
        }
        FieldInfo applicationImpactedFieldInfo = formData.get(CommonFieldEnums.APPLICATION_IMPACTED.getFieldCode());
        SelectManyField selectManyField = Objects.nonNull(applicationImpactedFieldInfo) ? (SelectManyField) applicationImpactedFieldInfo.getFieldValueObj() : null;
        Collection<String> labelList = Objects.nonNull(selectManyField) ? selectManyField.getLabel() : new ArrayList<>();
        StringBuilder resultAlertInfo = new StringBuilder();
        if(labelList.size() == 0){
            resultAlertInfo.append("\nThe following CyberArk objects are Invalid for this CR:[" + value +"].");
        }else {
            List<String> notInAppCodeCyberArk = this.getNotInAppCodeCyberArk(value.toString(), labelList);
            log.error("CyberObjectsFieldLink:time:{},noIncyber:{}", time, notInAppCodeCyberArk);
            if (notInAppCodeCyberArk.size() > 0) {
                resultAlertInfo.append("\nThe following CyberArk objects are Invalid for this CR:" + JsonUtils.toJsonString(notInAppCodeCyberArk) + ".");
            }
        }
        String result = resultAlertInfo.toString();
        if (StrUtil.length(result) > 0 ){
            CommonUtil.setResultMapRuleByMessage(resultMap, null, FieldInfoEnum.CyberArk_Object.name(), result);
        }else{
            CommonUtil.setResultMapRuleNoMessage(resultMap, null, FieldInfoEnum.CyberArk_Object.name(), null);
        }
    }

    @Resource
    private JdbcTemplate jdbcTemplate;

    /**
     * @param
     * @return
     */
    private List<String> getNotInAppCodeCyberArk(String values, Collection<String> appCodes) {
        if(CollectionUtils.isEmpty(appCodes)){
            return new ArrayList<>();
        }
        String[] split = values.split(",");
        List<String> targetlist = new ArrayList<>();
        String appCodeStr =  appCodes.stream().collect(Collectors.joining("','"));
        if (split != null) {
            Arrays.stream(split).forEach(one -> {
                String sql = "select application_code from cyberark_info where password_object = '" + one + "' and application_code in ('"+appCodeStr+"')";
                log.error("sql:{}", sql);
                Map<String, String> args = new HashMap<>();
                NamedParameterJdbcTemplate query = new NamedParameterJdbcTemplate(jdbcTemplate);
                List<String> list = query.query(sql, args, new RowMapper() {
                    @Override
                    public String mapRow(ResultSet rs, int rowNum) throws SQLException {
                        String appCode = rs.getString("application_code");
                        return appCode;
                    }
                });
                log.error("CyberObjectsFieldLink:time:{},value:{}", DateUtil.formatDateTime(new Date()), list);
                if(list.isEmpty()) {
                    targetlist.add(one);
                }
            });
        }
        return targetlist;

    }
}
