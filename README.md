
1) In Target Technology - 5 (Green)
2) In Progress towards Target Technology - 3 (Amber)
3) Disinvest - 1 (Red)
3) If Cat 1 or 2 is dependent on Cat  3 or 4 apps but it is not in critical path.  - 5 (Green)
"
2) If Cat 1 or 2 is dependent on Cat  3 or 4 apps but with risk mitigation  - 3 (Amber)"
1) If Cat 1 or 2 is dependent on Cat  3 or 4 apps - 1 (Red)
1) If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system and downtime is > zero - 1 (Red)
2) If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system and downtime is zero - 5 (Green)
3) If it is Cat 3 or Cat 4 which are not part of DTDT0 (rest of the systems that doesn't fall under #1) - < 30 mins - 5 (Green)
4) If it is Cat 3 or Cat 4 which are not part of DTDT0 (rest of the systems that doesn't fall under #1) - > 30 mins - 3 (Amber)
5) If it is Cat 3 or Cat 4 which are not part of DTDT0 (rest of the systems that doesn't fall under #1) - > 60 mins - 1 (Red)
1) If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system and downtime is > 30 mins - 1 (Red)
2) If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system and downtime is < 30 mins - 5 (Green)
3) If it is Cat 3 or Cat 4 which are not part of DTDT0 (rest of the systems that doesn't fall under #1) - < 60 mins - 5 (Green)
4) If it is Cat 3 or Cat 4 which are not part of DTDT0 (rest of the systems that doesn't fall under #1) - > 60 mins and less than < 120 mins - 3 (Amber)
5) If it is Cat 3 or Cat 4 which are not part of DTDT0 (rest of the systems that doesn't fall under #1) - > 120 mins - 1 (Red)
1) If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system and downtime has no alternative pathways - 1 (Red)
"2) If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system and downtime has alternative pathways which can be enabled in > 15 mins - 5 (Green)
"
"3) If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system and downtime has alternative pathways which can be enabled in < 15 mins and less than or equal to 30 mins - 3 (Amber)
"
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Automated recovery for infrastructure failures (node) in less than 10 mins - Green (5)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Manual recovery for infrastructure failures (node) and takes more than 10 mins - Red (1)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Data Centre failure (manual) - less than 1 hour - Green (5)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Data Centre failure (manual) - greater than 1 hour and less than 2 hours - Amber (3)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Data Centre failure (manual) - greater than  2 hours - Red (3)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Data Corruption due to system or human error Recovery - RPO Near Zero and RTO less than 1 hour (Manual) - Green (5)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Data Corruption due to system or human error Recovery - RPO Near Zero and RTO more than 1 hour (Manual) - Red (1)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Data Corruption due to ransomeware attack Recovery - RPO Near Zero and RTO less than 4 hours (Manual) - Green (5)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Data Corruption due to ransomeware attack Recovery - RPO Near Zero and RTO greater than 4 hours and less than 10 hours (Manual) - Amber (3)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Data Corruption due to ransomeware attack Recovery - RPO Near Zero and RTO more than 10 hours (Manual) - Red (1)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Control plane corruption recovery is less than 30 mins - Green (5)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Control plane corruption recovery is less than 60 mins but more than 30 mins - Amber (3)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then Control plane corruption recovery is more than 60 mins - Red (1)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then In-memory Cache out of sync or corrupted is recovered in less than 30 mins - Green (5)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then In-memory Cache out of sync or corrupted is recovered in less than 60 mins but more than 30 mins - Amber (3)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then In-memory Cache out of sync or corrupted is recovered in more than 60 mins - Red (1)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then 3rd Party failure - less than 10 mins - Green (5)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then 3rd Party failure - more than 10 mins and less than 30 mins - Amber (3)
If it is a Cat 1 or 2 or MAS644 or CII or DTDT0 system - checked automated recovery by technology layer - Compute, Database, Storage, Clustered Software, Network, Security, then 3rd Party failure - more than 30 mins - Red (1)
2) For Rest of the Cat 3 applications - as per DR guideline -  RTO & RPO - 8 hours RTO and Near Zero RPO for abover parameters. Flag only manual recovery scenarios for infrastructure failures as Red. If Recovery Time is more than 1 hour for non-data corruption / non-control plane corruption / non-DC failure scenarios , flag it is as Red. Rest can be green.
2) For Rest of the Cat 3 applications - as per DR guideline -  RTO & RPO - 8 hours RTO and Near Zero RPO for abover parameters. Flag only manual recovery scenarios for infrastructure failures as Red. If Recovery Time is more than 1 hour for non-data corruption / non-control plane corruption / non-DC failure scenarios , flag it is as Red. Rest can be green.
2) For Rest of the Cat 3 applications - as per DR guideline -  RTO & RPO - 8 hours RTO and Near Zero RPO for abover parameters. Flag only manual recovery scenarios for infrastructure failures as Red. If Recovery Time is more than 1 hour for non-data corruption / non-control plane corruption / non-DC failure scenarios , flag it is as Red. Rest can be green.
3) Cat 4 Applications - as per DR guideline -  RTO & RPO -  48 hours RTO and Near Zero RPO for abover parameters. Flag only manual recovery scenarios for infrastructure failures as Red.  If Recovery Time is more than 2 hours for non-data corruption / non-control plane corruption / non-DC failure scenarios , flag it is as Red. Rest can be green.
3) Cat 4 Applications - as per DR guideline -  RTO & RPO -  48 hours RTO and Near Zero RPO for abover parameters. Flag only manual recovery scenarios for infrastructure failures as Red.  If Recovery Time is more than 2 hours for non-data corruption / non-control plane corruption / non-DC failure scenarios , flag it is as Red. Rest can be green.
3) Cat 4 Applications - as per DR guideline -  RTO & RPO -  48 hours RTO and Near Zero RPO for abover parameters. Flag only manual recovery scenarios for infrastructure failures as Red.  If Recovery Time is more than 2 hours for non-data corruption / non-control plane corruption / non-DC failure scenarios , flag it is as Red. Rest can be green.
"App category is NOT important here. Cumulate numbers for each technology non-compliance. Assuming 5 technologies are used, 25 is a perfect score. Anything below, we need to categorise based on overall score. 
1) Buy - Green (5)
2) Hold - Amber (3)
3) Sell - Red (1) 
4) Grey - don't count them as they are not part of COE"
"Cumulate numbers for each pattern non-compliance. Assuming 5 patterns are used, 25 is a perfect score. Anything below, we need to categorise based on overall score. 
1) Compliant - Green (5)
2) Non-Compliant - Red (1) 
3)Grey - don't count them as they are not part of COE"
TPS , Performance Testing, Choas Testing - Design Vs Validation Result. This can be populated during ADR phase only. We'll exclude this scoring during ARC Review during design phase. Later, we can sync up by calling ADR system.
"Sherpa validate that actual build is conforming to design. This is again during ADR phase. We need to discuss the design.
1) Fully Compliant - 5 (Green)
2) Partially Compliant - 3 (Amber)
3) Non-Compliant - 1 (Red)"
"Affirm hardware/software are tracked for patch and hardening
Affirm PID is lodged
Affirm asset is registered in inventory
RDU Approval feeding to ARC"
