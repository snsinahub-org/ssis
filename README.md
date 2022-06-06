# ssis

## Pre-requisites
Install [JQ for windows](https://stedolan.github.io/jq/download/) in windows runner
Or run ```chocolatey install jq ```

## Build

### Standalone
This is lighter version of SSIS build, no Visual Studio installation required on actuib runner, it useses **SSIS DevOps tool**. This action installs the tool on your runner. If needed you can download the file from [here](https://www.microsoft.com/en-us/download/details.aspx?id=102263) and follow the instruction and add it to your Windows envrionment path.

```YAML
- name: ssis build
  uses : snsinahub-org/ssis@main
  with:
    ssis_action: 'build-sa'
    project_path: 'C:\Users\Siavash Namvar\a'
    output_path: 'C:\Users\Siavash Namvar\a_temp'
    project_configuration: "Development"
```
### Required Visual Studio 2017 installed on your runner
```YAML
- name: ssis build
  uses : snsinahub-org/ssis@main
  with:
    ssis_action: 'build'
    project_path: 'C:\Users\Siavash Namvar\a'
    output_path: 'C:\Users\Siavash Namvar\a_temp'
    project_configuration: "Development"
```

## Deploy

### Deploy single ispac to SQL server
```YAML
- name: ssis build
  uses : snsinahub-org/ssis@main
  with:
    ssis_action: 'ssis-deploy-win'
    ssis_source: 'C:\ssis\etl.ispac'
    ssis_destination: '/SSISDB/etl'
    ssis_server: 'myserver'
```

### Deploy multiple ispacs to SQL server
```YAML
- name: ssis build
  uses : snsinahub-org/ssis@main
  with:
    ssis_action: 'ssis-deploy-multiple-win'
    ssis_source: 'C:\ssis'
    ssis_destination: '/SSISDB/etl'
    ssis_server: 'myserver'
```
