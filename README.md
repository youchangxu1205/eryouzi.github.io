

package com.cloudwise.douc.customization.biz.service.ichampsync;


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
        BU_APPLICATION_OWNER_SIGNOFF("BU/Application Owner Signoff", "busignoffstatus","busignoffapproverlogin" ,"busignoffrejectionreason" ,null ),
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

//        ISS_SIGNOFF("ISS Signoff", "issoverduepatchapproverstatus", "issoverduepatchapprover", "issoverduepatchapproverrejectioncode",null );
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

{
  "pushUrl": "https://itsm-dev.cmwf.ocp.uat.dbs.com/common-server/ichamp/createCR",
  "updateUrl": "https://itsm-dev.cmwf.ocp.uat.dbs.com/common-server/ichamp/updateCR",
  "pushMethod": "POST",
  "createNodeId": "UserTask_13ikwht",
  "closeNodeId": "UserTask_1gehich",
  "crStatusFieldCode": "crStatus",
  "crStatusDictCode": "crStatus",
  "closeStatusFieldCode": "CloseStatus",
  "signoffNodeId": "UserTask_0da3d8t",
  "implementationNodeId": "AcceptanceTask_0qba8sk",
  "fieldMapping": {
    "summary": "changeSummary:TEXTAREA",
    "description": "changeDescribtion:TEXTAREA",
    "changerequestorgroups": "changeRequestorGroups:GROUP",
    "countryoforigin": "countryOfOrigin:SELECT",
    "lob": "lob:SELECT",
    "changetype": "changeType:SELECT",
    "changenature": "changeNature:SELECT",
    "changegroup": "changeGroup:SELECT",
    "cmcdatapatchnumberofrecord": "dataPatchNumberOfRecords:NUMBER",
    "mainframecrtype": "mainframeCRType:SELECT",
    "crclassification": "crClassification:SELECT",
    "location": "location:SELECT",
    "countryimpacted": "CountryImpacte:SELECT_MANY",
    "applicationimpacted": "applicationImpacte:SELECT_MANY",
    "otherapplicationimpacted": "otherApplicationImpacted:SELECT_MANY",
    "mainappimpacted": "mainApplicationRequiringChange:SELECT",
    "changecategory": "Change_Category:INPUT",
    "appresiliencyclassification": "appResiliencyClassification:SELECT",
    "lpar": "lpar:SELECT",
    "servicesimpactedandrecoveryplan": "Listtheimpactedbythechange:TEXTAREA",
    "riskscore": "riskScore:INPUT",
    "featurevalues": "featureValues:INPUT",
    "recommendedrisklevel": "recommendedRisk:INPUT",
    "explainability": "explainability:INPUT",
    "explainabilitybase": "explainabilityBase:INPUT",
    "explainabilityscores": "explainabilityScores:INPUT",
    "riskthreshold": "riskThreshold:INPUT",
    "diffcatselectedfrmaimlreason": "JustificationwhyCategory:TEXTAREA",
    "cmrmavailablity": "CustomerServices:SELECT",
    "cmrminherentresidualrisks": "InherentandResidualrisks:SELECT",
    "cmrmcontinuity": "BusinessUnitsOperations:SELECT",
    "cmrmcomplexity": "ChangeComplexity:SELECT",
    "cmrmimplement": "BackoutComplexity:SELECT",
    "cmrmtraining": "Documentation:SELECT",
    "cmrmsecurity": "Security:SELECT",
    "cmrminterfaces": "Interfaces:SELECT",
    "schedulestartdatetime,scheduleenddatetime": "ChangeSchedule_StartEndTime:DATE_RANGE",
    "implementationplanschedulestart,implementationplanscheduleenddate": "Implementation_Time:DATE_RANGE",
    "reversionplanschedulestartdate,reversionplanscheduleenddate": "Reversion_Time:DATE_RANGE",
    "downtimestartdatetime,downtimeenddatetime": "ScheduledDowntime_Time:DATE_RANGE",
    "scheduledmaintenancestartdatetime,scheduledmaintenanceenddatetime": "ScheduledMaintenance_Time:DATE_RANGE",
    "releaseticketslist": "DroneTicket:TEXTAREA",
    "liveverification": "liveVerificationAfterImplementat:SELECT",
    "uat": "uat:SELECT",
    "regressiontesting": "regressiontesting:SELECT",
    "rollbacktesting": "ReversionBackoutRollbackTesting:SELECT",
    "relatedincident": "relatedIncident:INPUT",
    "parentchange": "parentChange:INPUT",
    "designfordatachoice": "IsthisCRinscope_forD4D:SELECT",
    "designfordatarequirements": "dataRequirementsDetails:TEXTAREA",
    "designfordatajustification": "justificationReason:SELECT",
    "cyberarkobjects": "CyberArk_Object:TEXTAREA",
    "cmcappcodechange": "AnyapplicationCode:SELECT",
    "cmcmakerchecker": "MakerChecker:RADIO",
    "cmcmaker": "maker:MEMBER",
    "cmcchecker": "checker:MEMBER",
    "mainframepackagename": "mainframePackageName:INPUT",
    "mainframepackagetype": "MainframePackage_Type:SELECT",
    "mainframepackagecreatortsoid": "mainframePackageCreatorTSOID:INPUT",
    "mainframepackagecreator1bankid": "mainframePackageCreator1BankID:INPUT",
    "holdbatchjob": "Holdbatch_job::RADIO",
    "jobtype": "jobType:SELECT",
    "jobneedtoaccessonlinefiles": "JobNeedToAccessOnlineFiles:SELECT",
    "closeonlinefilesduringjobrun": "CloseOnlineFilesDuringJobRun:SELECT",
    "usehighcpuincreaseworkload": "UseHighCPUIncreaseWorkload:SELECT",
    "impactriskidentified": "identifiedImpactAndRisks:SELECT",
    "riskmitigated": "riskMitigationPlan:SELECT",
    "deploymentapproach": "Deployment_Approach:SELECT_MANY",
    "reversionapproach": "Reversion_Approach:SELECT_MANY",
    "resourceengagement": "Resource_Engagement:SELECT_MANY",
    "potentialblastradius": "blastRadiusDescription:SELECT",
    "changeduringonlinebusinesshours": "changeDescriptionDuringBusinessH:SELECT",
    "mainframecriticalmonthendbatchimpact": "Mainframeritica:SELECT",
    "majorinterdependencies": "interdependenciesDescription:SELECT",
    "drcapabilitiesimpact": "DRCapabilitiesImpact:SELECT",
    "upstreamdownstreaminterfacesimpact": "interfaceImpact:SELECT",
    "implementergroup": "implementergroup:GROUP",
    "hasignofftype": "HADRFlipsignoff:SELECT",
    "servicemonitoringsnocurllist": "TheProductionDashboardLinks:TEXTAREA",
    "servicemonitoringsnocappservicelist": "NewApplicationServices:TEXTAREA",
    "ticketprojectmanager": "projectCutOverProjectManager:MEMBER",
    "ticketprojectmanagermobile": "projectCutOverProjectManagerCont:INPUT",
    "projectobjective": "projectCutOverProjectObjective:INPUT",
    "projectscope": "projectCutOverProjectScope:INPUT",
    "alldocsandartifactsurl": "URLForProjectDocsandArtefac:TEXTAREA",
    "datacentersignofftype": "DataCenterOPSsignoff:SELECT",
    "impacttomainframetype": "ImpactToMainframe:SELECT",
    "d4dsignofftype": "D4Dsignoff:SELECT",
    "servicemonitoringsnoctype": "SNOCsignoff:SELECT",
    "cussignofftype": "CUSsignoff:SELECT",
    "idrsignofftype": "IDRsignoff:SELECT",
    "closedincompletereasoncode": "ClosedLogSummary:SELECT",
    "cmcmascategory": "MASCategory:SELECT",
    "changeriskupgradeddowngraded": "ChangeRiskUpgradedDowngraded:INPUT",
    "crsubmissiondatetime": "CRSubmissionDatetime:DATE",
    "codechecker": "Code_Checker:SELECT",
    "sicode": "standardCode:INPUT",
    "majorchangeflag": "IsMajorChange:RADIO",
    "implementationplan": "implementationPlan:CR_URL",
    "reversionplan": "reversionPlan:CR_URL",
    "cmcrollbackduration": "rollbackDurationInMinutes:INPUT",
    "securityriskjustification": "SecurityRiskJustification:TEXTAREA",
    "ticketfocalpoint": "ticketfocalpoint:INPUT",
    "ticketfocalpointmobile": "ticketfocalpointmobile:INPUT",
    "mccrequesterlogin": "mccrequesterlogin:MEMBER",
    "changecount": "ChangeCountForMainframe:NUMBER",
    "cmcmajorchange": "MajorChanges:SELECT_MANY",
    "state": "crStatus:SELECT"
  },
  "riskDictMapping": {
    "CustomerServices": "d7b6d17cf02c401a93144a6323915745",
    "InherentandResidualrisks": "f831cb01fd0f43b297a93ead68abc562",
    "BusinessUnitsOperations": "2008e9b96d5d413798d119b829648535",
    "ChangeComplexity": "845a783c49524348bd5804cfbc0ac06e",
    "BackoutComplexity": "c7199dacb70e4a07ad29dd898cf56860",
    "Documentation": "ff30d63471b14bc3870b90c849c66f7d",
    "Security": "3240ef33f1bd4dbcbb1ad5ac86bb0b53",
    "Interfaces": "aea98f47781147f8a8cb8eae078f7299"
  },
  "tableFieldMapping": {
    "mainframedata": [
      {
        "ichampTableFieldInfo": "Job Details",
        "itsmTableFieldInfo": "JobDetailsSection",
        "fieldMapping": {
          "jobname": "JobDetails:INPUT",
          "jobdatetime": "DateTimeModified:DATE",
          "jobduration": "Duration:NUMBER"
        }
      },
      {
        "ichampTableFieldInfo": "Load Module",
        "itsmTableFieldInfo": "LoadModuleDetailsSection",
        "fieldMapping": {
          "modulecompiled": "LoadModuleCompiled:INPUT",
          "moduledatetime": "DateTimeCompiled:DATE"
        }
      },
      {
        "ichampTableFieldInfo": "DBRM Members",
        "itsmTableFieldInfo": "DBRMMemberDetailsSection",
        "fieldMapping": {
          "dbrmmember": "DBRMMember:INPUT"
        }
      },
      {
        "ichampTableFieldInfo": "Input Data",
        "itsmTableFieldInfo": "InputDataDetailsSection",
        "fieldMapping": {
          "inputdata": "InputData:INPUT",
          "inputdatadatetime": "DateTimeLastModified:DATE"
        }
      }
    ]
  },
  "approveInfoMapping": [
    {
      "nodeId": "SingleApprove_0wb3mla",
      "nodeName": "App Govenance1",
      "rejectionCode": "approver1rejectioncode",
      "reasonCode": "approver1statusreason",
      "statusCode": "approver1status",
      "mapFieldMapping": {
        "approvergroup1": "ApproverGroup1:GROUP",
        "approver1": "Approver01:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_1y2bas2",
      "nodeName": "App Govenance2",
      "rejectionCode": "approver2rejectioncode",
      "reasonCode": "approver2statusreason",
      "statusCode": "approver2status",
      "mapFieldMapping": {
        "approvergroup2": "ApproverGroup2:GROUP",
        "approver2": "Approver02:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_09y6ddj",
      "nodeName": "App Govenance3",
      "rejectionCode": "approver3rejectioncode",
      "reasonCode": "approver3statusreason",
      "statusCode": "approver3status",
      "mapFieldMapping": {
        "approvergroup3": "ApproverGroup3:GROUP",
        "approver3": "Approver03:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_112fawy",
      "nodeName": "App Govenance4",
      "rejectionCode": "approver4rejectioncode",
      "reasonCode": "approver4statusreason",
      "statusCode": "approver4status",
      "mapFieldMapping": {
        "approvergroup4": "ApproverGroupfourr:GROUP",
        "approver4": "Approver04:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_01zjgji",
      "nodeName": "App Govenance5",
      "rejectionCode": "approver5rejectioncode",
      "reasonCode": "approver5statusreason",
      "statusCode": "approver5status",
      "mapFieldMapping": {
        "approvergroup5": "ApproverGroupfive:GROUP",
        "approver5": "Approver05:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_0fh11tl",
      "nodeName": "App Govenance6",
      "rejectionCode": "approver6rejectioncode",
      "reasonCode": "approver6statusreason",
      "statusCode": "approver6status",
      "mapFieldMapping": {
        "approvergroup6": "ApproverGroupsix:GROUP",
        "approver6": "Approver06:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_0cntz02",
      "nodeName": "Application Owner (For Data Patch)",
      "rejectionCode": "",
      "reasonCode": "applicationownercomment",
      "statusTimeCode": "appownerdatapatchapprovalstatustime",
      "statusCode": "applicationownerapprovalstatus",
      "mapFieldMapping": {
        "applicationownername": "ApplicationOwner:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_1tkhamt",
      "nodeName": "D4D Data Owner",
      "rejectionCode": "",
      "reasonCode": "designfordatarejectioncomment",
      "statusCode": "designfordataapproverstatus",
      "mapFieldMapping": {
        "designfordataowner,designfordataownername": "d4dDataOwner:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_10vwuti",
      "nodeName": "ISS Overdue Patch",
      "rejectionCode": "issoverduepatchapproverrejectioncode",
      "reasonCode": "issoverduepatchapproverstatusreason",
      "statusCode": "issoverduepatchapproverstatus",
      "mapFieldMapping": {
        "issoverduepatchapprovergroup": "ISSGroup:GROUP",
        "issoverduepatchapprover": "ISSApprover1:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_1kslrrp",
      "nodeName": "L1 Change Manager",
      "rejectionCode": "l1changemanagerapproverrejectioncode",
      "reasonCode": "l1changemanagerstatusreason",
      "statusCode": "l1changemanagerapproverstatus",
      "mapFieldMapping": {
        "l1changemanagerapprovername": "L1Change_Manager:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_1dv8pri",
      "nodeName": "L1 Risk Manager",
      "rejectionCode": "riskmanagerapproverrejectioncode",
      "reasonCode": "riskmanagerstatusreason",
      "statusCode": "riskmanagerapproverstatus",
      "mapFieldMapping": {
        "riskmanagerapprovername": "L1Risk_Manager:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_13119ha",
      "nodeName": "L1.5 Release Manager Team",
      "rejectionCode": "rmapproverrejectioncode",
      "reasonCode": "rmapproverrejectionreason",
      "statusCode": "rmapproverstatus",
      "mapFieldMapping": {
        "releasemanagergroup": "ReleaseManagerGroup:GROUP"
      }
    },
    {
      "nodeId": "SingleApprove_0fjvryb",
      "nodeName": "Scheduled Downtime/Maintenance Approver 1",
      "rejectionCode": "",
      "reasonCode": "downtimeapproverrejectioncomment1",
      "statusCode": "downtimeapproverstatus1",
      "mapFieldMapping": {
        "downtimeapprover1_,downtimeapprover1_name": "ScheduledApprover1:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_0uvdhk1",
      "nodeName": "Scheduled Downtime/Maintenance Approver 2",
      "rejectionCode": "",
      "reasonCode": "downtimeapproverrejectioncomment2",
      "statusCode": "downtimeapproverstatus2",
      "mapFieldMapping": {
        "downtimeapprover2_,downtimeapprover2_name": "ScheduledApprover2:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_0wv160x",
      "nodeName": "Scheduled Downtime/Maintenance Approver 3",
      "rejectionCode": "",
      "reasonCode": "downtimeapproverrejectioncomment3",
      "statusCode": "downtimeapproverstatus3",
      "mapFieldMapping": {
        "downtimeapprover3_,downtimeapprover3_name": "ScheduledApprover3:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_1lir1bh",
      "nodeName": "Scheduled Downtime/Maintenance Approver 4",
      "rejectionCode": "",
      "reasonCode": "downtimeapproverrejectioncomment4",
      "statusCode": "downtimeapproverstatus4",
      "mapFieldMapping": {
        "downtimeapprover4_,downtimeapprover4_name": "ScheduledApprover4:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_0zy9gcq",
      "nodeName": "Scheduled Downtime/Maintenance Approver 5",
      "rejectionCode": "",
      "reasonCode": "downtimeapproverrejectioncomment5",
      "statusCode": "downtimeapproverstatus5",
      "mapFieldMapping": {
        "downtimeapprover5_,downtimeapprover5_name": "ScheduledApprover5:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_0armfga",
      "nodeName": "Tech MD",
      "rejectionCode": "mdapproverrejectioncode",
      "reasonCode": "mdstatusreason",
      "statusCode": "mdapproverstatus",
      "mapFieldMapping": {
        "mdapprovergroup": "TechMDApproverGroup:GROUP",
        "mdapprover": "Tech_MDApprover:MEMBER"
      }
    },
    {
      "nodeId": "SingleApprove_01kn76b",
      "nodeName": "L1.5 Change Management Team",
      "rejectionCode": "changemanagerrejectioncode",
      "reasonCode": "changemanagerstatusreason",
      "statusCode": "changemanagerapproverstatus",
      "mapFieldMapping": {
        "changemanagergroup": "ChangeManagerGroup_OP:GROUP"
      }
    }
  ],
  "rcaApproveInfoMapping": [
    {
      "nodeId": "SingleApprove_1ojunx3",
      "nodeName": "App Manager Approval",
      "rejectionCode": "",
      "reasonCode": "rcaapprover1statusreason",
      "statusCode": "rcaapprover1status"
    },
    {
      "nodeId": "SingleApprove_1r1kd6y",
      "nodeName": "Risk Manager Approval",
      "rejectionCode": "",
      "reasonCode": "rcacmapproverstatusreason",
      "statusCode": "rcacmapproverstatus"
    }
  ],
  "rcaFieldMapping": {
    "crfailreason": "whythechangeimplementationfailed:TEXTAREA",
    "crfailrootcause": "whatistherootcause:TEXTAREA",
    "crfailcorrectiveaction": "whatwasthecorrectiveactiontaken:TEXTAREA",
    "crfailpreventivemeasure": "anypreventivemeasuretoavoidthis:TEXTAREA",
    "crfailedforappcode": "impactedapplication:SELECT",
    "targetisssuefixdate": "whenisthetargetdateofcompletion:DATE",
    "relatedincidentfromcrfail": "anyincidentticketraisedduetothe:TEXTAREA",
    "rcaincidentsummary": "IncidentSummary:TEXTAREA",
    "failedcrcategory": "whatisthefailedchangecategor:SELECT_MANY",
    "rcaapprover1": "appmanagerapproval:MEMBER",
    "rcastatus": "RCAStatus:SELECT"
  },
  "dictMapping": {
    "applicationImpacte": {
      "tableCode": "ic_appcode",
      "idCode": "id",
      "labelCode": "app_id"
    },
    "otherApplicationImpacted": {
      "tableCode": "ic_appcode",
      "idCode": "id",
      "labelCode": "app_id"
    }
  }
}
