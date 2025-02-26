SELECT
JSON_VALUE(mi.form_data,'$.AnyapplicationCode_value') AS cmcappcodechange,
JSON_VALUE(mi.form_data,'$.AnyapplicationCode') AS cmcappcodechange_id,
JSON_VALUE(mi.form_data,'$.appResiliencyClassification_value') AS appResiliencyClassification,
JSON_VALUE(mi.form_data,'$.appResiliencyClassification') AS appResiliencyClassification_id,
JSON_EXTRACT(mi.form_data,'$.applicationImpacte_value') AS applicationimpacted,
JSON_EXTRACT(mi.form_data,'$.applicationImpacte') AS applicationimpacted_id,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ApplicationOwner[0].userName'),'$') AS ApplicationOwner,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.Approver01[0].userName'),'$') AS approver1,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.Approver02[0].userName'),'$') AS approver2,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.Approver03[0].userName'),'$') AS approver3,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.Approver04[0].userName'),'$') AS approver4,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.Approver05[0].userName'),'$') AS approver5,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.Approver06[0].userName'),'$') AS approver6,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ApproverGroup1[0].groupName'),'$') AS approvergroup1,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ApproverGroup2[0].groupName'),'$') AS approvergroup2,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ApproverGroup3[0].groupName'),'$') AS approvergroup3,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ApproverGroupfour[0].groupName'),'$') AS approvergroup4,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ApproverGroupfive[0].groupName'),'$') AS approvergroup5,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ApproverGroupsix[0].groupName'),'$') AS approvergroup6,
JSON_VALUE(mi.form_data,'$.BackoutComplexity') AS cmrmimplement_id,
JSON_VALUE(mi.form_data,'$.BackoutComplexity_value') AS cmrmimplement,
mi.biz_key as itsmcrnumber,
JSON_VALUE(mi.form_data,'$.blastRadiusDescription_value') AS potentialblastradius,
JSON_VALUE(mi.form_data,'$.blastRadiusDescription') AS potentialblastradius_id,
JSON_VALUE(mi.form_data,'$.BusinessUnitsOperations_value') AS cmrmcontinuity,
JSON_VALUE(mi.form_data,'$.BusinessUnitsOperations') AS cmrmcontinuity_id,
JSON_VALUE(mi.form_data,'$.Change_Category') AS Change_Category,
JSON_VALUE(mi.form_data,'$.ChangeComplexity_value') AS cmrmcomplexity,
JSON_VALUE(mi.form_data,'$.ChangeComplexity') AS cmrmcomplexity_id,
JSON_VALUE(mi.form_data,'$.ChangeCountForMainframe') AS changecount,
JSON_VALUE(mi.form_data,'$.changeDescribtion') AS description,
JSON_VALUE(mi.form_data,'$.changeDescriptionDuringBusinessH_value') AS changeduringonlinebusinesshours,
JSON_VALUE(mi.form_data,'$.changeDescriptionDuringBusinessH') AS changeduringonlinebusinesshours_id,
JSON_VALUE(mi.form_data,'$.changeGroup_value') AS changegroup,
JSON_VALUE(mi.form_data,'$.changeGroup') AS changegroup_id,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ChangeManagerGroup_one[0].groupName'),'$') AS changemanagergroup,
JSON_VALUE(mi.form_data,'$.changeNature_value') AS changenature,
JSON_VALUE(mi.form_data,'$.changeNature') AS changenature_id,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.changeRequestorGroups[0].groupName'),'$') AS changerequestorgroups,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.ChangeSchedule_StartEndTime.endDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS scheduleenddatetime,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.ChangeSchedule_StartEndTime.startDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS schedulestartdatetime,
JSON_VALUE(mi.form_data,'$.changeSummary') AS summary,
JSON_VALUE(mi.form_data,'$.changeType_value') AS changetype,
JSON_VALUE(mi.form_data,'$.changeType') AS changetype_id,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.checker[0].userName'),'$') AS cmcchecker,
JSON_VALUE(mi.form_data,'$.ClosedLogSummary_value') AS closedincompletereasoncode,
JSON_VALUE(mi.form_data,'$.ClosedLogSummary') AS closedincompletereasoncode_id,
JSON_VALUE(mi.form_data,'$.CloseOnlineFilesDuringJobRun_value') AS closeonlinefilesduringjobrun,
JSON_VALUE(mi.form_data,'$.CloseOnlineFilesDuringJobRun') AS closeonlinefilesduringjobrun_id,
JSON_VALUE(mi.form_data,'$.Code_Checker_value') AS codechecker,
JSON_VALUE(mi.form_data,'$.Code_Checker') AS codechecker_id,
JSON_VALUE(mi.form_data,'$.CountryImpacte_value') AS countryimpacted,
JSON_VALUE(mi.form_data,'$.CountryImpacte') AS countryimpacted_id,
JSON_VALUE(mi.form_data,'$.countryOfOrigin_value') AS countryoforigin,
JSON_VALUE(mi.form_data,'$.countryOfOrigin') AS countryoforigin_id,
JSON_VALUE(mi.form_data,'$.crClassification_value') AS crclassification,
JSON_VALUE(mi.form_data,'$.crClassification') AS crclassification_id,
JSON_VALUE(mi.form_data,'$.crStatus_value') AS state,
JSON_VALUE(mi.form_data,'$.crStatus') AS state_id,
'crUrl' as implementationplan,
'crUrl' as reversionplan,
JSON_VALUE(mi.form_data,'$.CUSsignoff_value') AS cussignofftype,
JSON_VALUE(mi.form_data,'$.CUSsignoff') AS cussignofftype_id,
JSON_VALUE(mi.form_data,'$.CustomerServices_value') AS cmrmavailablity,
JSON_VALUE(mi.form_data,'$.CustomerServices') AS cmrmavailablity_id,
JSON_VALUE(mi.form_data,'$.CyberArk_Object') AS cyberarkobjects,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.d4dDataOwner[0].userName'),'$') AS designfordataownername,
JSON_VALUE(mi.form_data,'$.D4Dsignoff_value') AS d4dsignofftype,
JSON_VALUE(mi.form_data,'$.D4Dsignoff') AS d4dsignofftype_id,
JSON_VALUE(mi.form_data,'$.DataCenterOPSsignoff_value') AS datacentersignofftype,
JSON_VALUE(mi.form_data,'$.DataCenterOPSsignoff') AS datacentersignofftype_id,
JSON_VALUE(mi.form_data,'$.dataPatchNumberOfRecords') AS cmcdatapatchnumberofrecord,
JSON_VALUE(mi.form_data,'$.dataRequirementsDetails') AS designfordatarequirements,
JSON_VALUE(mi.form_data,'$.Deployment_Approach_value') AS deploymentapproach,
JSON_VALUE(mi.form_data,'$.Deployment_Approach') AS deploymentapproach_id,
JSON_VALUE(mi.form_data,'$.Documentation_value') AS cmrmtraining,
JSON_VALUE(mi.form_data,'$.Documentation') AS cmrmtraining_id,
JSON_VALUE(mi.form_data,'$.DRCapabilitiesImpact_value') AS drcapabilitiesimpact,
JSON_VALUE(mi.form_data,'$.DRCapabilitiesImpact') AS drcapabilitiesimpact_id,
JSON_VALUE(mi.form_data,'$.DroneTicket') AS releaseticketslist,
JSON_VALUE(mi.form_data,'$.explainability') AS explainability,
JSON_VALUE(mi.form_data,'$.explainabilityBase') AS explainabilitybase,
JSON_VALUE(mi.form_data,'$.explainabilityScores') AS explainabilityscores,
JSON_VALUE(mi.form_data,'$.featureValues') AS featurevalues,
JSON_VALUE(mi.form_data,'$.HADRFlipsignoff_value') AS hasignofftype,
JSON_VALUE(mi.form_data,'$.HADRFlipsignoff') AS hasignofftype_id,
JSON_VALUE(mi.form_data,'$.Holdbatch_job_value') AS holdbatchjob,
JSON_VALUE(mi.form_data,'$.Holdbatch_job') AS holdbatchjob_id,
JSON_VALUE(mi.form_data,'$.identifiedImpactAndRisks_value') AS impactriskidentified,
JSON_VALUE(mi.form_data,'$.identifiedImpactAndRisks') AS impactriskidentified_id,
JSON_VALUE(mi.form_data,'$.IDRsignoff_value') AS idrsignofftype,
JSON_VALUE(mi.form_data,'$.IDRsignoff') AS idrsignofftype_id,
JSON_VALUE(mi.form_data,'$.ImpactToMainframe_value') AS impacttomainframetype,
JSON_VALUE(mi.form_data,'$.ImpactToMainframe') AS impacttomainframetype_id,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.Implementation_Time.endDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS implementationplanscheduleenddate,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.Implementation_Time.startDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS implementationplanschedulestart,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.implementergroup[*].groupName'),'$') AS implementergroup,
JSON_VALUE(mi.form_data,'$.InherentandResidualrisks_value') AS cmrminherentresidualrisks,
JSON_VALUE(mi.form_data,'$.InherentandResidualrisks') AS cmrminherentresidualrisks_id,
JSON_VALUE(mi.form_data,'$.interdependenciesDescription_value') AS majorinterdependencies,
JSON_VALUE(mi.form_data,'$.interdependenciesDescription') AS majorinterdependencies_id,
JSON_VALUE(mi.form_data,'$.interfaceImpact_value') AS interfaceImpact,
JSON_VALUE(mi.form_data,'$.interfaceImpact') AS interfaceImpact_id,
JSON_VALUE(mi.form_data,'$.Interfaces_value') AS cmrminterfaces,
JSON_VALUE(mi.form_data,'$.Interfaces') AS cmrminterfaces_id,
JSON_VALUE(mi.form_data,'$.IsMajorChange_value') AS majorchangeflag,
JSON_VALUE(mi.form_data,'$.IsMajorChange') AS majorchangeflag_id,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ISSGroup[*].groupName'),'$') AS issoverduepatchapprovergroup,
JSON_VALUE(mi.form_data,'$.IsthisCRinscope_forD4D_value') AS designfordatachoice,
JSON_VALUE(mi.form_data,'$.IsthisCRinscope_forD4D') AS designfordatachoice_id,
JSON_VALUE(mi.form_data,'$.JobNeedToAccessOnlineFiles_value') AS jobneedtoaccessonlinefiles,
JSON_VALUE(mi.form_data,'$.JobNeedToAccessOnlineFiles') AS jobneedtoaccessonlinefiles_id,
JSON_VALUE(mi.form_data,'$.jobType_value') AS jobtype,
JSON_VALUE(mi.form_data,'$.jobType') AS jobtype_id,
JSON_VALUE(mi.form_data,'$.justificationReason_value') AS designfordatajustification,
JSON_VALUE(mi.form_data,'$.justificationReason') AS designfordatajustification_id,
JSON_VALUE(mi.form_data,'$.JustificationwhyCategory') AS diffcatselectedfrmaimlreason,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.L1Change_Manager[0].userName'),'$') AS l1changemanagerapprovername,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.L1Risk_Manager[0].userName'),'$') AS riskmanagerapprovername,
JSON_VALUE(mi.form_data,'$.Listtheimpactedbythechange') AS servicesimpactedandrecoveryplan,
JSON_VALUE(mi.form_data,'$.liveVerificationAfterImplementat_value') AS liveverification,
JSON_VALUE(mi.form_data,'$.liveVerificationAfterImplementat') AS liveverification_id,
JSON_VALUE(mi.form_data,'$.lob_value') AS lob,
JSON_VALUE(mi.form_data,'$.lob') AS lob_id,
JSON_VALUE(mi.form_data,'$.location_value') AS location,
JSON_VALUE(mi.form_data,'$.location') AS location_id,
JSON_VALUE(mi.form_data,'$.lpar_value') AS lpar,
JSON_VALUE(mi.form_data,'$.lpar') AS lpar_id,
JSON_VALUE(mi.form_data,'$.mainApplicationRequiringChange_value') AS mainappimpacted,
JSON_VALUE(mi.form_data,'$.mainApplicationRequiringChange') AS mainappimpacted_id,
JSON_VALUE(mi.form_data,'$.mainframeCRType_value') AS mainframecrtype,
JSON_VALUE(mi.form_data,'$.mainframeCRType') AS mainframecrtype_id,
JSON_VALUE(mi.form_data,'$.mainframeCRType_value') AS mainframecrtype,
JSON_VALUE(mi.form_data,'$.mainframeCRType') AS mainframecrtype_id,
JSON_VALUE(mi.form_data,'$.MainframePackage_Type_value') AS mainframepackagetype,
JSON_VALUE(mi.form_data,'$.MainframePackage_Type') AS mainframepackagetype_id,
JSON_VALUE(mi.form_data,'$.mainframePackageCreator1BankID') AS mainframepackagecreator1bankid,
JSON_VALUE(mi.form_data,'$.mainframePackageCreatorTSOID') AS mainframepackagecreatortsoid,
JSON_VALUE(mi.form_data,'$.mainframePackageName') AS mainframepackagename,
JSON_VALUE(mi.form_data,'$.Mainframeritica_value') AS mainframecriticalmonthendbatchimpact,
JSON_VALUE(mi.form_data,'$.Mainframeritica') AS mainframecriticalmonthendbatchimpact_id,
JSON_VALUE(mi.form_data,'$.MajorChanges_value') AS cmcmajorchange,
JSON_VALUE(mi.form_data,'$.MajorChanges') AS cmcmajorchange_id,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.maker[0].userName'),'$') AS cmcmaker,
JSON_VALUE(mi.form_data,'$.MakerChecker_value') AS cmcmakerchecker,
JSON_VALUE(mi.form_data,'$.MakerChecker') AS cmcmakerchecker_id,
JSON_VALUE(mi.form_data,'$.MASCategory_value') AS cmcmascategory,
JSON_VALUE(mi.form_data,'$.MASCategory') AS cmcmascategory_id,
JSON_VALUE(mi.form_data,'$.MDDelegateSignOff_value') AS mddelegatesignofftype,
JSON_VALUE(mi.form_data,'$.MDDelegateSignOff') AS mddelegatesignofftype_id,
JSON_VALUE(mi.form_data,'$.NewApplicationServices') AS servicemonitoringsnocappservicelist,
JSON_VALUE(mi.form_data,'$.otherApplicationImpacted_value') AS otherapplicationimpacted,
JSON_VALUE(mi.form_data,'$.otherApplicationImpacted') AS otherapplicationimpacted_id,
JSON_VALUE(mi.form_data,'$.parentChange') AS parentchange,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.projectCutOverProjectManager[0].userName'),'$') AS ticketprojectmanager,
JSON_VALUE(mi.form_data,'$.projectCutOverProjectManagerCont') AS ticketprojectmanagermobile,
JSON_VALUE(mi.form_data,'$.projectCutOverProjectObjective') AS projectobjective,
JSON_VALUE(mi.form_data,'$.projectCutOverProjectScope') AS projectscope,
JSON_VALUE(mi.form_data,'$.recommendedRisk') AS recommendedrisklevel,
JSON_VALUE(mi.form_data,'$.regressiontesting_value') AS regressiontesting,
JSON_VALUE(mi.form_data,'$.regressiontesting') AS regressiontesting_id,
JSON_VALUE(mi.form_data,'$.relatedIncident') AS relatedincident,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ReleaseManagerGroup[0].groupName'),'$') AS releasemanagergroup,
JSON_VALUE(mi.form_data,'$.Resource_Engagement_value') AS resourceengagement,
JSON_VALUE(mi.form_data,'$.Resource_Engagement') AS resourceengagement_id,
JSON_VALUE(mi.form_data,'$.reversionapproach_value') AS reversionapproach,
JSON_VALUE(mi.form_data,'$.reversionapproach') AS reversionapproach_id,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.Reversion_Time.endDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS reversionplanscheduleenddate,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.Reversion_Time.startDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS reversionplanschedulestartdate,
JSON_VALUE(mi.form_data,'$.ReversionBackoutRollbackTesting_value') AS rollbacktesting,
JSON_VALUE(mi.form_data,'$.ReversionBackoutRollbackTesting') AS rollbacktesting_id,
JSON_VALUE(mi.form_data,'$.riskMitigationPlan_value') AS riskmitigated,
JSON_VALUE(mi.form_data,'$.riskMitigationPlan') AS riskmitigated_id,
JSON_VALUE(mi.form_data,'$.riskScore') AS riskscore,
JSON_VALUE(mi.form_data,'$.riskThreshold') AS riskthreshold,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.ScheduledDowntime_Time.endDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS downtimeenddatetime,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.ScheduledDowntime_Time.startDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS downtimestartdatetime,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.ScheduledMaintenance_Time.endDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS scheduledmaintenanceenddatetime,
DATE_FORMAT(FROM_UNIXTIME(CAST(JSON_EXTRACT(mi.form_data,'$.ScheduledMaintenance_Time.startDate') AS UNSIGNED )/1000),'%Y-%m-%d %H:%i:%s') AS scheduledmaintenancestartdatetime,
JSON_VALUE(mi.form_data,'$.Security_value') AS cmrmsecurity,
JSON_VALUE(mi.form_data,'$.Security') AS cmrmsecurity_id,
JSON_VALUE(mi.form_data,'$.SNOCsignoff_value') AS servicemonitoringsnoctype,
JSON_VALUE(mi.form_data,'$.SNOCsignoff') AS servicemonitoringsnoctype_id,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.Tech_MDApprover[0].userName'),'$') AS mdapprover,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ScheduledApprover1[0].userName'),'$') AS downtimeapprover1_name,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ScheduledApprover2[0].userName'),'$') AS downtimeapprover2_name,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ScheduledApprover3[0].userName'),'$') AS downtimeapprover3_name,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ScheduledApprover4[0].userName'),'$') AS downtimeapprover4_name,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.ScheduledApprover5[0].userName'),'$') AS downtimeapprover5_name,
JSON_VALUE(JSON_EXTRACT(mi.form_data,'$.TechMDApproverGroup[0].groupName'),'$') AS mdapprovergroup,
JSON_VALUE(mi.form_data,'$.TheProductionDashboardLinks') AS servicemonitoringsnocurllist,
JSON_VALUE(mi.form_data,'$.uat_value') AS uat,
JSON_VALUE(mi.form_data,'$.uat') AS uat_id,
JSON_VALUE(mi.form_data,'$.URLForProjectDocsandArtefac') AS alldocsandartifactsurl,
JSON_VALUE(mi.form_data,'$.UseHighCPUIncreaseWorkload_value') AS usehighcpuincreaseworkload,
JSON_VALUE(mi.form_data,'$.UseHighCPUIncreaseWorkload') AS usehighcpuincreaseworkload_id,
(SELECT JSON_VALUE(JSON_EXTRACT(sign_off_user,'$[0].userName'),'$') FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','BU/Application Owner Signoff') IS NOT null AND
work_order_id = mi.id ) AS busignoffapproverlogin  ,
(SELECT
case `status`
when 'APPROVED' then 'Approved'
when 'REJECTED' then 'Rejected'
ELSE ''
END
FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','BU/Application Owner Signoff') IS NOT null AND
work_order_id = mi.id ) AS busignoffstatus,
(SELECT rejection_reason FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','BU/Application Owner Signoff') IS NOT null AND
work_order_id = mi.id ) AS busignoffrejectionreason,

