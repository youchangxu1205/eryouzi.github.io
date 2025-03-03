{
  "pushUrl":               "https://itsm-dev.cmwf.ocp.uat.dbs.com/common-server/ichamp/createCR",
  "updateUrl":             "https://itsm-dev.cmwf.ocp.uat.dbs.com/common-server/ichamp/updateCR",
  "pushMethod":            "POST",
  "createNodeId":          "UserTask_13ikwht",
  "closeNodeId":           "UserTask_1gehich",
  "crStatusFieldCode":     "crStatus",
  "crStatusDictCode":      "crStatus",
  "closeStatusFieldCode":  "CloseStatus",
  "signoffNodeId":         "UserTask_0da3d8t",
  "implementationNodeId":  "AcceptanceTask_0qba8sk",
  "fieldMapping": {
    "summary":                                                           "changeSummary:TEXTAREA",
    "description":                                                       "changeDescribtion:TEXTAREA",
    "changerequestorgroups":                                             "changeRequestorGroups:GROUP",
    "countryoforigin":                                                   "countryOfOrigin:SELECT",
    "lob":                                                               "lob:SELECT",
    "changetype":                                                        "changeType:SELECT",
    "changenature":                                                      "changeNature:SELECT",
    "changegroup":                                                       "changeGroup:SELECT",
    "cmcdatapatchnumberofrecord":                                        "dataPatchNumberOfRecords:NUMBER",
    "mainframecrtype":                                                   "mainframeCRType:SELECT",
    "crclassification":                                                  "crClassification:SELECT",
    "location":                                                          "location:SELECT",
    "countryimpacted":                                                   "CountryImpacte:SELECT_MANY",
    "applicationimpacted":                                               "applicationImpacte:SELECT_MANY",
    "otherapplicationimpacted":                                          "otherApplicationImpacted:SELECT_MANY",
    "mainappimpacted":                                                   "mainApplicationRequiringChange:SELECT",
    "changecategory":                                                    "Change_Category:INPUT",
    "appresiliencyclassification":                                       "appResiliencyClassification:SELECT",
    "lpar":                                                              "lpar:SELECT",
    "servicesimpactedandrecoveryplan":                                   "Listtheimpactedbythechange:TEXTAREA",
    "riskscore":                                                         "riskScore:INPUT",
    "featurevalues":                                                     "featureValues:INPUT",
    "recommendedrisklevel":                                              "recommendedRisk:INPUT",
    "explainability":                                                    "explainability:INPUT",
    "explainabilitybase":                                                "explainabilityBase:INPUT",
    "explainabilityscores":                                              "explainabilityScores:INPUT",
    "riskthreshold":                                                     "riskThreshold:INPUT",
    "diffcatselectedfrmaimlreason":                                      "JustificationwhyCategory:TEXTAREA",
    "cmrmavailablity":                                                   "CustomerServices:SELECT",
    "cmrminherentresidualrisks":                                         "InherentandResidualrisks:SELECT",
    "cmrmcontinuity":                                                    "BusinessUnitsOperations:SELECT",
    "cmrmcomplexity":                                                    "ChangeComplexity:SELECT",
    "cmrmimplement":                                                     "BackoutComplexity:SELECT",
    "cmrmtraining":                                                      "Documentation:SELECT",
    "cmrmsecurity":                                                      "Security:SELECT",
    "cmrminterfaces":                                                    "Interfaces:SELECT",
    "schedulestartdatetime,scheduleenddatetime":                         "ChangeSchedule_StartEndTime:DATE_RANGE",
    "implementationplanschedulestart,implementationplanscheduleenddate": "Implementation_Time:DATE_RANGE",
    "reversionplanschedulestartdate,reversionplanscheduleenddate":       "Reversion_Time:DATE_RANGE",
    "downtimestartdatetime,downtimeenddatetime":                         "ScheduledDowntime_Time:DATE_RANGE",
    "scheduledmaintenancestartdatetime,scheduledmaintenanceenddatetime": "ScheduledMaintenance_Time:DATE_RANGE",
    "downtimeapprover1_,downtimeapprover1_name":                         "ScheduledApprover1:MEMBER",
    "downtimeapprover2_,downtimeapprover2_name":                         "ScheduledApprover2:MEMBER",
    "downtimeapprover3_,downtimeapprover3_name":                         "ScheduledApprover3:MEMBER",
    "downtimeapprover4_,downtimeapprover4_name":                         "ScheduledApprover4:MEMBER",
    "downtimeapprover5_,downtimeapprover5_name":                         "ScheduledApprover5:MEMBER",
    "releaseticketslist":                                                "DroneTicket:TEXTAREA",
    "liveverification":                                                  "liveVerificationAfterImplementat:SELECT",
    "uat":                                                               "uat:SELECT",
    "regressiontesting":                                                 "regressiontesting:SELECT",
    "rollbacktesting":                                                   "ReversionBackoutRollbackTesting:SELECT",
    "relatedincident":                                                   "relatedIncident:INPUT",
    "parentchange":                                                      "parentChange:INPUT",
    "designfordatachoice":                                               "IsthisCRinscope_forD4D:SELECT",
    "designfordataowner,designfordataownername":                         "d4dDataOwner:MEMBER",
    "designfordatarequirements":                                         "dataRequirementsDetails:TEXTAREA",
    "designfordatajustification":                                        "justificationReason:SELECT",
    "cyberarkobjects":                                                   "CyberArk_Object:TEXTAREA",
    "cmcappcodechange":                                                  "AnyapplicationCode:SELECT",
    "cmcmakerchecker":                                                   "MakerChecker:RADIO",
    "cmcmaker":                                                          "maker:MEMBER",
    "cmcchecker":                                                        "checker:MEMBER",
    "mainframepackagename":                                              "mainframePackageName:INPUT",
    "mainframepackagetype":                                              "MainframePackage_Type:SELECT",
    "mainframepackagecreatortsoid":                                      "mainframePackageCreatorTSOID:INPUT",
    "mainframepackagecreator1bankid":                                    "mainframePackageCreator1BankID:INPUT",
    "holdbatchjob":                                                      "Holdbatch_job::RADIO",
    "jobtype":                                                           "jobType:SELECT",
    "jobneedtoaccessonlinefiles":                                        "JobNeedToAccessOnlineFiles:SELECT",
    "closeonlinefilesduringjobrun":                                      "CloseOnlineFilesDuringJobRun:SELECT",
    "usehighcpuincreaseworkload":                                        "UseHighCPUIncreaseWorkload:SELECT",
    "impactriskidentified":                                              "identifiedImpactAndRisks:SELECT",
    "riskmitigated":                                                     "riskMitigationPlan:SELECT",
    "deploymentapproach":                                                "Deployment_Approach:SELECT_MANY",
    "reversionapproach":                                                 "Reversion_Approach:SELECT_MANY",
    "resourceengagement":                                                "Resource_Engagement:SELECT_MANY",
    "potentialblastradius":                                              "blastRadiusDescription:SELECT",
    "changeduringonlinebusinesshours":                                   "changeDescriptionDuringBusinessH:SELECT",
    "mainframecriticalmonthendbatchimpact":                              "Mainframeritica:SELECT",
    "majorinterdependencies":                                            "interdependenciesDescription:SELECT",
    "drcapabilitiesimpact":                                              "DRCapabilitiesImpact:SELECT",
    "upstreamdownstreaminterfacesimpact":                                "interfaceImpact:SELECT",
    "implementergroup":                                                  "implementergroup:GROUP",
    "hasignofftype":                                                     "HADRFlipsignoff:SELECT",
    "servicemonitoringsnocurllist":                                      "TheProductionDashboardLinks:TEXTAREA",
    "servicemonitoringsnocappservicelist":                               "NewApplicationServices:TEXTAREA",
    "ticketprojectmanager":                                              "projectCutOverProjectManager:MEMBER",
    "ticketprojectmanagermobile":                                        "projectCutOverProjectManagerCont:INPUT",
    "projectobjective":                                                  "projectCutOverProjectObjective:INPUT",
    "projectscope":                                                      "projectCutOverProjectScope:INPUT",
    "alldocsandartifactsurl":                                            "URLForProjectDocsandArtefac:TEXTAREA",
    "mddelegatesignofftype":                                             "MDDelegateSignOff:SELECT",
    "datacentersignofftype":                                             "DataCenterOPSsignoff:SELECT",
    "impacttomainframetype":                                             "ImpactToMainframe:SELECT",
    "d4dsignofftype":                                                    "D4Dsignoff:SELECT",
    "servicemonitoringsnoctype":                                         "SNOCsignoff:SELECT",
    "cussignofftype":                                                    "CUSsignoff:SELECT",
    "idrsignofftype":                                                    "IDRsignoff:SELECT",
    "approvergroup1":                                                    "ApproverGroup1:GROUP",
    "approver1":                                                         "Approver01:MEMBER",
    "approvergroup2":                                                    "ApproverGroup2:GROUP",
    "approver2":                                                         "Approver02:MEMBER",
    "approvergroup3":                                                    "ApproverGroup3:GROUP",
    "approver3":                                                         "Approver03:MEMBER",
    "approvergroup4":                                                    "ApproverGroupfourr:GROUP",
    "approver4":                                                         "Approver04:MEMBER",
    "approvergroup5":                                                    "ApproverGroupfive:GROUP",
    "approver5":                                                         "Approver05:MEMBER",
    "approvergroup6":                                                    "ApproverGroupsix:GROUP",
    "approver6":                                                         "Approver06:MEMBER",
    "mdapprovergroup":                                                   "TechMDApproverGroup:GROUP",
    "mdapprover":                                                        "Tech_MDApprover:MEMBER",
    "releasemanagergroup":                                               "ReleaseManagerGroup:GROUP",
    "changemanagergroup":                                                "ChangeManagerGroup_one:GROUP",
    "l1changemanagerapprovername":                                       "L1Change_Manager:MEMBER",
    "riskmanagerapprovername":                                           "L1Risk_Manager:MEMBER",
    "applicationownername":                                              "ApplicationOwner:MEMBER",
    "issoverduepatchapprovergroup":                                      "ISSGroup:GROUP",
    "closedincompletereasoncode":                                        "ClosedLogSummary:SELECT",
    "cmcmascategory":                                                    "MASCategory:SELECT",
    "changeriskupgradeddowngraded":                                      "ChangeRiskUpgradedDowngraded:INPUT",
    "crsubmissiondatetime":                                              "CRSubmissionDatetime:DATE",
    "codechecker":                                                       "Code_Checker:SELECT",
    "sicode":                                                            "standardCode:INPUT",
    "majorchangeflag":                                                   "IsMajorChange:RADIO",
    "implementationplan":                                                "implementationPlan:CR_URL",
    "reversionplan":                                                     "reversionPlan:CR_URL",
    "cmcrollbackduration":                                               "rollbackDurationInMinutes:INPUT",
    "securityriskjustification":                                         "SecurityRiskJustification:TEXTAREA",
    "ticketfocalpoint":                                                  "ticketfocalpoint:INPUT",
    "ticketfocalpointmobile":                                            "ticketfocalpointmobile:INPUT",
    "mccrequesterlogin":                                                 "mccrequesterlogin:MEMBER",
    "changecount":                                                       "ChangeCountForMainframe:NUMBER",
    "cmcmajorchange":                                                    "MajorChanges:SELECT_MANY",
    "state":                                                             "crStatus:SELECT"
  },
  "riskDictMapping":       {
    "CustomerServices":         "d7b6d17cf02c401a93144a6323915745",
    "InherentandResidualrisks": "f831cb01fd0f43b297a93ead68abc562",
    "BusinessUnitsOperations":  "2008e9b96d5d413798d119b829648535",
    "ChangeComplexity":         "845a783c49524348bd5804cfbc0ac06e",
    "BackoutComplexity":        "c7199dacb70e4a07ad29dd898cf56860",
    "Documentation":            "ff30d63471b14bc3870b90c849c66f7d",
    "Security":                 "3240ef33f1bd4dbcbb1ad5ac86bb0b53",
    "Interfaces":               "aea98f47781147f8a8cb8eae078f7299"
  },
  "tableFieldMapping":     {
    "mainframedata": [
      {
        "ichampTableFieldInfo": "Job Details",
        "itsmTableFieldInfo":   "JobDetailsSection",
        "fieldMapping":         {
          "jobname":     "JobDetails:INPUT",
          "jobdatetime": "DateTimeModified:DATE",
          "jobduration": "Duration:NUMBER"
        }
      },
      {
        "ichampTableFieldInfo": "Load Module",
        "itsmTableFieldInfo":   "LoadModuleDetailsSection",
        "fieldMapping":         {
          "modulecompiled": "LoadModuleCompiled:INPUT",
          "moduledatetime": "DateTimeCompiled:DATE"
        }
      },
      {
        "ichampTableFieldInfo": "DBRM Members",
        "itsmTableFieldInfo":   "DBRMMemberDetailsSection",
        "fieldMapping":         {
          "dbrmmember": "DBRMMember:INPUT"
        }
      },
      {
        "ichampTableFieldInfo": "Input Data",
        "itsmTableFieldInfo":   "InputDataDetailsSection",
        "fieldMapping":         {
          "inputdata":         "InputData:INPUT",
          "inputdatadatetime": "DateTimeLastModified:DATE"
        }
      }
    ]
  },
  "approveInfoMapping":    [
    {
      "nodeId":        "SingleApprove_0wb3mla",
      "nodeName":      "App Govenance1",
      "rejectionCode": "approver1rejectioncode",
      "reasonCode":    "approver1statusreason",
      "statusCode":    "approver1status"
    },
    {
      "nodeId":        "SingleApprove_1y2bas2",
      "nodeName":      "App Govenance2",
      "rejectionCode": "approver2rejectioncode",
      "reasonCode":    "approver2statusreason",
      "statusCode":    "approver2status"
    },
    {
      "nodeId":        "SingleApprove_09y6ddj",
      "nodeName":      "App Govenance3",
      "rejectionCode": "approver3rejectioncode",
      "reasonCode":    "approver3statusreason",
      "statusCode":    "approver3status"
    },
    {
      "nodeId":        "SingleApprove_112fawy",
      "nodeName":      "App Govenance4",
      "rejectionCode": "approver4rejectioncode",
      "reasonCode":    "approver4statusreason",
      "statusCode":    "approver4status"
    },
    {
      "nodeId":        "SingleApprove_01zjgji",
      "nodeName":      "App Govenance5",
      "rejectionCode": "approver5rejectioncode",
      "reasonCode":    "approver5statusreason",
      "statusCode":    "approver5status"
    },
    {
      "nodeId":        "SingleApprove_0fh11tl",
      "nodeName":      "App Govenance6",
      "rejectionCode": "approver6rejectioncode",
      "reasonCode":    "approver6statusreason",
      "statusCode":    "approver6status"
    },
    {
      "nodeId":         "SingleApprove_0cntz02",
      "nodeName":       "Application Owner (For Data Patch)",
      "rejectionCode":  "",
      "reasonCode":     "applicationownercomment",
      "statusTimeCode": "appownerdatapatchapprovalstatustime",
      "statusCode":     "applicationownerapprovalstatus"
    },
    {
      "nodeId":        "SingleApprove_1tkhamt",
      "nodeName":      "D4D Data Owner",
      "rejectionCode": "",
      "reasonCode":    "designfordatarejectioncomment",
      "statusCode":    "designfordataapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_10vwuti",
      "nodeName":      "ISS Overdue Patch",
      "rejectionCode": "issoverduepatchapproverrejectioncode",
      "reasonCode":    "issoverduepatchapproverstatusreason",
      "statusCode":    "issoverduepatchapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_1kslrrp",
      "nodeName":      "L1 Change Manager",
      "rejectionCode": "l1changemanagerapproverrejectioncode",
      "reasonCode":    "l1changemanagerstatusreason",
      "statusCode":    "l1changemanagerapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_1dv8pri",
      "nodeName":      "L1 Risk Manager",
      "rejectionCode": "riskmanagerapproverrejectioncode",
      "reasonCode":    "riskmanagerstatusreason",
      "statusCode":    "riskmanagerapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_13119ha",
      "nodeName":      "L1.5 Release Manager Team",
      "rejectionCode": "rmapproverrejectioncode",
      "reasonCode":    "rmapproverrejectionreason",
      "statusCode":    "rmapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_0fjvryb",
      "nodeName":      "Scheduled Downtime/Maintenance Approver 1",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment1",
      "statusCode":    "downtimeapproverstatus1"
    },
    {
      "nodeId":        "SingleApprove_0uvdhk1",
      "nodeName":      "Scheduled Downtime/Maintenance Approver 2",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment2",
      "statusCode":    "downtimeapproverstatus2"
    },
    {
      "nodeId":        "SingleApprove_0wv160x",
      "nodeName":      "Scheduled Downtime/Maintenance Approver 3",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment3",
      "statusCode":    "downtimeapproverstatus3"
    },
    {
      "nodeId":        "SingleApprove_1lir1bh",
      "nodeName":      "Scheduled Downtime/Maintenance Approver 4",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment4",
      "statusCode":    "downtimeapproverstatus4"
    },
    {
      "nodeId":        "SingleApprove_0zy9gcq",
      "nodeName":      "Scheduled Downtime/Maintenance Approver 5",
      "rejectionCode": "",
      "reasonCode":    "downtimeapproverrejectioncomment5",
      "statusCode":    "downtimeapproverstatus5"
    },
    {
      "nodeId":        "SingleApprove_0armfga",
      "nodeName":      "Tech MD",
      "rejectionCode": "mdapproverrejectioncode",
      "reasonCode":    "mdstatusreason",
      "statusCode":    "mdapproverstatus"
    },
    {
      "nodeId":        "SingleApprove_01kn76b",
      "nodeName":      "L1.5 Change Management Team",
      "rejectionCode": "changemanagerrejectioncode",
      "reasonCode":    "changemanagerstatusreason",
      "statusCode":    "changemanagerapproverstatus"
    }
  ],
  "rcaApproveInfoMapping": [
    {
      "nodeId":        "SingleApprove_1ojunx3",
      "nodeName":      "App Manager Approval",
      "rejectionCode": "",
      "reasonCode":    "rcaapprover1statusreason",
      "statusCode":    "rcaapprover1status"
    },
    {
      "nodeId":        "SingleApprove_1r1kd6y",
      "nodeName":      "Risk Manager Approval",
      "rejectionCode": "",
      "reasonCode":    "rcacmapproverstatusreason",
      "statusCode":    "rcacmapproverstatus"
    }
  ],
  "rcaFieldMapping":       {
    "crfailreason":              "whythechangeimplementationfailed:TEXTAREA",
    "crfailrootcause":           "whatistherootcause:TEXTAREA",
    "crfailcorrectiveaction":    "whatwasthecorrectiveactiontaken:TEXTAREA",
    "crfailpreventivemeasure":   "anypreventivemeasuretoavoidthis:TEXTAREA",
    "crfailedforappcode":        "impactedapplication:SELECT",
    "targetisssuefixdate":       "whenisthetargetdateofcompletion:DATE",
    "relatedincidentfromcrfail": "anyincidentticketraisedduetothe:TEXTAREA",
    "rcaincidentsummary":        "IncidentSummary:TEXTAREA",
    "failedcrcategory":          "whatisthefailedchangecategor:SELECT_MANY",
    "rcaapprover1":              "appmanagerapproval:MEMBER",
    "rcastatus":                 "RCAStatus:SELECT"
  },
  "dictMapping":           {
    "applicationImpacte":       {
      "tableCode": "ic_appcode",
      "idCode":    "id",
      "labelCode": "app_id"
    },
    "otherApplicationImpacted": {
      "tableCode": "ic_appcode",
      "idCode":    "id",
      "labelCode": "app_id"
    }
  }
}
