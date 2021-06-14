# Bereitstellung von Umgebungen pro Feature-Branch im Kubernetes-Kluster

## Test-Anwendung
Die Anwendung läuft im Kubernetes-Kluster. Um den Code auf das wesentliche zu konzentrieren, habe ich eine leicht zu startende Anwendung gesucht, bei der möglichst wenig zu konfigurieren ist. Der Swagger-Editor ist hierfür perfekt. Zum Ausführen in Kubernetes benötigt man lediglich 3 kleine Konfigurationsdateien (deployment.yaml, service.yaml, ingress.yaml).

## Ziele
Eine Build-Pipeline existiert häufig erst ab dem Zeitpunkt, an dem man auf einen zentralen Branch merged wie z.B. der Develop-Branch. Arbeitet man noch am Feature-Branch, gibt es meist keine Möglichkeit unabhängig seine Änderungen in einer Server-Umgebung zu testen.

Schön wäre es, wenn man die zu erstellende Anwendung unter einer eigenen Feature-Branch-Url erreichen und testen könnte.

Wie es mit Kubernetes und der Azure-Pipeline funktionieren kann, wird hier gezeigt.

## Planung
1. Mit Erstellung eines Branches mit dem Namensmuster "feature/`feature-name`" wird in Kubernetes ein neuer Namespace mit dem Namensmuster "dev-`feature-name`" angelegt. Hierzu gibt es im Projekt noch eine Datei "namespace.yaml".

2. In den 4 Yaml-Files muss beim Build der Namespace an den neuen Feature-Branch angepasst werden.
Außerdem muss in der ingress.yaml die Eingangs-Url so angepasst werden, dass man die Anwendung unter http://domain/`feature-name`/... erreicht.

3. Schlussendlich muss noch die Bereinigung des Kubernetes-Klusters geplant werden. Bei gelöschten Feature-Branches, müssen die erstellten Namespaces inkl. aller Ressourcen wieder bereinigt werden. Da mir keine Möglichkeit bekannt ist, auf ein "Branch-Delete-Event" eine Pipeline zu starten, musste dies anders gelöst werden.

## Pipelines
1. Zu einem bestimmten Zeitpunkt werden alle erstellten Feature-Umgebungen im Kubernetes-Kluster gelöscht. Dies kann z.B. nachts geschehen.
```
# Reagiert auf keinen Checkin.
trigger: none

# Wird nur auf dem Master ausgeführt, damit sichergestellt ist, dass dieser
# Job nur einmal ausgeführt wird.
# Mit der einmaligen Ausführung werden alle Kubernetes-Namespaces
# mit dem Namensmuster "dev-*" gelöscht.
# 10 Minuten später werden alle noch existierenden Feature-Branches neu gebaut.
schedules:
  - cron: "50 21 * * *" # entspricht 23:50 unserer Zeit
    displayName: Daily midnight build
    branches:
      include: 
      - master 
    always: true

pool:
  vmImage: ubuntu-latest

steps:
# Delete All namespaces start with $(Environment-Prefix)
- task: AzureCLI@2
  inputs:
    azureSubscription: '$(Subscription)'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az aks get-credentials --resource-group $(RessourceGroup) --name $(K8sCluster)
      kubectl get ns
      kubectl get namespaces --no-headers=true -o custom-columns=:metadata.name | grep $(Environment-Prefix) | xargs kubectl delete namespace
      kubectl get ns
  displayName: 'Delete All namespaces start with $(EnvironmentPrefix)'
```

