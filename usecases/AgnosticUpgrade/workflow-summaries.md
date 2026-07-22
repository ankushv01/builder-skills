
## Agnostic Upgrade
- id: c4a22ed6-59d4-4019-94ae-be64793c7199
- inputs: ['version', 'devices', 'bootMode']
- outputs: ['version', 'devices', 'bootMode', '_id', 'initiator', 'emailMessage', 'reattempt', 'success', 'postCheckCommit', 'preCheckCommit']
- task count: 64
- childJob tasks:
    - [4597] -> @6994dbcc6d90cb331ac67eb3: File Transfer | File Transfer
    - [b9b6] -> @6994dbcc6d90cb331ac67eb3: Run Commands for Checks and Upgrades | Create Command Template of Checks
    - [b2ae] -> @6994dbcc6d90cb331ac67eb3: Get Netbox Device and Context | Get Netbox details
    - [08c9] -> @6994dbcc6d90cb331ac67eb3: Run Commands for Checks and Upgrades | Create Command Template of Rollback
    - [481b] -> @6994dbcc6d90cb331ac67eb3: Run Commands for Checks and Upgrades | Create Command Template of Upgrade
    - [f654] -> @6994dbcc6d90cb331ac67eb3: Run Commands for Checks and Upgrades | Create Command Template of Post Checks
    - [3f4d] -> @6994dbcc6d90cb331ac67eb3: Run Commands for Validations | Pre-check validations
    - [1fdd] -> @6994dbcc6d90cb331ac67eb3: Run Commands for Validations | Post-check validations
    - [13db] -> @698e67953d6a63dd1a8b851a: pushFileWithTextContentToNexus | Push to Nexus
    - [28e2] -> @698e67953d6a63dd1a8b851a: pushFileWithTextContentToNexus | Push to Nexus
    - [a2e1] -> @6994dbcc6d90cb331ac67eb3: File Transfer | File Transfer
- adapter/app tasks:
    - [9c30] isAlive (app=ConfigurationManager, locationType=None) | Check if device is connected
    - [ea5e] renderJinja2TemplateWithCast (app=TemplateBuilder, locationType=None) | Render Jinja2 Template With Data Cast
    - [40c1] backUpDevice (app=ConfigurationManager, locationType=None) | Backup device configuration
- manual tasks (approval/view gates):
    - [9207] ViewDiff | View Diff
- transition count: 73

## Clean Up Template
- id: 121c23c8-30fd-4008-bad8-b89d9482b036
- inputs: ['id']
- outputs: ['id', '_id', 'initiator', 'success']
- task count: 5
- adapter/app tasks:
    - [c58c] MOP.deleteTemplate (app=IAP MOP Command Template API:1.0.0, locationType=IAP MOP Command Template API:1.0.0) | Delete a Command Template
- transition count: 5

## Create Blob
- id: a4ad079d-8d1d-471a-855d-f8084a194431
- inputs: ['command', 'content']
- outputs: ['command', 'content', '_id', 'initiator', 'filePath', 'sha']
- task count: 10
- adapter/app tasks:
    - [72e3] getReposOwnerRepoContentsPath (app=GitHub, locationType=GitHub) | Get contents.
This method returns the contents of a file or directory in a repository.
Files and sym
    - [d3c1] genericAdapterRequest (app=GitHub, locationType=GitHub) | Makes the requested generic call
- transition count: 12

## Create Tree and commit
- id: 9cc97957-d816-425d-a45a-85cfa1c520be
- inputs: ['blobArray']
- outputs: ['blobArray', '_id', 'initiator', 'commitResult']
- task count: 16
- adapter/app tasks:
    - [2230] genericAdapterRequest (app=GitHub, locationType=GitHub) | Makes the requested generic call
    - [6682] genericAdapterRequest (app=GitHub, locationType=GitHub) | Makes the requested generic call
    - [9534] genericAdapterRequest (app=GitHub, locationType=GitHub) | Makes the requested generic call
    - [3da2] postReposOwnerRepoGitTrees (app=GitHub, locationType=GitHub) | Create a Tree.
The tree creation API will take nested entries as well. If both a tree and
a nested p
    - [7a8f] postReposOwnerRepoGitCommits (app=GitHub, locationType=GitHub) | Create a Commit.
- transition count: 15

## Create and Run Command Template
- id: a33634f1-5a02-4dcf-877e-f31ffded1396
- inputs: ['obj', 'version', 'step', 'devices']
- outputs: ['obj', 'version', 'step', 'devices', '_id', 'initiator', 'templateName', 'templateResults', 'success']
- task count: 8
- childJob tasks:
    - [31e8] -> @6994dbcc6d90cb331ac67eb3: DynamicTemplateCreation | Run Child Job
    - [4c86] -> @6994dbcc6d90cb331ac67eb3: Clean Up Template | Run Child Job
