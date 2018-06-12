# Building Concourse Windows Worker Environment

To build .Net Applications on Concourse, the Concourse environment should have Windows Workers registered with it. This document will provide the detailed steps in creating a Concourse environment with Windows Workers registered. You may pick any IaaS of choice. However this documentation uses Microsoft Azure as IaaS.

# Create a BOSH CLI ready Unbuntu jumpbox on Azure

Complete the below steps to create an Ubuntu jumpbox to install BOSH Director. 

1. Create a resourcegroup in the Azure Subscription.

2. Install "Ubuntu Server 16.04 LTS" from Azure market place in the resourcegroup and SSH into the jumpbox from terminal. 

3. Download the BOSH CLI v2 using the below commands and move it to the path.

    ```sh
    sudo wget https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.28-linux-amd64
    sudo chmod +x bosh-cli-*
    sudo mv bosh-cli-* /usr/local/bin/bosh
    ```

4. Clone the bosh-deployment using the below command

    ```sh
    git clone https://github.com/cloudfoundry/bosh-deployment
    ```

5. Install Ruby and other required development tools as these are required to build the BOSH Deployment code from git and create the BOSH environment. 

    ```sh
    sudo apt-get install ruby-full
    sudo apt-get install -y build-essential zlibc zlib1g-dev ruby ruby-dev openssl libxslt-dev libxml2-dev libssl-dev libreadline6 libreadline6-dev libyaml-dev libsqlite3-dev sqlite3
    ```

# Create a BOSH Director on Azure

Now that we have a working Ubuntu jumpbox on the IaaS of choice (Azure in this specific case), the next step is to create an operational BOSH Director on the IaaS. The Ubuntu jumpbox will be used to execute all the commands which are part of the BOSH Director installation.

1. Create a container named "stemcell" in the Azure storage account to be used by the BOSH director.

2. Execute the below command in the terminal to create the BOSH Director

    ```sh
    bosh2 create-env bosh-deployment/bosh.yml --state=state.json --vars-store=creds.yml -o bosh-deployment/azure/cpi.yml -v director_name=bosh-concourse -v internal_cidr=10.0.1.0/24 -v internal_gw=10.0.1.1 -v internal_ip=10.0.1.6 -v vnet_name=<vnetname> -v subnet_name=<subnetname> -v subscription_id=<subscriptionid> -v tenant_id=<tenantid> -v client_id=<clientid> -v client_secret=<clientsecret> -v resource_group_name=<resourcegroupname> -v storage_account_name=<storageaccountname> -v default_security_group=<networksecuritygroup>
    ```

3. The default cloud-config.yml in the "bosh-deployment/azure" folder will have place holders. These have to be replaced by appropriate values an execute the below command in the terminal.

    ```sh
    bosh2 interpolate cloud-config.yml  -v internal_cidr=10.0.1.0/24 -v internal_gw=10.0.1.1 -v vnet_name=<vnetname> -v subnet_name=<subnetname> -v security_group=<nsgname> >> cloud-config-interpolated.yml
    ```

4. Update the cloud-config on the BOSH Director by executing the below command in the terminal.

    ```sh
    bosh2 -e bosh-concourse update-cloud-config cloud-config-interpolated.yml 
    ```

5. Check if the correct cloud-config is applied. 

    ```sh
    bosh2 -e bosh-concourse cloud-config
    ```

    [cloud-config-sample] 


# Create a BOSH managed Concourse on Azure

BOSH Director requires the stemcells and the releases to be uploaded to it, before it can be used for the Concourse deployment. The steps for creating a BOSH managed Concourse are as below.

1. Upload Stemcells - As we are setting up a Concourse Windows Worker environment, we will be required to upload Windows Stemcells. However for the Concourse Windows Workers to work, we need to have the base Concourse setup and that involves several Linux VMs based out of the Linux Stemcells. As a result we will be uploading both the Linux and Window Stemcells to create the Concourse Windows Worker environment.

    Linux stemcell upload 
    ```sh
    bosh2 -e bosh-concourse upload-stemcell https://bosh.io/d/stemcells/bosh-azure-hyperv-ubuntu-trusty-go_agent
    ```

    Windows stemcell upload
    ```sh
    bosh2 -e bosh-concourse upload-stemcell https://s3.us-east-2.amazonaws.com/bosh-windows-stemcells-production/light-bosh-stemcell-1200.3-azure-hyperv-windows2012R2-go_agent.tgz
    ```

    Note: The above mentioned stemcells are Azure specific. If you are using any other IaaS, download the corresponding stemcells from [boshio]

2. Verify if the stemcells were uploaded successfully by executing the below command in the terminal.

    ```sh
    bosh2 -e bosh-concourse stemcells
    ```

    You should be able to see your stemcells listed in the output.

    ```sh
    Name                                      Version  OS             CPI  CID                                                       
    bosh-azure-hyperv-ubuntu-trusty-go_agent  3445.7   ubuntu-trusty  -    bosh-stemcell-5f5860b4-a450-4563-99a7-8c2bb6ea88df        
    bosh-azure-hyperv-windows2012R2-go_agent  1200.3   windows2012R2  -    bosh-light-stemcell-3c6eac20-9bc8-434b-a93f-67aa5bd5db2a  

    (*) Currently deployed

    2 stemcells

    Succeeded
    ```

