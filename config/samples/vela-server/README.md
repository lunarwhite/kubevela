# Definition Docs

## Reserved word
### patch
Perform the CUE AND operation with the content declared by 'patch' and workload cr

### output
Generate a new cr, which is generally associated with workload cr

## Workload Definition
The following workload definition is to generate a deployment
```
apiVersion: core.oam.dev/v1alpha2
kind: WorkloadDefinition
metadata:
  name: worker
  annotations:
    definition.oam.dev/description: "Long-running scalable backend worker without network endpoint"
spec:
  definitionRef:
    name: deployments.apps
  extension:
    template: |
      output: {
      	apiVersion: "apps/v1"
      	kind:       "Deployment"
      	spec: {
      		selector: matchLabels: {
      			"app.oam.dev/component": context.name
      		}

      		template: {
      			metadata: labels: {
      				"app.oam.dev/component": context.name
      			}

      			spec: {
      				containers: [{
      					name:  context.name
      					image: parameter.image

      					if parameter["cmd"] != _|_ {
      						command: parameter.cmd
      					}
      				}]
      			}
      		}
      	}
      }

      parameter: {
      	// +usage=Which image would you like to use for your service
      	// +short=i
      	image: string

      	cmd?: [...string]
      }
```

If defined an application as follows
```
apiVersion: core.oam.dev/v1alpha2
kind: Application
metadata:
  name: application-sample
spec:
  services:
    myweb:
      type: worker
      image: "busybox"
      cmd:
      - sleep
      - "1000"
```
we will get a deployment
```
apiVersion: apps/v1
kind: Deployment
spec:
  selector:
    matchLabels:
      app.oam.dev/component: myweb
  template:
    metadata:
      labels:
        app.oam.dev/component: myweb
    spec:
      containers:
      - command:
        - sleep
        - "1000"
        image: busybox
        name: myweb
```
##  Service Trait Definition

Define a trait Definition that appends service to workload(worker) , as shown below
```
apiVersion: core.oam.dev/v1alpha2
kind: TraitDefinition
metadata:
  annotations:
    definition.oam.dev/description: "service the app"
  name: service
spec:
  appliesToWorkloads:
    - webservice
    - worker
  definitionRef:
    name: service.v1
  extension:
    template: |-
      patch: {spec: template: metadata: labels: app: context.name}
      output: {
        apiVersion: "v1"
        kind: "Service"
        metadata: name: context.name
        spec: {
          selector:  app: context.name
          ports: [
            for k, v in parameter.http {
              port: v
              targetPort: v
            }
          ]
        }
      }
      parameter: {
        http: [string]: int
      }
```

If add service capability to the application, as follows
```
apiVersion: core.oam.dev/v1alpha2
kind: Application
metadata:
  name: application-sample
spec:
  services:
    myweb:
      type: worker
      image: "busybox"
      cmd:
      - sleep
      - "1000"
      service:
        http:
          server: 80
```

we will get a new deployment and service
```
// origin deployment  template add labels
apiVersion: apps/v1
kind: Deployment
spec:
  selector:
    matchLabels:
      app.oam.dev/component: myweb
  template:
    metadata:
      labels:
        // add label app
        app: myweb
        app.oam.dev/component: myweb
    spec:
      containers:
      - command:
        - sleep
        - "1000"
        image: busybox
        name: myweb
---
apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: myweb         
```

## Scaler Trait Definition

Define a trait Definition that scale workload(worker) replicas
```
apiVersion: core.oam.dev/v1alpha2
kind: TraitDefinition
metadata:
  annotations:
    definition.oam.dev/description: "Manually scale the app"
  name: scaler
spec:
  appliesToWorkloads:
    - webservice
    - worke
  extension:
    template: |-
      patch: {
         spec: replicas: parameter.replicas
      }
      parameter: {
      	//+short=r
      	replicas: *1 | int
      }
```
If add scaler capability to the application, as follows
```
apiVersion: core.oam.dev/v1alpha2
kind: Application
metadata:
  name: application-sample
spec:
  services:
    myweb:
      type: worker
      image: "busybox"
      cmd:
      - sleep
      - "1000"
      service:
        http:
          server: 80
      scaler:
        replicas: 10     
```

The deployment replicas will be scale to 10
```
apiVersion: apps/v1
kind: Deployment
spec:
  selector:
    matchLabels:
      app.oam.dev/component: myweb
  // scale to 10    
  replicas: 10    
  template:
    metadata:
      labels:
        // add label app
        app: myweb
        app.oam.dev/component: myweb
    spec:
      containers:
      - command:
        - sleep
        - "1000"
        image: busybox
        name: myweb
```

## Sidecar Trait Definition

Define a trait Definition that append containers to workload(worker)
```
apiVersion: core.oam.dev/v1alpha2
kind: TraitDefinition
metadata:
  annotations:
    definition.oam.dev/description: "add sidecar to the app"
  name: sidecar
spec:
  appliesToWorkloads:
    - webservice
    - worke
  extension:
    template: |-
      _containers: context.input.spec.template.spec.containers+[parameter]
      patch: {
         spec: template: spec: containers: _containers
      }
      parameter: {
         name: string
         image: string
         command?: [...string]
      }
```

If add sidercar capability to the application, as follows
```
apiVersion: core.oam.dev/v1alpha2
kind: Application
metadata:
  name: application-sample
spec:
  services:
    myweb:
      type: worker
      image: "busybox"
      cmd:
      - sleep
      - "1000"
      service:
        http:
          server: 80
      scaler:
        replicas: 10    
      sidercar:
        name: "sidecar-test"
        image: "nginx"   
```
The deployment updated as follows
```
apiVersion: apps/v1
kind: Deployment
spec:
  selector:
    matchLabels:
      app.oam.dev/component: myweb
  // scale to 10    
  replicas: 10    
  template:
    metadata:
      labels:
        // add label app
        app: myweb
        app.oam.dev/component: myweb
    spec:
      containers:
      - command:
        - sleep
        - "1000"
        image: busybox
        name: myweb
      - name: sidecar-test
        image: nginx  
```