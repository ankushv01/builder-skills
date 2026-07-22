
### Agnostic Upgrade
- uuid: `c4a22ed6-59d4-4019-94ae-be64793c7199`
- **Inputs:**
    - `version`: string (required)
    - `devices`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `bootMode`: string (required)
- **Outputs:**
    - `version`: string
    - `devices`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `bootMode`: string
    - `_id`: string
    - `initiator`: string
    - `emailMessage`: string
    - `reattempt`: number
    - `success`: string
    - `postCheckCommit`: object
    - `preCheckCommit`: object

### Upgrade Wrapper
- uuid: `d896d8e6-8a38-42da-b6cc-caf3308d83c3`
- **Inputs:**
    - `version`: string (required)
    - `bootMode`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `devices`: ['array'] (required)
    - `emails`: string (required)
- **Outputs:**
    - `version`: string
    - `bootMode`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `devices`: ['array']
    - `emails`: string
    - `_id`: string
    - `initiator`: string

### Get Netbox Device and Context
- uuid: `90f1de84-6afb-49a7-ae87-e41af86b447d`
- **Inputs:**
    - `version`: string (required)
    - `name`: string (required)
- **Outputs:**
    - `version`: string
    - `name`: string
    - `_id`: string
    - `initiator`: string
    - `success`: string
    - `check_command_sets`: array<?>
    - `diff_ignore_patterns`: array<?>
    - `images`: array<?>
    - `upgrade_sequence`: array<?>
    - `rollback_sequence`: array<?>
    - `image_transfer`: array<?>
    - `validation_rules`: array<?>
    - `plugin`: object

### File Transfer
- uuid: `d88ecc9e-05d8-4e2e-87ea-a14f48a0764d`
- **Inputs:**
    - `image`: object (required)
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `devices`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `transferObject`: object (required)
- **Outputs:**
    - `image`: object
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `devices`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `transferObject`: object
    - `_id`: string
    - `initiator`: string
    - `skipTransferResults`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `transferResults`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `transferSuccess`: boolean

### Run Commands for Checks and Upgrades
- uuid: `3118ab8b-976b-47d0-afc4-d5fdb4b2dd77`
- **Inputs:**
    - `image`: object (required)
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `devices`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `object`: object (required)
- **Outputs:**
    - `image`: object
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `devices`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `object`: object
    - `_id`: string
    - `initiator`: string
    - `commandResults`: ['array', 'boolean', 'null', 'number', 'object', 'string']

### Run Commands for Validations
- uuid: `6be1b700-82f5-40c2-a14a-5e57e2ef91ce`
- **Inputs:**
    - `obj`: array<?> (required)
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `devices`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `image`: object (required)
- **Outputs:**
    - `obj`: array<?>
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `devices`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `image`: object
    - `_id`: string
    - `initiator`: string
    - `commandResults`: ['array', 'boolean', 'null', 'number', 'object', 'string']

### Create and Run Command Template
- uuid: `a33634f1-5a02-4dcf-877e-f31ffded1396`
- **Inputs:**
    - `obj`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `devices`: array<string> (required)
- **Outputs:**
    - `obj`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `devices`: array<string>
    - `_id`: string
    - `initiator`: string
    - `templateName`: ?
    - `templateResults`: object
    - `success`: boolean

### Create and Run Verification Command Template
- uuid: `4e110285-fbba-494d-aeb0-1c53adbede58`
- **Inputs:**
    - `obj`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `devices`: array<string> (required)
- **Outputs:**
    - `obj`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `devices`: array<string>
    - `_id`: string
    - `initiator`: string
    - `templateName`: ?
    - `templateResults`: object
    - `success`: boolean

### DynamicTemplateCreation
- uuid: `74872f50-01cf-4ca0-9b91-281fa22a40fb`
- **Inputs:**
    - `obj`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
- **Outputs:**
    - `obj`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `_id`: string
    - `initiator`: string
    - `templateName`: ['array', 'string', 'boolean', 'number', 'object']
    - `success`: boolean

### DynamicTemplateCreation with Validations
- uuid: `73aaefed-d37b-4016-a209-5a1579ef713f`
- **Inputs:**
    - `obj`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
- **Outputs:**
    - `obj`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `version`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `step`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `_id`: string
    - `initiator`: string
    - `templateName`: ['array', 'string', 'boolean', 'number', 'object']
    - `success`: boolean

### Clean Up Template
- uuid: `121c23c8-30fd-4008-bad8-b89d9482b036`
- **Inputs:**
    - `id`: string (required)