(SELECT JSON_VALUE(JSON_EXTRACT(sign_off_user,'$[0].userName'),'$') FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Design For Data (D4D) Signoff') IS NOT null AND
work_order_id = mi.id ) AS d4dsignoffapproverlogin  ,
(SELECT
case `status`
when 'APPROVED' then 'Approved'
when 'REJECTED' then 'Rejected'
ELSE ''
END
FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Design For Data (D4D) Signoff') IS NOT null AND
work_order_id = mi.id ) AS d4dsignoffstatus,
(SELECT rejection_reason FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Design For Data (D4D) Signoff') IS NOT null AND
work_order_id = mi.id ) AS d4dsignoffrejectionreason,

(SELECT JSON_VALUE(JSON_EXTRACT(sign_off_user,'$[0].userName'),'$') FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Data Center OPS (Batch) Signoff') IS NOT null AND
work_order_id = mi.id ) AS datacentersignoffapproverlogin  ,
(SELECT
case `status`
when 'APPROVED' then 'Approved'
when 'REJECTED' then 'Rejected'
ELSE ''
END
FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Data Center OPS (Batch) Signoff') IS NOT null AND
work_order_id = mi.id ) AS datacentersignoffstatus,
(SELECT rejection_reason FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Data Center OPS (Batch) Signoff') IS NOT null AND
work_order_id = mi.id ) AS datacentersignoffrejectionreason,

