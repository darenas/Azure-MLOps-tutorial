# Tutorial for setting up a MLOps pipeline in Azure 

# I. Preparations:
- Create an azure account
- Create a new DevOps project 
- If you dont have an Azure subscription, create a new Free subscription
- If you dont have a ML workspace, create one 
- Configure the 'Service Principle' on the ML workspace, several ways to do this, for instance, using the 'cloud shell' (console button on https://portal.azure.com) execute the following command and save the results (these are the service principle credentials)
		
    ```$ az ad sp create-for-rbac --name myFirstMLOpsProject_sp```  (or choose another name)

- Add a service principle to the DevOps project:
	- Go to the 'Project settings' under the main DevOps project page
	- Select the 'Service connections' on the menu, select 'Create a new service connection', select 'Azure Ressource Manager' and select 'Using service principal (manual):
		- Scope level: subscription
		- Subscription Id: <your-subscription-id> (can be obtained from the 'Overview' of your ML workspace)
		- Subscription Name: <your-subscription-name> (can be obtained from the 'Overview' of your ML workspace)
		- Service Principal Id: <the-contents-of-"appId"-from-the-service-principle-credentials-obtained-before>
		- Credential: Service principal key
		- Service principal key: <the-contents-of-"password"-from-the-service-principle-credentials-obtained-before>
		- Tennant ID: <the-contents-of-"tenant"-from-the-service-principle-credentials-obtained-before>
		- Service connection name: myAzureConnection
		- Grant access permission to all pipelines: True (checked)
		- Click on 'Verify' to validate, a 'succesful verification' is necessary to continue.

# II. The code:
- Prepare your code in a proper repository 
- Data can be added to a blob storage at this stage, or it can be also added during the pipeline
- Your train script must contain the necessary code for being executed in the Azure ML environment (check example)
- Do not forget to add the necessary files that specify the environment for your code to work (requirements, conda environment, etc)


# III. Building the MLOps pipeline:
- Access to your MLOps project
- Import your repo using the link (or create a new one and upload the files manually) (menu Repos)
- Create and configure the CI Pipeline (see below for details)
- Create and configure the CD pipeline (see below for details)
- Adding tasks to a pipeline:
	- Click on the '+' button (Add task to agent) on the desired Agent (by default 'Agent job 1')
	- Search for the desired task type and click 'Add'
	- Click on the newly created task and modify parameters (good practice: at least personalize the Name so it can be more descriptive)	
- Adding variables to a pipeline:
	- On the 'Variables' tab of the pipeline select 'Pipeline Variables'
	- Click the 'Add +' button, then input: <variable-name> | <variable-value>


## III.A. CI Pipeline

1. Create a new CI Pipeline (Pipelines >> Pipelines >> New Pipeline):
	- Select 'Use Classic editor' 
	- Select 'Azure Repos Git', then the Project, the Repository and the Branch 
	- Select 'Start with an Empty Job'
	- On the 'Tasks' tab, 'Pipeline' subtab, select the host specification (VM that will run the pipeline), e.g., set it to Ubuntu 18.04
	- (optional) On the 'Triggers' tab, 'Continuous Integration' subtab, check the 'Enable Continuous Integration' box and configure as desired, e.g., Type: Include Branch: master
	- Several agents may run in the same pipeline, for the moment we will use only one, the next step is to add a task for this agent

2. Add the following variables to the pipeline:
	```azureml.resourceGroup | <resource-group-name>
    azureml.workspaceName | <workspace-name>
    azureml.location | westeurope
    amlcompute.clusterName	| <cluster-name> 
    amlcompute.vmSize | STANDARD_DS2_V2  (this specify the type of VM, personalize according to your needs)
    amlcompute.minNodes | 0 (so it costs nothing while not being used)
    amlcompute.maxNodes | 2 (modify according to your needs)
    amlcompute.idleSecondsBeforeScaledown | 300 (modify according to your needs)
    amlexperiment.name | iris_mbm_exp_01
    model.name | iris_mbm_model

3. Use Python Version: Add task of type 'Use Python version'
	- Personalize name and select the desired version of python (e.g. 3.6)
	
4. Setup python environment for the current pipeline: Add task of type 'Bash'
	- Personalize name  
	- Fill the 'script path' with `setup/pipeline_install_requirements.sh` (use path or the browse button)
	- Set 'Working directory' (under 'Advanced' section) to point to the 'setup' folder
	
5. Add azure ML extension: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 
        `az extension add -n azure-cli-ml`

6. Create Azure ML Service Workspace: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 
        
        `az ml workspace create -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -l $(azureml.location) --exist-ok --yes`


7. Create Azure ML Compute Cluster: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 
		
        `az ml computetarget create amlcompute -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(amlcompute.clusterName) -s $(amlcompute.vmSize) --min-nodes $(amlcompute.minNodes) --max-nodes $(amlcompute.maxNodes) --idle-seconds-before-scaledown $(amlcompute.idleSecondsBeforeScaledown)`
		
8. Upload data to the default Datastore:  Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put:
    
        `az ml datastore upload -w $(azureml.workspaceName) -g $(azureml.resourceGroup) -n $(az ml datastore show-default -w $(azureml.workspaceName) -g $(azureml.resourceGroup) --query name -o tsv) -p data -u iris --overwrite true`

9. Prepare folders to store metadata and the trained models: Add task of type 'Bash'
	- Personalize name
	- Select 'Inline Script' as type and fill with:
		
        `mkdir metadata && mkdir models`

10. Train the model: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 
		
        `az ml run submit-script -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -e $(amlexperiment.name) --ct $(amlcompute.clusterName) -c mbm_training --path setup -t metadata/run.json --source-directory src/models/mybasicmodel/ mbm_training.py --container_name iris --input_csv iris.csv --model_filename $(model.name).save.pkl  --dataset_name iris_ds --dataset_desc "IRIS Data Set"`

11. Register the saved model: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 
    
        `az ml model register -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(model.name) --asset-path outputs/models/$(model.name).save.pkl -d "My Basic Model for Iris dataset" --tag "data"="iris" --tag "model"="Decision Tree" --model-framework ScikitLearn -f metadata/run.json -t metadata/model.json`

12. Download model from the Model Registry into the Pipeline VM: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 	
		
        `az ml model download -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -i $(jq -r .modelId metadata/model.json) -t models --overwrite`
		
13. Copy Files to the Artifact Staging Directory: Add task of type 'Copy Files'
	- Personalize name
	- In 'source folder' put:
		`$(Build.SourcesDirectory)`		
	- In 'contents' put:
		```**/metadata/*
		**/setup/*
		**/models/*
		**/deployment/*
		**/tests/integration/*```
	- In 'target folder' put:
		`$(Build.ArtifactStagingDirectory)`

14. Publish artifacts: Add task of type 'Publish Pipeline Artifacts'
	- Personalize name
	- In 'File or Directory path' put:
		`$(Build.ArtifactStagingDirectory)`
	- In 'Artifact name'
		`my_mbm_artifact` (or some other Name for the artifact)

## III.B CD Pipeline 
It contains 2 stages: Pre-prod then Prod

### Pre-prod Stage

1. Create a new CD pipeline (Pipelines >> Releases >> New Pipeline):
	- Select 'Start with an Empty Job'
	- Put the Stage name, e.g., 'Deploy to Pre-Production'
	- Click on 'Add an Artifact'
	- Select 'Build' in 'Source type'
	- Select the name of your project and the build pipeline (CI). The other fields may be left by default (latest version and proposed alias) 
	- Click on the 'Tasks' tab
	- Click on the 'Agent job' and then set the Agent pool, e.g., Ubuntu 18.04
	- Add tasks (see the details for the CD Pipeline tasks)

2. Add the following variables to the pipeline:
	```azureml.resourceGroup | <resource-group-name>
    azureml.workspaceName | <workspace-name>
    service.name.staging | iris-mbm-service-aci (or any other name for the service)
	service.name.production | iris-mbm-service-aks (or any other name for the service)```

3. Use Python Version: Add task of type 'Use Python version'
	- Personalize name and select the desired version of python (e.g. 3.6)  
	
4. Add azure ML extension: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 

		`az extension add -n azure-cli-ml`

5. Deploy to a ML service as an ACI (Azure Container Instance): Add task of type 'Azure CLI'   
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Click 'Browse Working Directory' and select the 'deployement' folder within the artifacts of the CI pipeline
	- Select inline script and put:  

		`az ml model deploy -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.staging) -f ../metadata/model.json --dc aciDeploymentConfig.yml --ic inferenceConfig.yml --overwrite`

6. Setup python environment for the current pipeline : Add task of type 'Bash'
	- Personalize name  
	- Fill the 'script path' with `setup/pipeline_install_requirements.sh` (use path or the browse button)
	- Set 'Working directory' (under 'Advanced' section) to point to the 'setup' folder

7. Perform Integration Tests: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Click 'Browse Working Directory' and select the 'integration' subfolder within the artifacts of the CI pipeline 
	- Select inline script and put: 

		`pytest mbm_integration_test.py --doctest-modules --junitxml=junit/TEST-results.xml --cov=mbm_integration_test --cov-report=xml --cov-report=html --scoreurl $(az ml service show -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.staging) --query scoringUri -o tsv)`
		
8. Publish Test Results: Add task of type 'Publish Test Results'
	- Personalize name
	- Set 'Test Result Format' to 'JUnit'  (should be the default option) 
	- Set 'Test Result Files' to '**/TEST-*.xml'  (should be the default option)
	- Set the 'Test run title' to 'Integration Tests (Staging)'
	
### Prod Stage 
(NOTE: This will stage uses AKS for deployement, which is not available to free accounts. If you are using a free account, this stage will fail to work)

0. Create a new stage for the Release Pipeline
	- Click on the '+ Add' under the 'Deploy to Pre-Production stage
	- Click on 'Start with an empty job'
	- Name the stage to 'Deploy to Production'
	- Click on the 'Tasks' tab
	- Click on the 'Agent job' and then set the Agent pool, e.g., Ubuntu 18.04
	- Add tasks (see the details for the CD Pipeline tasks)

1. Add the following variables to the pipeline:
	aks.clusterName | myAKSclust
	aks.vmSize | Standard_B4ms
	aks.agentCount | 3

2. Use Python Version: Add task of type 'Use Python version'
	- Personalize name and select the desired version of python (e.g. 3.6)

3. Add azure ML extension: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 
		`az extension add -n azure-cli-ml`
		
4. Create a Kubernetes Service: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Select inline script and put: 
	 
5. Deploy to a AKS (Azure Kubernetes Service): Add task of type 'Azure CLI'    
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Click 'Browse Working Directory' and select the 'deployement' folder within the artifacts of the CI pipeline
	- Select inline script and put:  
		`az ml model deploy -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.production) -f ../metadata/model.json --dc aksDeploymentConfig.yml --ic inferenceConfig.yml --ct $(aks.clusterName) --overwrite`

6. Setup python environment for the current pipeline : Add task of type 'Bash'
	- Personalize name  
	- Fill the 'script path' with 'setup/pipeline_install_requirements.sh' (use path or the browse button)
	- Set 'Working directory' (under 'Advanced' section) to point to the 'setup' folder

7. Perform Integration Tests: Add task of type 'Azure CLI'
	- Personalize name
	- Select the Azure subscription previously created (e.g., myAzureConnection)
	- Select 'Shell' as 'Script Type'
	- Click 'Browse Working Directory' and select the 'integration' subfolder within the artifacts of the CI pipeline 
	- Select inline script and put: 
		`pytest mbm_integration_test.py --doctest-modules --junitxml=junit/TEST-results.xml --cov=mbm_integration_test --cov-report=xml --cov-report=html --scoreurl $(az ml service show -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.production) --query scoringUri -o tsv) --scorekey $(az ml service get-keys -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.production) --query primaryKey -o tsv)`
		

8. Publish Test Results: Add task of type 'Publish Test Results'
	- Personalize name
	- Set 'Test Result Format' to 'JUnit'  (should be the default option) 
	- Set 'Test Result Files' to '**/TEST-*.xml'  (should be the default option)
	- Set the 'Test run title' to 'Integration Tests (Staging)'

9. (optional) Add a validation step betweeen Preproduction and Production stages
	- Add a 'Pre-deployement Condition' to the 'Deploy to production' stage by clicking on the correspondig button in the whole CD pipeline view ('Pipeline' tab)
	- Activate the 'Pre-deployement approval'
	- In the 'Approvers' field add the persons that should approuve the pass to production (for the example yourself)


 
### IV. Credits
This tutorial is a compilation from several tutorials:
- MLOps on Azure (https://github.com/microsoft/MLOps)
- MLOps with Azure ML (https://github.com/Microsoft/MLOpsPython)
- MLOps Lab (https://github.com/SaschaDittmann/MLOps-Lab)
- Azure MLOps Talk (https://www.youtube.com/watch?v=pLd7xF0z5Zs)