2. Bei jedem Checkin in einen Feature-Branch, muss die Feature-Branch-Umgebung neu erstellt werden. Außerdem müssen nach der nächtlichen Bereinigung (s. Ziff. 1) alle Feature-Branch-Umgebungen neu erstellt werden, zu denen noch ein Feature-Branch existiert. Dies kann nachts z.B. 10 Minuten nach dem Löschen aller Umgebungen erfolgten.
````
# Jeder Branch erhält ein eigenes Deployment und ist unter einen eigenen 
# Url erreichbar. Aktuell:
# http://20.79.116.165/dev-<Branch-Name>/swagger/
# CI-Trigger wird bei jedem Eincheckvorgang im Branch mit dem Namensmuster
# "feature/*" gestartet.
trigger:
  branches:
    include:
    - feature/*
# Jede Nacht um 0 Uhr werden alle Feature-Branches neu gebaut, die zu diesem
# Zeitpunkt existieren. Kurz vorher werden alle Feature-Branches gelöscht.
# Dies erfolgt in einem separatem Skript.
# So werden Deployments entfernt, bei denen der dazugehörige Branch im Laufe
# des Tages gelöscht wurde.
schedules:
  - cron: "00 22 * * *" # entspricht 0 Uhr unserer Zeit.
    displayName: Daily midnight build
    branches:
      include:
      - feature/*
    always: true

variables:
- group: Globals

pool:
  vmImage: ubuntu-latest
steps:
# Setzen des Namespaces nach dem Namensmuster "dev-<Branch-Name>" 
# in den jeweiligen yaml-files.
- task: PowerShell@2
  displayName: Ersetze __branch__ string in ingress.yaml
  inputs:
    targetType: 'inline'
    script: '(Get-Content -path $(Build.SourcesDirectory)/k8s/ingress.yaml -Raw) -replace "__branch__","$(EnvironmentPrefix)$(Build.SourceBranchName)"|Set-Content -Path $(Build.SourcesDirectory)/k8s/ingress.yaml'
- task: PowerShell@2
  displayName: Ersetze __branch__ string in service.yaml
  inputs:
    targetType: 'inline'
    script: '(Get-Content -path $(Build.SourcesDirectory)/k8s/service.yaml -Raw) -replace "__branch__","$(EnvironmentPrefix)$(Build.SourceBranchName)"|Set-Content -Path $(Build.SourcesDirectory)/k8s/service.yaml'
- task: PowerShell@2
  displayName: Ersetze __branch__ string in deployment.yaml
  inputs:
    targetType: 'inline'
    script: '(Get-Content -path $(Build.SourcesDirectory)/k8s/deployment.yaml -Raw) -replace "__branch__","$(EnvironmentPrefix)$(Build.SourceBranchName)"|Set-Content -Path $(Build.SourcesDirectory)/k8s/deployment.yaml'
- task: PowerShell@2
  displayName: Ersetze __branch__ string in namespace.yaml
  inputs:
    targetType: 'inline'
    script: '(Get-Content -path $(Build.SourcesDirectory)/k8s/namespace.yaml -Raw) -replace "__branch__","$(EnvironmentPrefix)$(Build.SourceBranchName)"|Set-Content -Path $(Build.SourcesDirectory)/k8s/namespace.yaml'

- task: Kubernetes@1
  displayName: Deploye namespace.yaml
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscriptionEndpoint: '$(Subscription)'
    azureResourceGroup: '$(RessourceGroup)'
    kubernetesCluster: '$(K8sCluster)'
    command: 'apply'
    useConfigurationFile: true
    configuration: '$(Build.SourcesDirectory)/k8s/namespace.yaml'
    secretType: 'dockerRegistry'
    containerRegistryType: 'Azure Container Registry'
- task: Kubernetes@1
  displayName: Deploye deployment.yaml
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscriptionEndpoint: '$(Subscription)'
    azureResourceGroup: '$(RessourceGroup)'
    kubernetesCluster: '$(K8sCluster)'
    command: 'apply'
    useConfigurationFile: true
    configuration: '$(Build.SourcesDirectory)/k8s/deployment.yaml'
    secretType: 'dockerRegistry'
    containerRegistryType: 'Azure Container Registry'

- task: Kubernetes@1
  displayName: Deploye service.yaml
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscriptionEndpoint: '$(Subscription)'
    azureResourceGroup: '$(RessourceGroup)'
    kubernetesCluster: '$(K8sCluster)'
    command: 'apply'
    useConfigurationFile: true
    configuration: '$(Build.SourcesDirectory)/k8s/service.yaml'
    secretType: 'dockerRegistry'
    containerRegistryType: 'Azure Container Registry'
    arguments: 
- task: Kubernetes@1
  displayName: Deploye ingress.yaml
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscriptionEndpoint: '$(Subscription)'
    azureResourceGroup: '$(RessourceGroup)'
    kubernetesCluster: '$(K8sCluster)'
    command: 'apply'
    useConfigurationFile: true
    configuration: '$(Build.SourcesDirectory)/k8s/ingress.yaml'
    secretType: 'dockerRegistry'
    containerRegistryType: 'Azure Container Registry'
````

## Ergebnis
Durch die dynamische Erstellung von Test-Umgebungen pro Feature-Branch kommen die Vorteile von Kubernetes richtig zum Tragen. Mit der nächtlichen Bereinigung wird sichergestellt, dass keine unnötigen Ressourcen verbraucht werden.