(SELECT JSON_VALUE(JSON_EXTRACT(sign_off_user,'$[0].userName'),'$') FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','HA & DR Flip Signoff') IS NOT null AND
work_order_id = mi.id ) AS hasignoffapproverlogin  ,
(SELECT
case `status`
when 'APPROVED' then 'Approved'
when 'REJECTED' then 'Rejected'
ELSE ''
END
FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','HA & DR Flip Signoff') IS NOT null AND
work_order_id = mi.id ) AS hasignoffstatus,
(SELECT rejection_reason FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','HA & DR Flip Signoff') IS NOT null AND
work_order_id = mi.id ) AS hasignoffrejectionreason,

(SELECT JSON_VALUE(JSON_EXTRACT(sign_off_user,'$[0].userName'),'$') FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Impact To Mainframe Signoff') IS NOT null AND
work_order_id = mi.id ) AS impacttomainframeapproverlogin  ,
(SELECT
case `status`
when 'APPROVED' then 'Approved'
when 'REJECTED' then 'Rejected'
ELSE ''
END
FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Impact To Mainframe Signoff') IS NOT null AND
work_order_id = mi.id ) AS impacttomainframestatus,
(SELECT rejection_reason FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','Impact To Mainframe Signoff') IS NOT null AND
work_order_id = mi.id ) AS impacttomainframerejectionreason,