- adapter/app tasks:
    - [fc6d] RunCommandTemplate (app=MOP, locationType=None) | Run Command Template
- transition count: 8

## Create and Run Verification Command Template
- id: 4e110285-fbba-494d-aeb0-1c53adbede58
- inputs: ['obj', 'version', 'step', 'devices']
- outputs: ['obj', 'version', 'step', 'devices', '_id', 'initiator', 'templateName', 'templateResults', 'success']
- task count: 7
- childJob tasks:
    - [31e8] -> @6994dbcc6d90cb331ac67eb3: DynamicTemplateCreation with Validations | Run Child Job
    - [898d] -> @6994dbcc6d90cb331ac67eb3: Clean Up Template | Run Child Job
- adapter/app tasks:
    - [fc6d] RunCommandTemplate (app=MOP, locationType=None) | Run Command Template
- transition count: 7

## DynamicTemplateCreation
- id: 74872f50-01cf-4ca0-9b91-281fa22a40fb
- inputs: ['obj', 'version', 'step']
- outputs: ['obj', 'version', 'step', '_id', 'initiator', 'templateName', 'success']
- task count: 12
- adapter/app tasks:
    - [3672] renderJinja2TemplateWithCast (app=TemplateBuilder, locationType=None) | Render Jinja2 Template With Data Cast
    - [ac44] renderJinjaTemplate (app=TemplateBuilder, locationType=None) | Render Jinja Template
    - [2a00] MOP.importTemplate (app=IAP MOP Command Template API:1.0.0, locationType=IAP MOP Command Template API:1.0.0) | Import Template
- transition count: 12

## DynamicTemplateCreation with Validations
- id: 73aaefed-d37b-4016-a209-5a1579ef713f
- inputs: ['obj', 'version', 'step']
- outputs: ['obj', 'version', 'step', '_id', 'initiator', 'templateName', 'success']
- task count: 9
- adapter/app tasks:
    - [3672] renderJinja2TemplateWithCast (app=TemplateBuilder, locationType=None) | Render Jinja2 Template With Data Cast
    - [4316] MOP.importTemplate (app=IAP MOP Command Template API:1.0.0, locationType=IAP MOP Command Template API:1.0.0) | Import Template
- transition count: 9

## Execute Upgrade Steps
- id: 45611fcc-6d45-43e9-b520-0144abc667d2
- inputs: ['pathTofile', 'fileName']
- outputs: ['pathTofile', 'fileName', '_id', 'initiator']
- task count: 7
- childJob tasks:
    - [3575] -> @6994dbcc6d90cb331ac67eb3: getGithubFile | Get Steps File from Github
    - [e9a2] -> @6994dbcc6d90cb331ac67eb3: Run Steps with File Contents | Run steps with File Contents
- transition count: 6

## Execute Upgrade Steps TAD
- id: 96ae2cce-a1c7-4286-ac26-c2e8479f9ca8
- inputs: ['pathTofile', 'fileName']
- outputs: ['pathTofile', 'fileName', '_id', 'initiator', 'fileContents']
- task count: 6
- childJob tasks:
    - [3575] -> @6994dbcc6d90cb331ac67eb3: getGithubFile | Get Steps File from Github
- transition count: 5

## Fetch CRQ Details
- id: 798283e5-cdf0-4b78-b0c1-089c40df3e19
- inputs: ['Infrastructure_Change_Id']
- outputs: ['Infrastructure_Change_Id', '_id', 'initiator', 'devices', 'success']
- task count: 9
- childJob tasks:
    - [a278] -> @6882500f543cc2c6724d7bd7: Get_CRQ_CI_List | Get_CRQ_CI_Device(s)
    - [b3fe] -> @6882500f543cc2c6724d7bd7: Get_CRQ_Status | Get_CRQ_Status
- transition count: 9

## File Transfer
- id: d88ecc9e-05d8-4e2e-87ea-a14f48a0764d
- inputs: ['image', 'version', 'devices', 'transferObject']
- outputs: ['image', 'version', 'devices', 'transferObject', '_id', 'initiator', 'skipTransferResults', 'transferResults', 'transferSuccess']
- task count: 16
- childJob tasks:
    - [bf79] -> @6994dbcc6d90cb331ac67eb3: Create and Run Command Template | Run Skip Transfer Commands
    - [8c9b] -> @6994dbcc6d90cb331ac67eb3: Create and Run Command Template | Run Verify Presence Commands
- adapter/app tasks:
    - [9ca1] agnostic_load_image.py (app=AGManager, locationType=None) | Shell Script agnostic_load_image.py
    - [a0d9] agnostic_scp.py (app=AGManager, locationType=None) | Shell Script agnostic_scp.py
- transition count: 19

