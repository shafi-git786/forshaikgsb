trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

variables:
  WORKING_DIR: 'example'
  TF_VERSION: '1.4.6'

stages:
- stage: Terraform
  jobs:
  - job: terraform_plan
    displayName: 'Terraform Plan'
    steps:
    - checkout: self

    # - task: UsePythonVersion@0
    #   inputs:
    #     versionSpec: '3.x'
    #     addToPath: true

    - task: Bash@3
      inputs:
        targetType: 'inline'
        script: |
          echo "##vso[task.setvariable variable=PLAN_FILE]${BUILD_SOURCEBRANCHNAME}-dev.tfplan"
          echo "##vso[task.setvariable variable=STATE_FILE]${BUILD_SOURCEBRANCHNAME}.terraform.tfstate"

    - task: AzureCLI@2
      inputs:
        azureSubscription: 'myterradeploy-scn'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          az login --service-principal -u $(servicePrincipalId) -p $(servicePrincipalKey) --tenant $(tenantId)
          az account set --subscription $(subscriptionId)
          echo "Setting environment variables"
          export STORAGE_ACCOUNT_NAME=$(STORAGE_ACCOUNT_NAME)
          export CONTAINER_NAME=$(CONTAINER_NAME)
          export KEY=$(KEY)
          export ACCESS_KEY=$(ACCESS_KEY)

    - task: Bash@3
      inputs:
        targetType: 'inline'
        workingDirectory: '$(WORKING_DIR)'
        script: |
          terraform fmt -recursive

    - task: TerraformInstaller@0
      inputs:
        terraformVersion: $(TF_VERSION)

    - task: TerraformTaskV2@2
      inputs:
        provider: 'azurerm'
        command: 'init'
        workingDirectory: '$(WORKING_DIR)'
        backendServiceArm: 'myterradeploy-scn'
        backendAzureRmResourceGroupName: 'terra-rg'
        backendAzureRmStorageAccountName: 'teststgacnt6789backend'
        backendAzureRmContainerName: 'container-bknd'
        backendAzureRmKey: '$(STATE_FILE)'

    - task: AzurePowerShell@5
      inputs:
        azureSubscription: 'myterradeploy-scn'
        ScriptType: 'InlineScript'
        Inline: 'Get-AzContext'
        azurePowerShellVersion: 'LatestVersion'

    - task: TerraformTaskV2@2
      inputs:
        provider: 'azurerm'
        command: 'validate'
        workingDirectory: '$(WORKING_DIR)'

    - task: TerraformTaskV2@2
      inputs:
        provider: 'azurerm'
        command: 'plan'
        workingDirectory: '$(WORKING_DIR)'
        commandOptions: '-out=$(PLAN_FILE)'
        environmentServiceNameAzureRM: 'myterradeploy-scn'

    - task: TerraformTaskV2@2
      displayName: 'Terraform Apply'
      condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
      inputs:
        provider: 'azurerm'
        command: 'apply'
        workingDirectory: '$(WORKING_DIR)'
        commandOptions: '-auto-approve $(PLAN_FILE)'
        environmentServiceNameAzureRM: 'myterradeploy-scn'

        ######################################
        yamltrigger:
- main
- develop  # Optional: triggers on your development branch too

pool:
  vmImage: 'ubuntu-latest'

variables:
  WORKING_DIR: 'example'
  TF_VERSION: '1.4.6'

stages:

