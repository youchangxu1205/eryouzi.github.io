package com.cloudwise.dosm.button;

import com.cloudwise.dosm.api.adv.extend.ExtendConfig;
import com.cloudwise.dosm.api.adv.extend.IButtonActionExt;
import com.cloudwise.dosm.api.bean.adv.extend.BtnActionBo;
import com.cloudwise.dosm.api.bean.adv.extend.BtnCallResultBo;
import com.cloudwise.dosm.api.bean.enums.BtnTypeEnum;
import com.cloudwise.dosm.api.bean.enums.ResultCodeEnum;
import com.cloudwise.dosm.api.bean.utils.JsonUtils;
import com.cloudwise.dosm.biz.instance.dao.MdlInstanceMapper;
import com.cloudwise.dosm.biz.instance.entity.MdlInstance;
import com.cloudwise.dosm.core.pojo.bo.RequestDomain;
import com.cloudwise.dosm.core.utils.UserHolder;
import com.cloudwise.dosm.dict.dao.DataDictDetailMapper;
import com.cloudwise.dosm.dict.dao.DataDictMapper;
import com.cloudwise.dosm.dict.entity.DataDict;
import com.cloudwise.dosm.dict.entity.DataDictDetail;
import com.cloudwise.dosm.douc.entity.user.UserGroupInfo;
import com.cloudwise.dosm.douc.service.UserService;
import com.cloudwise.dosm.facewall.extension.base.startup.util.SpringContextUtils;
import com.cloudwise.douc.dto.DubboCommonResp;
import com.cloudwise.douc.dto.DubboUserIdAccountIdRequest;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.Optional;

/**
 * @ClassName ClosedCancelBannerButton
 * @Description 注释
 * @Author Gray Li
 * @Date 2025/2/8 2:26 PM
 * @Version 1.0
 */
@Slf4j
@ExtendConfig(id = "ClosedCancelBannerButton", name = "ClosedCancelBannerButton", desc = "commitButton")
public class ClosedCancelBannerButton implements IButtonActionExt {

    private final static String Select_CR_Status="crStatus";

    private static final String Select_MainframeCRType="mainframeCRType";

    private static final String Select_ChangeType="changeType";

    private static final String Group_ChangeRequestorGroups="changeRequestorGroups";
    @Autowired
    private MdlInstanceMapper mdlInstanceMapper;

    @Override
    public List<BtnTypeEnum> getActionList() {
        List<BtnTypeEnum> button = new ArrayList<>();
        button.add(BtnTypeEnum.GETBACK);
        return button;
    }

    @Override
    public void before(BtnActionBo btnActionBo, BtnCallResultBo resultBo) {

        String bizKey = btnActionBo.getBizKey();

        resultBo.setSkipSysExecutor(false);
        resultBo.setCode(ResultCodeEnum.SUCCESS);

        // Only the users of L1.5 Change Team can closed cancel such CRs (Except ECR):
        if (!bizKey.startsWith("ECR")) {

            BtnCallResultBo btnCallResultBo =  handleCrAction(btnActionBo);

            if (btnCallResultBo.getCode().equals(ResultCodeEnum.FAILURE)) {
                resultBo.setSkipSysExecutor(true);
                resultBo.setCode(btnCallResultBo.getCode());
                resultBo.setMsg(btnCallResultBo.getMsg());
                return;
            }
        }
    }


    @Override
    public void after(BtnActionBo btnActionBo, Object sysResultData, BtnCallResultBo resultBo) {

    }

    @Override
    public void rollbackBefore(BtnActionBo btnActionBo, Object sysResultData, Object rollbackParam) {

    }

