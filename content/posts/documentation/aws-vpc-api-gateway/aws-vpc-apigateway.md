---
title: "AWS API Gateway 之 Resource Policy探討"
date: 2022-03-15T12:00:00+08:00
tags: ["aws", "api-gateway", "vpc", "resource policy"]
categories: ["documentation"]
toc: 
    auto: true
---

## 前言

AWS API Gateway是AWS Serverless的服務之一，用於建立REST API服務用，而有些狀況下，REST API我們想要只開放給特定服務，或是自己內部的AWS Service使用

本篇文章會簡單說明，如何透過Resource Policy來控制AWS API Gateway的Access

## API Gateway Type

API Gateway根據開放特性，還有延遲性等有不同的Type可以選擇

### Regional and Edge Optimized

這2種類型都是屬於Public Type，Internet都可以Access到，2者細部的差異在於Edge Optimized可以自行產生Cookie，來跟CDN做搭配，Regional則屬於放置在特地區域的API Gateway，當然還有更多細節差異，不過與本篇要探討的無關

### Private 

如字面意思，這種Type需要搭配Resource Policy來過濾你的Request來源，也是本篇的重點，基本上可以有3種方式來進行Filter，AWS IAM, AWS VPC, AWS VPC Endpoint

## Resource Policy

除了可以針對整個API Gateway Resource去做限制之外，也可以細節到後續的Path限制，算是很大的細分
如果被Deny成功，後續也不會再近到AWS Lambda這層，可以節省費用

Deny的Response通常會長這樣
![picture 1](../2.png)

### AWS Account

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::{{otherAWSAccountID}}:root",
                    "arn:aws:iam::{{otherAWSAccountID}}:user/{{otherAWSUserName}}",
                    "arn:aws:iam::{{otherAWSAccountID}}:role/{{otherAWSRoleName}}"
                ]
            },
            "Action": "execute-api:Invoke",
            "Resource": [
                "execute-api:/{{stageNameOrWildcard*}}/{{httpVerbOrWildcard*}}/{{resourcePathOrWildcard*}}"
            ]
        }
    ]
}
```

### IP Range

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "execute-api:/{{stageNameOrWildcard}}/{{httpVerbOrWildcard}}/{{resourcePathOrWildcard}}",
            "Condition" : {
                "IpAddress": {
                    "aws:SourceIp": [ "10.0.0.0/16", "200.200.200.200/32" ]
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "execute-api:/{{stageNameOrWildcard}}/{{httpVerbOrWildcard}}/{{resourcePathOrWildcard}}"
        }
    ]
}
```

### VPC

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "execute-api:/{{stageNameOrWildcard}}/{{httpVerbOrWildcard}}/{{resourcePathOrWildcard}}",
            "Condition": {
                "StringNotEquals": {
                    "aws:sourceVpc": "{{vpcID}}"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "execute-api:/{{stageNameOrWildcard}}/{{httpVerbOrWildcard}}/{{resourcePathOrWildcard}}"
        }
    ]
}
```

### VPC Endpoint

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "execute-api:/{{stageNameOrWildcard}}/{{httpVerbOrWildcard}}/{{resourcePathOrWildcard}}",
            "Condition": {
                "StringNotEquals": {
                    "aws:sourceVpce": "{{vpc endpoint ID}}"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "execute-api:/{{stageNameOrWildcard}}/{{httpVerbOrWildcard}}/{{resourcePathOrWildcard}}"
        }
    ]
}
```

## Private Type 的限制與條件

### VPC Endpoint

設定成Private Type有個先決條件，你也必須建立一個VPC Endpoint，然後去做綁定，不然Default Endpoint任何Service都會Access不到

![picture 1](../1.png)

Endpoint format會變成
`https://{{API GATEWAY ID}}-vpce-{{VPC Endpoint ID}}.execute-api.{{Region}}.amazonaws.com/{{Stage Name}}`

### Custom Domain

API Gateway可以設定Domain功能，不過這功能只有在你設定成Edge Optimized 或 Regional才有用

### Resource Policy

一定要設定Resource Policy，不然你也沒有辦法Deploy

## Public Type的限制

### Policy on VPC and VPC Endpoint

你設定成Public，基本上這2個Type就完全沒有作用，可以參考下面AWS提供的文件

![image](../3.png)

Ref: https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-resource-policies-aws-condition-keys.html


## 總結表格

這邊以常用的狀況，將一些限制跟搭配做成Table供參考

| APIGateway Type            | Need VPC Endpoint | Resource Policy Way | Available on Custom Domain | Domain format                                                                                           |
|----------------------------|-------------------|---------------------|----------------------------|---------------------------------------------------------------------------------------------------------|
| Edge Optimized or Regional | No                | AWS Account, AWS:SourceIP        | Yes                        | https://{{CUSTOM DOMAIN}}                                                                               |
| Private                    | Yes               | All | No                         | https://{{API GATEWAY ID}}-vpce-{{VPC Endpoint ID}}.execute-api.{{Region}}.amazonaws.com/{{Stage Name}} |