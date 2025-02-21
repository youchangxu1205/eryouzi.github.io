

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
        IDR_SIGNOFF_PROCUTOVER("IDR Signoff", null,null ,null , "idrsignoffurl"),

        ISS_SIGNOFF("ISS Signoff", "issoverduepatchapproverstatus", "issoverduepatchapprover", "issoverduepatchapproverrejectioncode",null );
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
  "pushUrl":              "https://itsm-dev.cmwf.ocp.uat.dbs.com/common-server/ichamp/createCR",
  "updateUrl":            "https://itsm-dev.cmwf.ocp.uat.dbs.com/common-server/ichamp/updateCR",
  "pushMethod":           "POST",
  "createNodeId":         "UserTask_13ikwht",
  "closeNodeId":          "UserTask_1gehich",
  "crStatusFieldCode":    "crStatus",
  "crStatusDictCode":     "crStatus",
  "closeStatusFieldCode": "CloseStatus",
  "signoffNodeId":        "UserTask_0da3d8t",
  "implementationNodeId": "AcceptanceTask_0qba8sk",
  "fieldMapping":         {
    "changerequestorgroups":                                             "changeRequestorGroups",
    "countryoforigin":                                                   "countryOfOrigin",
    "lob":                                                               "lob",
    "changetype":                                                        "changeType",
    "changegroup":                                                       "changeGroup",
    "changenature":                                                      "changeNature",
    "cmcmascategory":                                                    "MASCategory",
    "location":                                                          "location",
    "countryimpacted":                                                   "CountryImpacte",
    "applicationimpacted":                                               "applicationImpacte",
    "otherapplicationimpacted":                                          "otherApplicationImpacted",
    "changecategory":                                                    "Change_Category",
    "mainappimpacted":                                                   "mainApplicationRequiringChange",
    "servicesimpactedandrecoveryplan":                                   "Listtheimpactedbythechange",
    "lpar":                                                              "lpar",
    "mainframecrtype":                                                   "mainframeCRType",
    "codechecker":                                                       "Code_Checker",
    "sicode":                                                            "standardCode",
    "majorchangeflag":                                                   "IsMajorChange",
    "appresiliencyclassification":                                       "appResiliencyClassification",
    "crclassification":                                                  "crClassification",
    "implementergroup":                                                  "implementergroup",
    "schedulestartdatetime,scheduleenddatetime":                         "ChangeSchedule_StartEndTime",
    "implementationplanschedulestart,implementationplanscheduleenddate": "Implementation_Time",
    "reversionplanschedulestartdate,reversionplanscheduleenddate":       "Reversion_Time",
    "downtimestartdatetime,downtimeenddatetime":                         "ScheduledDowntime_Time",
    "scheduledmaintenancestartdatetime,scheduledmaintenanceenddatetime": "ScheduledMaintenance_Time",
    "downtimeapprover1_,downtimeapprover1_name":                         "ScheduledApprover1",
    "downtimeapprover2_,downtimeapprover2_name":                         "ScheduledApprover2",
    "downtimeapprover3_,downtimeapprover3_name":                         "ScheduledApprover3",
    "downtimeapprover4_,downtimeapprover4_name":                         "ScheduledApprover4",
    "downtimeapprover5_,downtimeapprover5_name":                         "ScheduledApprover5",
    "summary":                                                           "changeSummary",
    "description":                                                       "changeDescribtion",
    "implementationplan":                                                "implementationPlan",
    "reversionplan":                                                     "reversionPlan",
    "releaseticketslist":                                                "DroneTicket",
    "liveverification":                                                  "liveVerificationAfterImplementat",
    "uat":                                                               "uat",
    "regressiontesting":                                                 "regressiontesting",
    "rollbacktesting":                                                   "ReversionBackoutRollbackTesting",
    "relatedincident":                                                   "relatedIncident",
    "parentchange":                                                      "parentChange",
    "applicationownername":                                              "ApplicationOwner",
    "mainframepackagename":                                              "mainframePackageName",
    "mainframepackagetype":                                              "MainframePackage_Type",
    "mainframepackagecreatortsoid":                                      "mainframePackageCreatorTSOID",
    "mainframepackagecreator1bankid":                                    "mainframePackageCreator1BankID",
    "holdbatchjob":                                                      "Holdbatch_job",
    "jobtype":                                                           "jobType",
    "designfordatachoice":                                               "IsthisCRinscope_forD4D",
    "designfordataowner":                                                "d4dDataOwner",
    "designfordatarequirements":                                         "dataRequirementsDetails",
    "ticketprojectmanager":                                              "projectCutOverProjectManager",
    "ticketprojectmanagermobile":                                        "projectCutOverProjectManagerCont",
    "projectobjective":                                                  "projectCutOverProjectObjective",
    "projectscope":                                                      "projectCutOverProjectScope",
    "cyberarkobjects":                                                   "CyberArk_Object",
    "cmcappcodechange":                                                  "AnyapplicationCode",
    "cmcmakerchecker":                                                   "MakerChecker",
    "cmcmaker":                                                          "maker",
    "cmcchecker":                                                        "checker",
    "cmcrollbackduration":                                               "rollbackDurationInMinutes",
    "cmcdatapatchnumberofrecord":                                        "dataPatchNumberOfRecords",
    "approvergroup1":                                                    "ApproverGroup1",
    "approver1":                                                         "Approver01",
    "approvergroup2":                                                    "ApproverGroup2",
    "approver2":                                                         "Approver02",
    "approvergroup3":                                                    "ApproverGroup3",
    "approver3":                                                         "Approver03",
    "approvergroup4":                                                    "ApproverGroupfourr",
    "approver4":                                                         "Approver04",
    "approvergroup5":                                                    "ApproverGroupfive",
    "approver5":                                                         "Approver05",
    "approvergroup6":                                                    "ApproverGroupsix",
    "approver6":                                                         "Approver06",
    "mdapprovergroup":                                                   "TechMDApproverGroup",
    "mdapprover":                                                        "Tech_MDApprover",
    "releasemanagergroup":                                               "ReleaseManagerGroup",
    "changemanagergroup":                                                "ChangeManagerGroup_one",
    "crfailreason":                                                      "whythechangeimplementationfailed",
    "crfailrootcause":                                                   "whatistherootcause",
    "crfailcorrectiveaction":                                            "whatwasthecorrectiveactiontaken",
    "crfailpreventivemeasure":                                           "anypreventivemeasuretoavoidthis",
    "crfailedforappcode":                                                "impactedapplication",
    "targetisssuefixdate":                                               "whenisthetargetdateofcompletion",
    "relatedincidentfromcrfail":                                         "anyincidentticketraisedduetothe",
    "rcaincidentsummary":                                                "IncidentSummary",
    "failedcrcategory":                                                  "whatisthefailedchangecategor",
    "deploymentapproach":                                                "Deployment_Approach",
    "reversionapproach":                                                 "Reversion_Approach",
    "resourceengagement":                                                "Resource_Engagement",
    "potentialblastradius":                                              "blastRadiusDescription",
    "changeduringonlinebusinesshours":                                   "changeDescriptionDuringBusinessH",
    "mainframecriticalmonthendbatchimpact":                              "Mainframeritica",
    "majorinterdependencies":                                            "interdependenciesDescription",
    "drcapabilitiesimpact":                                              "DRCapabilitiesImpact",
    "upstreamdownstreaminterfacesimpact":                                "interfaceImpact",
    "jobneedtoaccessonlinefiles":                                        "JobNeedToAccessOnlineFiles",
    "closeonlinefilesduringjobrun":                                      "CloseOnlineFilesDuringJobRun",
    "usehighcpuincreaseworkload":                                        "UseHighCPUIncreaseWorkload",
    "riskscore":                                                         "riskScore",
    "featurevalues":                                                     "featureValues",
    "recommendedrisklevel":                                              "recommendedRisk",
    "explainability":                                                    "explainability",
    "explainabilitybase":                                                "explainabilityBase",
    "explainabilityscores":                                              "explainabilityScores",
    "riskthreshold":                                                     "riskThreshold",
    "diffcatselectedfrmaimlreason":                                      "JustificationwhyCategory",
    "securityriskjustification":                                         "SecurityRiskJustification",
    "alldocsandartifactsurl":                                            "URLForProjectDocsandArtefac",
    "cmrmavailablity":                                                   "CustomerServices",
    "cmrminherentresidualrisks":                                         "InherentandResidualrisks",
    "cmrmcontinuity":                                                    "BusinessUnitsOperations",
    "cmrmcomplexity":                                                    "ChangeComplexity",
    "cmrmimplement":                                                     "BackoutComplexity",
    "cmrmtraining":                                                      "Documentation",
    "cmrmsecurity":                                                      "Security",
    "cmrminterfaces":                                                    "Interfaces",
    "closedincompletereasoncode":                                        "ClosedLogSummary",
    "l1changemanagerapprovername":                                       "L1Change_Manager",
    "riskmanagerapprovername":                                           "L1Risk_Manager",
    "issoverduepatchapprovalgroup":                                      "ISSGroup",
    "servicemonitoringsnoctype":                                         "SNOCsignoff",
    "cussignofftype":                                                    "CUSsignoff",
    "idrsignofftype":                                                    "IDRsignoff",
    "servicemonitoringsnocappservicelist":                               "NewApplicationServices",
    "servicemonitoringsnocurllist":                                      "TheProductionDashboardLinks",
    "ticketfocalpoint":                                                  "ticketfocalpoint",
    "ticketfocalpointmobile":                                            "ticketfocalpointmobile",
    "mccrequesterlogin":                                                 "mccrequesterlogin",
    "designfordataownername":                                            "d4dDataOwner",
    "impactriskidentified":                                              "identifiedImpactAndRisks",
    "riskmitigated":                                                     "riskMitigationPlan",
    "d4dsignofftype":                                                    "D4Dsignoff",
    "impacttomainframetype":                                             "ImpactToMainframe",
    "datacentersignofftype":                                             "DataCenterOPSsignoff",
    "mddelegatesignofftype":                                             "MDDelegateSignOff",
    "hasignofftype":                                                     "HADRFlipsignoff",
    "designfordatajustification":                                        "justificationReason",
    "changecount":                                                       "ChangeCountForMainframe",
    "cmcmajorchange":                                                    "MajorChanges",
    "state":                                                             "crStatus"
  },
  "riskDictMapping":      {
    "CustomerServices":         "d7b6d17cf02c401a93144a6323915745",
    "InherentandResidualrisks": "f831cb01fd0f43b297a93ead68abc562",
    "BusinessUnitsOperations":  "2008e9b96d5d413798d119b829648535",
    "ChangeComplexity":         "845a783c49524348bd5804cfbc0ac06e",
    "BackoutComplexity":        "c7199dacb70e4a07ad29dd898cf56860",
    "Documentation":            "ff30d63471b14bc3870b90c849c66f7d",
    "Security":                 "3240ef33f1bd4dbcbb1ad5ac86bb0b53",
    "Interfaces":               "aea98f47781147f8a8cb8eae078f7299"
  },
  "tableFieldMapping":    {
    "mainframedata": [
      {
        "ichampTableFieldInfo": "Job Details",
        "itsmTableFieldInfo":   "JobDetailsSection",
        "fieldMapping":         {
          "jobname":     "JobDetails",
          "jobdatetime": "DateTimeModified",
          "jobduration": "Duration"
        }
      },
      {
        "ichampTableFieldInfo": "Load Module",
        "itsmTableFieldInfo":   "LoadModuleDetailsSection",
        "fieldMapping":         {
          "modulecompiled": "LoadModuleCompiled",
          "moduledatetime": "DateTimeCompiled"
        }
      },
      {
        "ichampTableFieldInfo": "DBRM Members",
        "itsmTableFieldInfo":   "DBRMMemberDetailsSection",
        "fieldMapping":         {
          "dbrmmember": "DBRMMember"
        }
      },
      {
        "ichampTableFieldInfo": "Input Data",
        "itsmTableFieldInfo":   "InputDataDetailsSection",
        "fieldMapping":         {
          "inputdata":         "InputData",
          "inputdatadatetime": "DateTimeLastModified"
        }
      }
    ]
  },
  "approveInfoMapping":   [
    {
      "nodeId":        "SingleApprove_0wb3mla",
      "rejectionCode": "approver1rejectioncode",
      "reasonCode":    "approver1statusreason",
      "statusCode":    "approver1status"
    },
    {
      "nodeId":        "SingleApprove_1y2bas2",
      "rejectionCode": "approver2rejectioncode",
      "reasonCode":    "approver2statusreason",
      "statusCode":    "approver2status"
    },
    {
      "nodeId":        "SingleApprove_09y6ddj",
      "rejectionCode": "approver3rejectioncode",
      "reasonCode":    "approver3statusreason",
      "statusCode":    "approver3status"
    },
    {
      "nodeId":        "SingleApprove_112fawy",
      "rejectionCode": "approver4rejectioncode",
      "reasonCode":    "approver4statusreason",
      "statusCode":    "approver4status"
    },
    {
      "nodeId":        "SingleApprove_01zjgji",
      "rejectionCode": "approver5rejectioncode",
      "reasonCode":    "approver5statusreason",
      "statusCode":    "approver5status"
    },
    {
      "nodeId":        "SingleApprove_0fh11tl",
      "rejectionCode": "approver6rejectioncode",
      "reasonCode":    "approver6statusreason",
      "statusCode":    "approver6status"
    },
    {
      "nodeId":        "SingleApprove_0cntz02",
      "rejectionCode": "",
      "reasonCode":    "applicationownercomment",
      "statusCode":    "applicationownerapprovalstatus"
    },
    {
      "nodeId":        "SingleApprove_1tkhamt",
      "rejectionCode": "",
      "reasonCode":    "designfordatarejectioncomment",
      "statusCode":    "designfordataapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_10vwuti",
      "rejectionCode": "l1changemanagerapproverrejectioncode",
      "reasonCode":    "l1changemanagerstatusreason",
      "statusCode":    "l1changemanagerapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_1kslrrp",
      "rejectionCode": "l1changemanagerapproverrejectioncode",
      "reasonCode":    "l1changemanagerstatusreason",
      "statusCode":    "l1changemanagerapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_1dv8pri",
      "rejectionCode": "riskmanagerapproverrejectioncode",
      "reasonCode":    "riskmanagerstatusreason",
      "statusCode":    "riskmanagerapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_13119ha",
      "rejectionCode": "rmapproverrejectioncode",
      "reasonCode":    "rmapproverrejectionreason",
      "statusCode":    "rmapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_0fjvryb",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment1",
      "statusCode":    "downtimeapproverstatus1"
    },
    {
      "nodeId":        "SingleApprove_0uvdhk1",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment2",
      "statusCode":    "downtimeapproverstatus2"
    },
    {
      "nodeId":        "SingleApprove_0wv160x",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment3",
      "statusCode":    "downtimeapproverstatus3"
    },
    {
      "nodeId":        "SingleApprove_1lir1bh",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment4",
      "statusCode":    "downtimeapproverstatus4"
    },
    {
      "nodeId":        "SingleApprove_0zy9gcq",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment5",
      "statusCode":    "downtimeapproverstatus5"
    },
    {
      "nodeId":        "SingleApprove_0armfga",
      "rejectionCode": "mdapproverrejectioncode",
      "reasonCode":    "mdstatusreason",
      "statusCode":    "mdapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_01kn76b",
      "rejectionCode": "changemanagerrejectioncode",
      "reasonCode":    "changemanagerstatusreason",
      "statusCode":    "changemanagerapproverstatus"
    }
  ],
  "rcaRiskMapping":       {
    "crfailreason":              "whythechangeimplementationfailed",
    "crfailrootcause":           "whatistherootcause",
    "crfailcorrectiveaction":    "whatwasthecorrectiveactiontaken",
    "crfailpreventivemeasure":   "anypreventivemeasuretoavoidthis",
    "crfailedforappcode":        "impactedapplication",
    "targetisssuefixdate":       "whenisthetargetdateofcompletion",
    "relatedincidentfromcrfail": "anyincidentticketraisedduetothe",
    "rcaincidentsummary":        "IncidentSummary",
    "failedcrcategory":          "whatisthefailedchangecategor"
  }
}