    private BtnCallResultBo handleCrAction(BtnActionBo btnActionBo){
        BtnCallResultBo resultBo = new BtnCallResultBo();
        resultBo.setSkipSysExecutor(true);
        resultBo.setCode(ResultCodeEnum.FAILURE);
        resultBo.setMsg("Once the CR is Approved, only user(s) from L1.5 Change Team can 'Closed Cancel' CRs related to Mainframe Package Deployment. Please contact them (dbschg@dbs.com) for cancelling CR.");
        JsonNode formData = null;
        MdlInstance mdlInstance = mdlInstanceMapper.selectByWorkOrderId(btnActionBo.getTopAccountId(), btnActionBo.getWorkOrderId());
        if(btnActionBo.getFormData() == null){
            formData = JsonUtils.parseJsonNode(mdlInstance.getFormData());
        }else {
            formData = JsonUtils.parseJsonNode(btnActionBo.getFormData());
        }


        RequestDomain requestDomain = UserHolder.get();

        UserService userService = SpringContextUtils.getBean(UserService.class);

        boolean status=false;

        //L1.5 Change Team
        DubboUserIdAccountIdRequest req2 = new DubboUserIdAccountIdRequest();
        req2.setUserId(Long.valueOf(requestDomain.getUserId()));
        req2.setAccountId(Long.valueOf(requestDomain.getAccountId()));
        req2.setAccountScope("all");
        req2.setGroupScope("all");
        DubboCommonResp<List<UserGroupInfo>> userGroupInfosByUserId = userService.getUserGroupInfosByUserId(req2);
        Long loginUserGroupId=null;
        if (Objects.nonNull(userGroupInfosByUserId) && Objects.nonNull(userGroupInfosByUserId.getData())) {
            List<UserGroupInfo> userGroupInfoList = userGroupInfosByUserId.getData();
            if (userGroupInfoList != null && userGroupInfoList.size() > 0) {
                //Is Member of CR Requestor Group
                for (int i = 0; i < userGroupInfoList.size(); i++) {
                    UserGroupInfo item = userGroupInfoList.get(i);
                    loginUserGroupId= item.getGroupId();
                    if ("L1.5 Change Management Group".equals(item.getGroupName())) {
                        status = true;
                        break;
                    }
                }
            }
        }

        if(status){
            resultBo.setSkipSysExecutor(false);
            resultBo.setCode(ResultCodeEnum.SUCCESS);
            handleStatus2CloseCancel(mdlInstance);
            return resultBo;
        }

        String crStatus=null,mainframeCRType=null,changeType=null, groupId=null;
        if(formData.has(Select_CR_Status+"_value") && formData.get(Select_CR_Status+"_value") !=null){
            crStatus = formData.get(Select_CR_Status+"_value").asText("");
        }

        if(formData.has(Select_MainframeCRType+"_value") && formData.get(Select_MainframeCRType+"_value") !=null){
            mainframeCRType = formData.get(Select_MainframeCRType+"_value").asText("");
        }

        if(formData.has(Select_ChangeType+"_value") && formData.get(Select_ChangeType+"_value") !=null){
            changeType = formData.get(Select_ChangeType+"_value").asText("");
        }

        try {
        if(formData.has(Group_ChangeRequestorGroups) && formData.get(Group_ChangeRequestorGroups) !=null){
            JsonNode jsonNode = formData.get(Group_ChangeRequestorGroups);
            groupId = jsonNode.get(0).get("groupId").asText("");
            }
        }catch (Exception e){
            log.error("ClosedCancelBannerButton Group_ChangeRequestorGroups error:{}",formData);
        }

        if(!"ECR".equals(changeType) && "Approved".equals(crStatus) && "Mainframe Package Deployment".equals(mainframeCRType) && (groupId != null && loginUserGroupId != null && groupId.equals(String.valueOf(loginUserGroupId)))){
            log.info("ClosedCancelBannerButton handleCrAction changeType:{}",changeType);
            return resultBo;
        }else {
            resultBo.setSkipSysExecutor(false);
            resultBo.setCode(ResultCodeEnum.SUCCESS);
            handleStatus2CloseCancel(mdlInstance);
            return resultBo;
        }
    }

    public void handleStatus2CloseCancel(MdlInstance mdlInstance){
        JsonNode formData = com.cloudwise.dosm.core.utils.JsonUtils.parseJsonNode(mdlInstance.getFormData());
        ObjectNode formDataResult = (ObjectNode) formData;
        DataDictDetailMapper dataDictDetailMapper = com.cloudwise.dosm.core.utils.SpringContextUtils.getBean(DataDictDetailMapper.class);
        DataDictMapper dataDictMapper = com.cloudwise.dosm.core.utils.SpringContextUtils.getBean(DataDictMapper.class);
        DataDict dataDict = new DataDict();
        dataDict.setIsDel(0);
        dataDict.setDictCode("crStatus");
        DataDict crStatusDataDict = dataDictMapper.selectOneByParam(dataDict);
        List<DataDictDetail> dataDictDetails = dataDictDetailMapper.selectDetailByDictIdAndLevel(crStatusDataDict.getId(), 1, crStatusDataDict.getAccountId());
        Optional<DataDictDetail> first = dataDictDetails.stream().filter(item -> item.getData().equals("Closed Cancel")).findFirst();
        if (first.isPresent()) {
            formDataResult.put("crStatus", first.get().getId());
            formDataResult.put("crStatus_value", "Closed Cancel");
        }
        mdlInstance.setFormData(com.cloudwise.dosm.core.utils.JsonUtils.toJsonString(formDataResult));
        mdlInstanceMapper.updateByIdSelective(mdlInstance);
    }
}