--(SELECT JSON_VALUE(JSON_EXTRACT(sign_off_user,'$[0].userName'),'$') FROM sign_off_manager
--WHERE JSON_SEARCH(sign_off_type,'one','ISS Signoff') IS NOT null AND
--work_order_id = mi.id ) AS issoverduepatchapprover  ,
--(SELECT
--case `status`
--when 'APPROVED' then 'Approved'
--when 'REJECTED' then 'Rejected'
--ELSE ''
--END
--FROM sign_off_manager
--WHERE JSON_SEARCH(sign_off_type,'one','ISS Signoff') IS NOT null AND
--work_order_id = mi.id ) AS issoverduepatchapproverstatus,
--(SELECT rejection_reason FROM sign_off_manager
--WHERE JSON_SEARCH(sign_off_type,'one','ISS Signoff') IS NOT null AND
--work_order_id = mi.id ) AS issoverduepatchapproverrejectioncode,



(SELECT JSON_VALUE(JSON_EXTRACT(sign_off_user,'$[0].userName'),'$') FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','MD Delegate Signoff') IS NOT null AND
work_order_id = mi.id ) AS mddelegatesignoffapproverlogin  ,
(SELECT
case `status`
when 'APPROVED' then 'Approved'
when 'REJECTED' then 'Rejected'
ELSE ''
END
FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','MD Delegate Signoff') IS NOT null AND
work_order_id = mi.id ) AS mddelegatesignoffstatus,
(SELECT rejection_reason FROM sign_off_manager
WHERE JSON_SEARCH(sign_off_type,'one','MD Delegate Signoff') IS NOT null AND
work_order_id = mi.id ) AS mddelegatesignoffrejectionreason,

