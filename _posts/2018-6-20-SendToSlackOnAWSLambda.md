---
layout: post
title: AWS 람다에서 powershell 실행하고 Slack 에 메시지 보내기
published: false
date: 2018-6-20
categories: [script]
tags: [aws,lambda,slack,ssm]
---

####

1. 실행하려고 하는 장비에 AWS_SSM 설치
2. SLACK Outgoing Hook 설정

```javascript
const aws = require('aws-sdk');
const https = require('https');
const url = require('url');

const ssm = new aws.SSM({ apiVersion: '2014-11-06' })

const slackURL = 'SLACK HOOK URL'
const slackReqOpts = url.parse(slackURL);
slackReqOpts.method = 'POST';
slackReqOpts.headers = { 'Content-Type': 'application/json' };

const SSM_Params = {
    DocumentName: 'AWS-RunPowerShellScript',
    /* required */
    Comment: 'Run PowerShell',
    InstanceIds: [
        'target_id',
        /* more items */
    ],
    Parameters: {
        'commands': [
            'echo hello',
        ],
    },
    TimeoutSeconds: 60
};

const sendSlack = function(message) {
    return new Promise(function(resolve, reject) {
        let req = https.request(slackReqOpts, function(res) {
            if (res.statusCode === 200) {
                resolve(message);
            } else {
                reject(res.statusCode);
            }
        });
        req.write('{"text":"' + message + '"}');
        req.end();
    });
}

const sendSSMcommand = function(param) {
    return new Promise(function(resolve, reject) {
        ssm.sendCommand(SSM_Params, function(err, data) {
            if (err) {
                reject(err);
            } else {
                resolve('complete');
            }
        });
    });
}

exports.handler = (event, context, callback) => {

    sendSlack("run power shell. start")
        .then(sendSSMcommand)
        .then(function(data) {
            sendSlack("run power shell. complete");
            callback(null, { "errorMessage" : "complete" } );
        }).catch(function(err) {
            sendSlack("run power shell. failed");
            callback(err, { "errorMessage" :  err  } );
        });
};
```