## Get Netbox Device and Context
- id: 90f1de84-6afb-49a7-ae87-e41af86b447d
- inputs: ['version', 'name']
- outputs: ['version', 'name', '_id', 'initiator', 'success', 'check_command_sets', 'diff_ignore_patterns', 'images', 'upgrade_sequence', 'rollback_sequence', 'image_transfer', 'validation_rules', 'plugin']
- task count: 7
- adapter/app tasks:
    - [328c] genericAdapterRequest (app=Netbox, locationType=Netbox) | Makes the requested generic call
    - [da56] getDevice (app=ConfigurationManager, locationType=None) | Get Device Details
- transition count: 8

## Retrieve from GitHub
- id: 258d0f07-f6f4-4d7b-a0d1-56a3afd8ae27
- inputs: []
- outputs: ['_id', 'initiator']
- task count: 11
- childJob tasks:
    - [1818] -> @6984ff4371ab83a64cbf75c9: TAD Transfer File - IOS-XR | Run Child Job
    - [54bf] -> @6984ff4371ab83a64cbf75c9: run command loop | command loop
    - [eafc] -> @6984ff4371ab83a64cbf75c9: Update Version TXT | Run Child Job
- adapter/app tasks:
    - [0831] getReposOwnerRepoContentsPath (app=GitHub, locationType=GitHub) | metadata
    - [6a4f] getReposOwnerRepoContentsPath (app=GitHub, locationType=GitHub) | Show commands
- transition count: 12

## Run Commands for Checks and Upgrades
- id: 3118ab8b-976b-47d0-afc4-d5fdb4b2dd77
- inputs: ['image', 'version', 'step', 'devices', 'object']
- outputs: ['image', 'version', 'step', 'devices', 'object', '_id', 'initiator', 'commandResults']
- task count: 6
- childJob tasks:
    - [bf79] -> @6994dbcc6d90cb331ac67eb3: Create and Run Command Template | Run Commands
- transition count: 5

## Run Commands for Validations
- id: 6be1b700-82f5-40c2-a14a-5e57e2ef91ce
- inputs: ['obj', 'version', 'step', 'devices', 'image']
- outputs: ['obj', 'version', 'step', 'devices', 'image', '_id', 'initiator', 'commandResults']
- task count: 5
- childJob tasks:
    - [bf79] -> @6994dbcc6d90cb331ac67eb3: Create and Run Verification Command Template | Run Commands
- transition count: 4

## Run Steps with File Contents
- id: a1809855-bf71-4744-86a5-d7a78dac31e2
- inputs: []
- outputs: ['_id', 'initiator']
- task count: 3
- transition count: 2

## Run Transfer Scripts
- id: c7be8778-bb22-44a4-9c4e-f671553f3b26
- inputs: []
- outputs: ['_id', 'initiator']
- task count: 5
- adapter/app tasks:
    - [18fe] agnostic_load_image.py (app=AGManager, locationType=None) | Shell Script agnostic_load_image.py
    - [e66f] agnostic_scp.py (app=AGManager, locationType=None) | Shell Script agnostic_scp.py
    - [bad7] scp_cisco.py (app=AGManager, locationType=None) | Shell Script scp_cisco.py
- transition count: 4

## Software Upgrade
- id: 695e06de-982e-456c-9c1e-1c4d2b627303
- inputs: ['formData']
- outputs: ['formData', '_id', 'initiator', 'pathToFile']
- task count: 8
- childJob tasks:
    - [1f49] -> @6994dbcc6d90cb331ac67eb3: getGithubFile | Get Manifest File
    - [4ec5] -> @6994dbcc6d90cb331ac67eb3: Execute Upgrade Steps | Execute Upgrade Steps
- transition count: 7

## Software Upgrade TAD
- id: d8cc7b53-f8e7-4ec3-8d1f-10c1e4b956fc
- inputs: ['version']
- outputs: ['version', '_id', 'initiator', 'upgradeSteps', 'manifest']
- task count: 8
- childJob tasks:
    - [1f49] -> @6994dbcc6d90cb331ac67eb3: getGithubFile | Get Manifest File
    - [4ec5] -> @6994dbcc6d90cb331ac67eb3: Execute Upgrade Steps TAD | Execute Upgrade Steps
- transition count: 7

## Take Device Backup
- id: f49e1ea2-d37d-4f6a-b712-f993498ac676
- inputs: ['name']
- outputs: ['name', '_id', 'initiator']
- task count: 3
- adapter/app tasks:
    - [8498] backUpDevice (app=ConfigurationManager, locationType=None) | Backup device configuration
- transition count: 2

