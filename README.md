# ssis

## Pre-requisites
Install [JQ for windows](https://stedolan.github.io/jq/download/) in windows runner
Or run ```chocolatey install jq ```

## Build

Recommended
```YAML
- name: ssis build
  uses : snsinahub-org/ssis@main
  with:
    project_path: 'C:\Users\Siavash Namvar\a'
    output_path: 'C:\Users\Siavash Namvar\a_temp'
    project_configuration: "Development"
    ssis_action: 'build-sa'
```

```YAML
- name: ssis build
  uses : snsinahub-org/ssis@main
  with:
    project_path: 'C:\Users\Siavash Namvar\a'
    output_path: 'C:\Users\Siavash Namvar\a_temp'
    project_configuration: "Development"
    ssis_action: 'build'
    file_extenstion: '*.sln'
```