- **Outputs:**
    - `id`: string
    - `_id`: string
    - `initiator`: string
    - `success`: boolean

### Take Device Backup
- uuid: `f49e1ea2-d37d-4f6a-b712-f993498ac676`
- **Inputs:**
    - `name`: string (required)
- **Outputs:**
    - `name`: string
    - `_id`: string
    - `initiator`: string

### Fetch CRQ Details
- uuid: `798283e5-cdf0-4b78-b0c1-089c40df3e19`
- **Inputs:**
    - `Infrastructure_Change_Id`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
- **Outputs:**
    - `Infrastructure_Change_Id`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `_id`: string
    - `initiator`: string
    - `devices`: ?
    - `success`: boolean

### Software Upgrade
- uuid: `695e06de-982e-456c-9c1e-1c4d2b627303`
- **Inputs:**
    - `formData`: object (required)
- **Outputs:**
    - `formData`: object
    - `_id`: string
    - `initiator`: string
    - `pathToFile`: ?

### Software Upgrade TAD
- uuid: `d8cc7b53-f8e7-4ec3-8d1f-10c1e4b956fc`
- **Inputs:**
    - `version`: string (required)
- **Outputs:**
    - `version`: string
    - `_id`: string
    - `initiator`: string
    - `upgradeSteps`: ?
    - `manifest`: ?

### Execute Upgrade Steps
- uuid: `45611fcc-6d45-43e9-b520-0144abc667d2`
- **Inputs:**
    - `pathTofile`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `fileName`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
- **Outputs:**
    - `pathTofile`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `fileName`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `_id`: string
    - `initiator`: string

### Execute Upgrade Steps TAD
- uuid: `96ae2cce-a1c7-4286-ac26-c2e8479f9ca8`
- **Inputs:**
    - `pathTofile`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `fileName`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
- **Outputs:**
    - `pathTofile`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `fileName`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `_id`: string
    - `initiator`: string
    - `fileContents`: ?

### Run Steps with File Contents
- uuid: `a1809855-bf71-4744-86a5-d7a78dac31e2`
- **Inputs:**
    - (none)
- **Outputs:**
    - `_id`: string
    - `initiator`: string

### Retrieve from GitHub
- uuid: `258d0f07-f6f4-4d7b-a0d1-56a3afd8ae27`
- **Inputs:**
    - (none)
- **Outputs:**
    - `_id`: string
    - `initiator`: string

### getGithubFile
- uuid: `7f47aac3-2f1b-48d5-968b-c5d8a0ee4814`
- **Inputs:**
    - `fileName`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
    - `pathToFile`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
- **Outputs:**
    - `fileName`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `pathToFile`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `_id`: string
    - `initiator`: string
    - `fileContents`: ['array', 'string', 'boolean', 'number', 'object']

### run command loop
- uuid: `cec567b8-eb9e-455a-8d1a-a9c81cebe5e2`
- **Inputs:**
    - `command`: string (required)
- **Outputs:**
    - `command`: string
    - `_id`: string
    - `initiator`: string
    - `result`: object

### Run Transfer Scripts
- uuid: `c7be8778-bb22-44a4-9c4e-f671553f3b26`
- **Inputs:**
    - (none)
- **Outputs:**
    - `_id`: string
    - `initiator`: string

### Upgrade POC
- uuid: `8af2da4b-372c-4662-9876-5599cbf22dcc`
- **Inputs:**
    - (none)
- **Outputs:**
    - `_id`: string
    - `initiator`: string

### Create Blob
- uuid: `a4ad079d-8d1d-471a-855d-f8084a194431`
- **Inputs:**
    - `command`: string (required)
    - `content`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
- **Outputs:**
    - `command`: string
    - `content`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `_id`: string
    - `initiator`: string
    - `filePath`: string
    - `sha`: ?

### Create Tree and commit
- uuid: `9cc97957-d816-425d-a45a-85cfa1c520be`
- **Inputs:**
    - `blobArray`: array<?> (required)
- **Outputs:**
    - `blobArray`: array<?>
    - `_id`: string
    - `initiator`: string
    - `commitResult`: object

### Update Version TXT
- uuid: `9e310a06-fb92-4644-b391-4c757d8f921c`
- **Inputs:**
    - `content`: ['array', 'boolean', 'null', 'number', 'object', 'string'] (required)
- **Outputs:**
    - `content`: ['array', 'boolean', 'null', 'number', 'object', 'string']
    - `_id`: string
    - `initiator`: string