## Update Version TXT
- id: 9e310a06-fb92-4644-b391-4c757d8f921c
- inputs: ['content']
- outputs: ['content', '_id', 'initiator']
- task count: 7
- adapter/app tasks:
    - [4e93] putReposOwnerRepoContentsPath (app=GitHub, locationType=GitHub) | Create a file.
    - [72e3] getReposOwnerRepoContentsPath (app=GitHub, locationType=GitHub) | Get contents.
This method returns the contents of a file or directory in a repository.
Files and sym
- transition count: 8

## Upgrade POC
- id: 8af2da4b-372c-4662-9876-5599cbf22dcc
- inputs: []
- outputs: ['_id', 'initiator']
- task count: 22
- childJob tasks:
    - [54bf] -> @6994dbcc6d90cb331ac67eb3: run command loop | command loop-precheck
    - [eafc] -> @6994dbcc6d90cb331ac67eb3: Create Blob | Run Child Job
    - [4b21] -> @6994dbcc6d90cb331ac67eb3: Create Tree and commit | Run Child Job
    - [714c] -> @6994dbcc6d90cb331ac67eb3: Create Blob | Post Check Files
    - [ddaf] -> @6994dbcc6d90cb331ac67eb3: Create Tree and commit | Post Check Commit
- adapter/app tasks:
    - [0831] getReposOwnerRepoContentsPath (app=GitHub, locationType=GitHub) | Pre/Post Checks
    - [6a4f] getReposOwnerRepoContentsPath (app=GitHub, locationType=GitHub) | Upgrade commands
    - [36bf] renderJinjaTemplate (app=TemplateBuilder, locationType=None) | Render Jinja Template
    - [e959] mailWithOptions (app=EmailOpensource, locationType=EmailOpensource) | Send email with cc, bcc, and attachment options
- transition count: 24

## Upgrade Wrapper
- id: d896d8e6-8a38-42da-b6cc-caf3308d83c3
- inputs: ['version', 'bootMode', 'devices', 'emails']
- outputs: ['version', 'bootMode', 'devices', 'emails', '_id', 'initiator']
- task count: 7
- childJob tasks:
    - [039d] -> @6994dbcc6d90cb331ac67eb3: Agnostic Upgrade | Run Child Job
- adapter/app tasks:
    - [01ca] mailWithOptions (app=EmailOpensource, locationType=EmailOpensource) | Send email with cc, bcc, and attachment options
    - [dac9] renderJinja2TemplateWithCast (app=TemplateBuilder, locationType=None) | Render Jinja2 Template With Data Cast
- transition count: 6

## getGithubFile
- id: 7f47aac3-2f1b-48d5-968b-c5d8a0ee4814
- inputs: ['fileName', 'pathToFile']
- outputs: ['fileName', 'pathToFile', '_id', 'initiator', 'fileContents']
- task count: 7
- adapter/app tasks:
    - [e830] getReposOwnerRepoContentsPath (app=GitHub, locationType=GitHub) | Show commands
- transition count: 6

## run command loop
- id: cec567b8-eb9e-455a-8d1a-a9c81cebe5e2
- inputs: ['command']
- outputs: ['command', '_id', 'initiator', 'result']
- task count: 4
- adapter/app tasks:
    - [df35] RunCommand (app=MOP, locationType=None) | Run a Command against a Device
- transition count: 4

## testEmail
- id: 3178c5ff-9bd2-48fd-9bb9-1ea613b1c5ed
- inputs: []
- outputs: ['_id', 'initiator']
- task count: 6
- adapter/app tasks:
    - [34e7] mailWithOptions (app=EmailOpensource, locationType=EmailOpensource) | Send email with cc, bcc, and attachment options
    - [10a3] renderJinjaTemplate (app=TemplateBuilder, locationType=None) | Render Jinja Template
- transition count: 8

## test
- id: 3577ef51-fe55-40ca-bd60-8fa58f1f9c37
- inputs: []
- outputs: ['_id', 'initiator']
- task count: 4
- adapter/app tasks:
    - [382b] renderJinja2TemplateWithCast (app=TemplateBuilder, locationType=None) | Render Jinja2 Template With Data Cast
- transition count: 3

## test email
- id: de2af333-1d84-4878-8e87-05c90c4193c6
- inputs: []
- outputs: ['_id', 'initiator']
- task count: 4
- adapter/app tasks:
    - [1595] renderJinja2TemplateWithCast (app=TemplateBuilder, locationType=None) | Render Jinja2 Template With Data Cast
    - [c52f] mailWithOptions (app=EmailOpensource, locationType=EmailOpensource) | Send email with cc, bcc, and attachment options
- transition count: 3

## update ref test
- id: 46057b5d-2387-46f4-95e5-df2e8ded59b2
- inputs: []
- outputs: ['_id', 'initiator']
- task count: 3
- adapter/app tasks:
    - [2d88] genericAdapterRequest (app=GitHub, locationType=GitHub) | Makes the requested generic call
- transition count: 2