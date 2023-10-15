# Cloudformation으로 Appmesh Resource 생성
```
---
Parameters:
  AppMeshMeshName:
    Type: String
    Description: Name of mesh
    Default: skills-mesh
  CloudMapNamespace:
    Type: String
    Default: skills.local
  CloudMapProductService:
    Type: String
    Default: product
  CloudMapStressService:
    Type: String
    Default: stress

Resources:

#############################################
# AppMesh virtual services & router
#############################################
  ProductVirtualRouter:
    Type: AWS::AppMesh::VirtualRouter
    Properties:
        MeshName: !Ref AppMeshMeshName
        VirtualRouterName: product-vr
        Spec:
          Listeners:
            - PortMapping:
                Port: 8080
                Protocol: http

  ProductRoute:
    Type: AWS::AppMesh::Route
    DependsOn:
      - ProductVirtualRouter
      - ProductAppVirtualNode
    Properties:
      MeshName: !Ref AppMeshMeshName
      VirtualRouterName: product-vr
      RouteName: product-route
      Spec:
        HttpRoute:
          Action:
            WeightedTargets:
              - VirtualNode: product-service-vn
                Weight: 1 # enable traffic distribution = 100% will be directed to this virtual node
          Match:
            Prefix: "/v1/product" #all traffic will be forwared to our virtual node (Product-service)

  /* StressVirtualRouter:
    Type: AWS::AppMesh::VirtualRouter
    Properties:
        MeshName: !Ref AppMeshMeshName
        VirtualRouterName: stress-vr
        Spec:
          Listeners:
            - PortMapping:
                Port: 8080
                Protocol: http

  StressRoute:
    Type: AWS::AppMesh::Route
    DependsOn:
      - StressVirtualRouter
      - StressAppVirtualNode
    Properties:
      MeshName: !Ref AppMeshMeshName
      VirtualRouterName: stress-vr
      RouteName: stress-route
      Spec:
        HttpRoute:
          Action:
            WeightedTargets:
              - VirtualNode: stress-service-vn
                Weight: 1 # enable traffic distribution = 100% will be directed to this virtual node
          Match:
            Prefix: "/v1/stress" #all traffic will be forwared to our virtual node (Product-service) */ route로 할 경우

  ProductVirtualService:
    Type: AWS::AppMesh::VirtualService
    DependsOn:
      - ProductVirtualRouter
    Properties:
      MeshName: !Ref AppMeshMeshName
      VirtualServiceName: !Sub "${CloudMapProductService}.${CloudMapNamespace}"
      Spec:
        Provider:
          VirtualRouter:
            VirtualRouterName: product-vr

  StressVirtualService:
    Type: AWS::AppMesh::VirtualService
    DependsOn:
      - StressAppVirtualNode
    Properties:
      MeshName: !Ref AppMeshMeshName
      VirtualServiceName: !Sub "${CloudMapStressService}.${CloudMapNamespace}"
      Spec:
        Provider:
          VirtualNode: # VirtualRouter route 경우
            VirtualNodeName: stress-service-vn # VirtualRouterName: stress-vr route 경우

#############################################
# virtual nodes for "physical" ECS services
#############################################
  ProductAppVirtualNode:
    Type: AWS::AppMesh::VirtualNode
    Properties:
      MeshName: !Ref AppMeshMeshName
      VirtualNodeName: product-service-vn
      Spec:
        Listeners:
          - PortMapping:
              Port: 8080
              Protocol: http
            HealthCheck:
              Protocol: http
              Path: "/health"
              HealthyThreshold: 2
              UnhealthyThreshold: 2
              TimeoutMillis: 2000
              IntervalMillis: 5000
        ServiceDiscovery:
          AWSCloudMap:
            NamespaceName: !Ref CloudMapNamespace
            ServiceName: !Ref CloudMapProductService
        
  StressAppVirtualNode:
    Type: AWS::AppMesh::VirtualNode
    Properties:
      MeshName: !Ref AppMeshMeshName
      VirtualNodeName: stress-service-vn
      Spec:
        Listeners:
          - PortMapping:
              Port: 8080
              Protocol: http
            HealthCheck:
              Protocol: http
              Path: "/health"
              HealthyThreshold: 2
              UnhealthyThreshold: 2
              TimeoutMillis: 2000
              IntervalMillis: 5000
        ServiceDiscovery:
          AWSCloudMap:
            NamespaceName: !Ref CloudMapNamespace
            ServiceName: !Ref CloudMapStressService

Outputs:    
  ProductAppVirtualNodeName:
    Description: Name of the ProductApp VirtualNode
    Value:
      Fn::GetAtt:
      - ProductAppVirtualNode
      - VirtualNodeName
  StressAppVirtualNodeName:
    Description: Name of the StressApp VirtualNode
    Value:
      Fn::GetAtt:
      - StressAppVirtualNode
      - VirtualNodeName     
```