JSON_VALUE(rca.form_data,'$.whatwasthecorrectiveactiontaken') as crfailcorrectiveaction,
JSON_VALUE(rca.form_data,'$.impactedapplication') as crfailedforappcode,
JSON_VALUE(rca.form_data,'$.anypreventivemeasuretoavoidthis') as crfailpreventivemeasure,
JSON_VALUE(rca.form_data,'$.anyincidentticketraisedduetothe') as relatedincidentfromcrfail,
JSON_VALUE(rca.form_data,'$.whythechangeimplementationfailed') as crfailreason,
JSON_VALUE(rca.form_data,'$.whatistherootcause') as crfailrootcause,
JSON_VALUE(rca.form_data,'$.RCAStatus') as rcastatus_id,
JSON_VALUE(rca.form_data,'$.RCAStatus_value') as rcastatus_id,
JSON_VALUE(rca.form_data,'$.IncidentSummary') as rcaincidentsummary,
JSON_VALUE(rca.form_data,'$.whatisthefailedchangecategor') as failedcrcategory,
JSON_VALUE(rca.form_data,'$.whenisthetargetdateofcompletion') as targetisssuefixdate,
JSON_VALUE(JSON_EXTRACT(rca.form_data,'$.appmanagerapproval[0].userName'),'$') as rcaapprover1,


