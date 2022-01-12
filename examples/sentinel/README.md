# Sentinel Example

This example shows how to write cost policies with HashiCorp's [Sentinel](https://www.hashicorp.com/sentinel). For simplicity, this example evaluates policies with the Sentinel CLI (a.k.a. Sentinel Simulator). The point of this example is to show how a policy could be written against the Infracost JSON format, not how to run Sentinel, since that's tied to HashiCorp's cloud platform.

When the policy checks pass, the Azure Devops pipeline step called "Check Policies" passes and outputs `Policy check passed.` in the task logs. When the policy checks fail, that step fails and the task logs show the Sentinel output indicating failing policies.

Create a policy file (e.g. `policy.policy) that checks a global parameter 'breakdown' containing the Infracost JSON: 
```policy
import "strings"

limitTotalDiff = rule {
    float(breakdown.totalMonthlyCost) < 1500
}

awsInstances = filter breakdown.projects[0].breakdown.resources as _, resource {
	strings.split(resource.name, ".")[0] is "aws_instance"
}

limitInstanceCost = rule {
  all awsInstances as _, instance {
  	float(instance.hourlyCost) <= 2.00
  }
}

instanceBaseCost = func(instance) {
  cost = 0.0
 	for instance.costComponents as cc {
    cost += float(cc.hourlyCost)
  }
  return cost
}

instanceIOPSCost = func(instance) {
  cost = 0.0
 	for instance.subresources as sr {
    for sr.costComponents as cc {
      if cc.name == "Provisioned IOPS" {
        cost += float(cc.hourlyCost)
      }
    }
  }
  return cost
}

limitInstanceIOPSCost = rule {
   all awsInstances as _, instance {
  	instanceIOPSCost(instance) <= instanceBaseCost(instance)
  }
}

main = rule {
    limitTotalDiff and
    limitInstanceCost and
    limitInstanceIOPSCost
}
```

Then use Sentinel to test infrastructure cost changes against the policy.

[//]: <> (BEGIN EXAMPLE)
```yml
pr:
  - master

jobs:
  - job: sentinel
    displayName: Sentinel
    pool: 
      vmImage: ubuntu-latest

    steps:
      - task: InfracostSetup@0
        displayName: Setup Infracost
        inputs:
          apiKey: $(apiKey)
          
      - bash: |
          INSTALL_LOCATION="$(Build.SourcesDirectory)/sentinel"
          mkdir -p ${INSTALL_LOCATION}

          curl -o sentinel.zip https://releases.hashicorp.com/sentinel/0.18.4/sentinel_0.18.4_linux_amd64.zip
          unzip -d ${INSTALL_LOCATION} sentinel.zip
          
          echo "##vso[task.setvariable variable=PATH;]$(PATH):${INSTALL_LOCATION}"
            
        displayName: Setup Sentinel

      - bash: infracost breakdown --path=examples/sentinel/code/plan.json --format=json --out-file=/tmp/infracost.json
        displayName: Run Infracost

      - bash: sentinel apply -global breakdown="$(cat /tmp/infracost.json)" examples/sentinel/policy/policy.policy | tee /tmp/sentinel.out
        displayName: Run Sentinel

      - bash: |
          result=$(</tmp/sentinel.out)
          
          if [ "$result" != "Pass - policy.policy" ]; then
            echo "##vso[task.logissue type=error]Policy check failed"
            echo "##vso[task.logissue type=error]$result"
            echo "##vso[task.complete result=Failed;]DONE"
          else
            echo "Policy check passed."
          fi
        displayName: Check Policies
```
[//]: <> (END EXAMPLE)