3. Upload Releases - The regular Concourse with Linux workers require both the concourse release and the garden-runc container release. In addition to those, windows worker specific concourse is also required as we intend to support windows workers in this environment. All three releases will be uploaded in this step.

    Garden container release upload
    ```sh
    bosh2  -e bosh-concourse upload-release https://github.com/concourse/concourse/releases/download/v3.1.1/garden-runc-1.6.0.tgz
    ```

    Regular concourse release upload
    ```sh
    bosh2  -e bosh-concourse upload-release https://github.com/concourse/concourse/releases/download/v3.1.1/concourse-3.1.1.tgz
    ```

    Windows Worker concourse release upload
    ```sh
    bosh2  -e bosh-concourse upload-release https://github.com/pivotal-cf-experimental/concourse-windows-worker-release/releases/download/3.4.1/release.tgz
    ```

4. Verify if the releases were uploaded successfully by executing the below command in the terminal.

    ```sh
    bosh2 -e bosh-concourse releases
    ```

    You should be able to see your releases listed in the output.

    ```sh
    Using environment '10.0.1.6' as client 'admin'

    Name                      Version  Commit Hash  
    concourse                 3.1.1    37e6254      
    concourse-windows-worker  3.4.1    8d71e2f      
    garden-runc               1.6.0    cc2f4478     

    (*) Currently deployed
    (+) Uncommitted changes

    3 releases

    Succeeded
    ```

5. TSA is the only entry point for any of the Concourse workers. The workers are expected to get registered with the concourse using the TSA. In the case of the default Linux workers for Concourse, they use the bosh links and do not require any explicit registration with the Concourse. However at the time of this writing, the Windows Workers do not support bosh links. So it is required to explicitly register the Windows Workers with the TSA using each others private and public keys.

    Generate the keys for TSA and Windows Workers using the below commands.

    ```sh
    ssh-keygen -f tsakey -t rsa -N ''
    ssh-keygen -f workerkey -t rsa -N ''
    ```

    Refer to [winworkers] for further details.

6. Create a concourse manifest similar to the one in the below sample. 

    Sample [concourse-manifest] 

7. Execute the below command to deploy Concourse.

    ```sh
    bosh2 -e bosh-concourse -d concourse deploy concourse.yml
    ```
8. Enabling external access for concourse web UI

    At the time of deploying concourse, you might not have the external IP address. So might have to give some random IP address. However after you have deployed the concourse, it would have created a “web” VM and associated it to a nic. The nic would have the internal IP address of the web VM attached to it. But it would not have any external IP. 

    So create an external IP in Azure in the same resource group and associate it to the nic which is liked to web VM. When you complete this process, your external IP will be displayed adamant the Public IP address you have created. Update the concourse.yml with this value in external_url property and redeploy concourse using the same command as before.

    ```sh
    bosh2 -e bosh-concourse -d concourse deploy concourse.yml
    ```

9. Ensure that you have enabled inbound access for port 8080 on the Network Security Group associated with the concourse web VM.

10. Access the Concourse from your browser using the url http://<<external-ip-of-concourse-web-vm>:8080. UserId and Password can be found in the [concourse-manifest].

# Deploy Windows Workers for BOSH Managed Concourse

1. Create a concourse windows worker manifest similar to the one in the below sample.

    Sample [concourse-windows-manifest]

2. Execute the below command to deploy the Concourse Windows Workers and register them with the Concourse.

    ```sh
    bosh2 -e bosh-concourse -d concourse-windows deploy concourse-windows.yml
    ```
# Verify installation

1. Login to the Concourse installation using the below command

    ```sh
    fly -t bosh-concourse-on-azure login --concourse-url  http://52.183.78.215:8080
    ```

    You should see output as shown below in the terminal.

    ```sh
    logging in to team 'main'

    username: admin
    password: 

    target saved
    ```

2. Verify if all the linux and windows workers are registered with the concourse by executing the below command

    ```sh
    fly -t bosh-concourse-on-azure workers
    ```

    You should see output similar to the one below if you have configured the same number of workers as in the sample.

    ```sh
    name                                  containers  platform  tags  team  state    version
    0c8e6731-b277-482e-91ec-6e2b2e1a0a7b  8           linux     none  none  running  1.1    
    dbqr70r7te02r2a                       0           windows   none  none  running  1.2    
    dbqr70sq1vj2r24                       0           windows   none  none  running  1.2    
    dbqr70tj6t92r22                       0           windows   none  none  running  1.2    
    ```

# Troubleshooting 