(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0wb3mla' order by updated_time desc limit 1  ) as approver1rejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0wb3mla' order by updated_time desc limit 1  ) as approver1statusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0wb3mla' order by updated_time desc limit 1  ) as approver1status,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1y2bas2' order by updated_time desc limit 1  ) as approver2rejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1y2bas2' order by updated_time desc limit 1  ) as approver2statusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1y2bas2' order by updated_time desc limit 1  ) as approver2status,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_09y6ddj' order by updated_time desc limit 1  ) as approver3rejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_09y6ddj' order by updated_time desc limit 1  ) as approver3statusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_09y6ddj' order by updated_time desc limit 1  ) as approver3status,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_112fawy' order by updated_time desc limit 1  ) as approver4rejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_112fawy' order by updated_time desc limit 1  ) as approver4statusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_112fawy' order by updated_time desc limit 1  ) as approver5status,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_01zjgji' order by updated_time desc limit 1  ) as approver5rejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_01zjgji' order by updated_time desc limit 1  ) as approver5statusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_01zjgji' order by updated_time desc limit 1  ) as approver6status,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0fh11tl' order by updated_time desc limit 1  ) as approver6rejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0fh11tl' order by updated_time desc limit 1  ) as approver6statusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0fh11tl' order by updated_time desc limit 1  ) as approver6status,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0fh11tl' order by updated_time desc limit 1  ) as approver6rejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0fh11tl' order by updated_time desc limit 1  ) as approver6statusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0fh11tl' order by updated_time desc limit 1  ) as approver6status,