# ==========================================
# 1. DEVELOPMENT ENVIRONMENT STAGE
# ==========================================
- stage: Deploy_Dev
  displayName: 'Deploy to Dev'
  jobs:
  - job: dev_deployment
    displayName: 'Dev Plan and Apply'
    steps:
    - checkout: self

    - task: Bash@3
      displayName: 'Set Dev File Names'
      inputs:
        targetType: 'inline'
        script: |
          echo "##vso[task.setvariable variable=PLAN_FILE]dev-${BUILD_SOURCEBRANCHNAME}.tfplan"
          echo "##vso[task.setvariable variable=STATE_FILE]dev-${BUILD_SOURCEBRANCHNAME}.terraform.tfstate"

    - task: TerraformInstaller@0
      inputs:
        terraformVersion: $(TF_VERSION)

    - task: TerraformTaskV2@2
      displayName: 'Terraform Init (Dev)'
      inputs:
        provider: 'azurerm'
        command: 'init'
        workingDirectory: '$(WORKING_DIR)'
        backendServiceArm: 'myterradeploy-scn'
        backendAzureRmResourceGroupName: 'terra-rg'
        backendAzureRmStorageAccountName: 'teststgacnt6789backend'
        backendAzureRmContainerName: 'container-bknd'
        backendAzureRmKey: '$(STATE_FILE)' # Separates dev state from prod state

    - task: TerraformTaskV2@2
      displayName: 'Terraform Validate'
      inputs:
        provider: 'azurerm'
        command: 'validate'
        workingDirectory: '$(WORKING_DIR)'

    - task: TerraformTaskV2@2
      displayName: 'Terraform Plan (Dev)'
      inputs:
        provider: 'azurerm'
        command: 'plan'
        workingDirectory: '$(WORKING_DIR)'
        commandOptions: '-var="env=dev" -out=$(PLAN_FILE)' # Passes dev variables
        environmentServiceNameAzureRM: 'myterradeploy-scn'

    - task: TerraformTaskV2@2
      displayName: 'Terraform Apply (Dev)'
      inputs:
        provider: 'azurerm'
        command: 'apply'
        workingDirectory: '$(WORKING_DIR)'
        commandOptions: '-auto-approve $(PLAN_FILE)'
        environmentServiceNameAzureRM: 'myterradeploy-scn'

# ==========================================
# 2. PRODUCTION ENVIRONMENT STAGE
# ==========================================
- stage: Deploy_Prod
  displayName: 'Deploy to Prod'
  dependsOn: Deploy_Dev # Forces Prod to wait for Dev to succeed
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/main') # Only runs on main branch
  jobs:
  - job: prod_deployment
    displayName: 'Prod Plan and Apply'
    steps:
    - checkout: self

    - task: Bash@3
      displayName: 'Set Prod File Names'
      inputs:
        targetType: 'inline'
        script: |
          echo "##vso[task.setvariable variable=PLAN_FILE]prod-${BUILD_SOURCEBRANCHNAME}.tfplan"
          echo "##vso[task.setvariable variable=STATE_FILE]prod-${BUILD_SOURCEBRANCHNAME}.terraform.tfstate"

    - task: TerraformInstaller@0
      inputs:
        terraformVersion: $(TF_VERSION)

    - task: TerraformTaskV2@2
      displayName: 'Terraform Init (Prod)'
      inputs:
        provider: 'azurerm'
        command: 'init'
        workingDirectory: '$(WORKING_DIR)'
        backendServiceArm: 'myterradeploy-scn'
        backendAzureRmResourceGroupName: 'terra-rg'
        backendAzureRmStorageAccountName: 'teststgacnt6789backend'
        backendAzureRmContainerName: 'container-bknd'
        backendAzureRmKey: '$(STATE_FILE)'

    - task: TerraformTaskV2@2
      displayName: 'Terraform Plan (Prod)'
      inputs:
        provider: 'azurerm'
        command: 'plan'
        workingDirectory: '$(WORKING_DIR)'
        commandOptions: '-var="env=prod" -out=$(PLAN_FILE)'
        environmentServiceNameAzureRM: 'myterradeploy-scn'

    - task: TerraformTaskV2@2
      displayName: 'Terraform Apply (Prod)'
      inputs:
        provider: 'azurerm'
        command: 'apply'
        workingDirectory: '$(WORKING_DIR)'
        commandOptions: '-auto-approve $(PLAN_FILE)'
        environmentServiceNameAzureRM: 'myterradeploy-scn'