1. Viewing the vms under the Concourse environment.

    ```sh
    bosh2 -e bosh-concourse vms
    ```

    Sample output

    ```sh
    Using environment '10.0.1.6' as client 'admin'

    Task 227. Done

    Deployment 'concourse'

    Instance                                     Process State  AZ  IPs         VM CID                                                                                                                             VM Type  
    db/c07eb7af-984e-42be-8ae7-7dcbd868bb94      running        z1  10.0.1.4    agent_id:96fcb622-6292-45dc-9cbe-374334209400;resource_group_name:srajaramconcourse;storage_account_name:srajaramconcoursestorage  default  
    web/72e92e2d-1dc7-44f5-b7f8-2f0d037130b9     running        z1  10.0.1.115  agent_id:e5cb60ba-626a-4deb-a603-c4e11c4e4c4c;resource_group_name:srajaramconcourse;storage_account_name:srajaramconcoursestorage  default  
    worker/0c8e6731-4dea-4c94-b834-6869cdc823ab  running        z1  10.0.1.5    agent_id:7d583b40-b277-482e-91ec-6e2b2e1a0a7b;resource_group_name:srajaramconcourse;storage_account_name:srajaramconcoursestorage  large    

    3 vms

    Task 228. Done

    Deployment 'concourse-windows'

    Instance                                             Process State  AZ  IPs       VM CID                                                                                                                             VM Type  
    windows_worker/184f1fc2-e96b-4be0-81e3-a7e854a67832  running        z1  10.0.1.7  agent_id:0fc61c4c-0437-4a43-b681-f53a87e6ab83;resource_group_name:srajaramconcourse;storage_account_name:srajaramconcoursestorage  large    
    windows_worker/24b04405-5cef-4a17-89e4-6be02676f3fe  running        z1  10.0.1.8  agent_id:33e4b974-1064-45fe-b1e5-fa9d9c178939;resource_group_name:srajaramconcourse;storage_account_name:srajaramconcoursestorage  large    
    windows_worker/c81690c5-45ee-4131-96c8-6d53a63bc7a1  running        z1  10.0.1.9  agent_id:2cef21d1-118d-420b-9ffe-305c8eaeb418;resource_group_name:srajaramconcourse;storage_account_name:srajaramconcoursestorage  large    

    3 vms

    Succeeded
    ```

2. You may obtain the logs from workers using the below command. The machine name is obtained using the vms command above.

    ```sh
    bosh2 -e bosh-concourse -d concourse-windows logs windows_worker/184f1fc2-e96b-4be0-81e3-a7e854a67832
    ```

3. SSH into workers using the below command. However at the time of this writing, ssh into windows workers was not supported.

    ```sh
    bosh2 -e bosh-concourse -d concourse-windows ssh web/72e92e2d-1dc7-44f5-b7f8-2f0d037130b9
    ```

4. If there is any issue in the workers getting registered to the Concourse, first verify the web logs and then the logs in the workers.

5. If for any reason you want to delete a deployment and redeploy it, use the below command.

    ```sh
    bosh2 -e bosh-concourse delete-deployment -d concourse-windows
    ```

    Refer to previous steps for deploying.

6. Verifying the status of a deployment.

    ```sh
    bosh2 -e bosh-concourse -d concourse-windows cloud-check
    ```

    Sample output.

    ```sh
    Using environment '10.0.1.6' as client 'admin'

    Using deployment 'concourse-windows'

    Task 229

    20:40:26 | Scanning 3 VMs: Checking VM states (00:00:01)
    20:40:27 | Scanning 3 VMs: 3 OK, 0 unresponsive, 0 missing, 0 unbound (00:00:00)
    20:40:27 | Scanning 0 persistent disks: Looking for inactive disks (00:00:00)
    20:40:27 | Scanning 0 persistent disks: 0 OK, 0 missing, 0 inactive, 0 mount-info mismatch (00:00:00)

    Started  Tue Sep 19 20:40:26 UTC 2017
    Finished Tue Sep 19 20:40:27 UTC 2017
    Duration 00:00:01

    Task 229 done

    #  Type  Description  

    0 problems

    Succeeded

    ```

References:

"Concourse Windows Workers" [winworkers]

"Dave's Windows Workers for United" [dave-united-windows-workwers]

[winworkers]: http://www.chrisumbel.com/article/windows_worker_to_bosh_deployed_concourse
[dave-united-windows-workwers]: https://github.com/pivotalservices/united/tree/master/platform
[cloud-config-sample]: https://github.com/sreenivasmrpivot/concourse-windows-workers/blob/master/cloud-config.yml
[concourse-manifest]: https://github.com/sreenivasmrpivot/concourse-windows-workers/blob/master/concourse.yml
[concourse-windows-manifest]: https://github.com/sreenivasmrpivot/concourse-windows-workers/blob/master/concourse-windows.yml
[boshio]: https://bosh.io/
[bosh-release-for-windows-workers]: https://github.com/pivotal-cf-experimental/concourse-windows-worker-release