(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_01kn76b' order by updated_time desc limit 1  ) as changemanagerrejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_01kn76b' order by updated_time desc limit 1  ) as changemanagerstatusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_01kn76b' order by updated_time desc limit 1  ) as changemanagerapproverstatus,


(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1tkhamt' order by updated_time desc limit 1  ) as designfordatarejectioncomment,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1tkhamt' order by updated_time desc limit 1  ) as designfordataapproverstatus,

(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0fjvryb' order by updated_time desc limit 1  ) as downtimeapproverrejectioncomment1,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0fjvryb' order by updated_time desc limit 1  ) as downtimeapproverstatus1,

(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0uvdhk1' order by updated_time desc limit 1  ) as downtimeapproverrejectioncomment2,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0uvdhk1' order by updated_time desc limit 1  ) as downtimeapproverstatus2,

(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0wv160x' order by updated_time desc limit 1  ) as downtimeapproverrejectioncomment3,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0wv160x' order by updated_time desc limit 1  ) as downtimeapproverstatus3,

(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1lir1bh' order by updated_time desc limit 1  ) as downtimeapproverrejectioncomment4,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1lir1bh' order by updated_time desc limit 1  ) as downtimeapproverstatus4,

(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0zy9gcq' order by updated_time desc limit 1  ) as downtimeapproverrejectioncomment5,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0zy9gcq' order by updated_time desc limit 1  ) as downtimeapproverstatus5,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_10vwuti' order by updated_time desc limit 1  ) as issoverduepatchapproverrejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_10vwuti' order by updated_time desc limit 1  ) as issoverduepatchapproverstatusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_10vwuti' order by updated_time desc limit 1  ) as issoverduepatchapproverstatus,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1kslrrp' order by updated_time desc limit 1  ) as l1changemanagerapproverrejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1kslrrp' order by updated_time desc limit 1  ) as l1changemanagerstatusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1kslrrp' order by updated_time desc limit 1  ) as l1changemanagerapproverstatus,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0armfga' order by updated_time desc limit 1  ) as mdapproverrejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0armfga' order by updated_time desc limit 1  ) as mdstatusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_0armfga' order by updated_time desc limit 1  ) as mdapproverstatus,

(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1dv8pri' order by updated_time desc limit 1  ) as riskmanagerapproverrejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1dv8pri' order by updated_time desc limit 1  ) as riskmanagerstatusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_1dv8pri' order by updated_time desc limit 1  ) as riskmanagerapproverstatus,


(select reason from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_13119ha' order by updated_time desc limit 1  ) as rmapproverrejectioncode,
(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_13119ha' order by updated_time desc limit 1  ) as rmapproverrejectionreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = mi.id and mard.node_id = 'SingleApprove_13119ha' order by updated_time desc limit 1  ) as rmapproverstatus,


(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = rca.id and mard.node_id = 'SingleApprove_1ojunx3' order by updated_time desc limit 1  ) as rcaapprover1statusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = rca.id and mard.node_id = 'SingleApprove_1ojunx3' order by updated_time desc limit 1  ) as rcaapprover1status,


(select approve_msg from mdl_approve_record_detail mard
where mard.work_order_id = rca.id and mard.node_id = 'SingleApprove_1ojunx3' order by updated_time desc limit 1  ) as rcacmapproverstatusreason,
(select
case  approve_type
when 'PASS' then 'Approved'
when 'REJECT' then 'Rejected'
else null
end
from mdl_approve_record_detail mard
where mard.work_order_id = rca.id and mard.node_id = 'SingleApprove_1ojunx3' order by updated_time desc limit 1  ) as rcacmapproverstatus



FROM mdl_instance mi
LEFT JOIN mdl_instance rca ON rca.biz_key = CONCAT('RCA-',mi.biz_key)
WHERE mi.biz_key LIKE 'CR20250225%';

